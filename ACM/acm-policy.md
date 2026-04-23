# stolostron Policy 框架实现分析

## 概述

stolostron (Open Cluster Management) 的 Policy 框架是一个用于在多集群环境中实现治理、合规性检查和自动修复的完整系统。该框架采用 Hub-Spoke 架构，支持将策略从 Hub 集群分发到多个 Managed 集群，并收集合规性结果。

## 系统架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                       Hub Cluster                            │
│  ┌─────────────────┐         ┌──────────────────────────┐  │
│  │  User Namespace │         │  Cluster Namespace       │  │
│  │  (policy-test)  │────────▶│  (managed1, managed2)    │  │
│  │                 │         │                          │  │
│  │  Root Policy    │         │  Replicated Policies     │  │
│  │  Placement      │         │                          │  │
│  │  PlacementBinding│        │                          │  │
│  └─────────────────┘         └──────────────────────────┘  │
│           │                              │                   │
│           │                              │                   │
│  ┌────────▼────────────┐       ┌────────▼─────────────┐    │
│  │ Policy Propagator   │       │  Framework Addon     │    │
│  │  - Root Policy      │       │  - Spec Sync         │    │
│  │    Reconciler       │       │  - Status Sync       │    │
│  │  - Replicated       │       │  - Secret Sync       │    │
│  │    Policy Reconciler│       │                      │    │
│  └─────────────────────┘       └──────────────────────┘    │
└─────────────────────────────────────┬───────────────────────┘
                                      │
                                      │ Policy Distribution
                                      │ Status Collection
                                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    Managed Cluster                           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Cluster Namespace (managed1)                        │  │
│  │  - Replicated Policies                               │  │
│  └──────────────────────────────────────────────────────┘  │
│           │                                                  │
│  ┌────────▼────────────┐       ┌─────────────────────┐    │
│  │ Framework Addon     │       │  Policy Controllers  │    │
│  │  - Template Sync    │────▶  │  - Config Policy     │    │
│  │  - Status Sync      │       │  - Cert Policy       │    │
│  │  - Spec Sync        │       │  - IAM Policy        │    │
│  │                     │       │  - Operator Policy   │    │
│  └─────────────────────┘       └─────────────────────┘    │
│                                          │                   │
│                                          ▼                   │
│                                 Kubernetes Resources         │
└─────────────────────────────────────────────────────────────┘
```

## 核心组件

### 1. Governance Policy Propagator (Hub 集群)

**位置**: `governance-policy-propagator`

**职责**: 在 Hub 集群上运行，负责策略的分发和状态聚合

**主要控制器**:

#### RootPolicyReconciler
- **文件**: `controllers/propagator/rootpolicy_controller.go`
- **功能**:
  - 监视用户命名空间中的 Policy 资源
  - 根据 PlacementBinding 和 Placement 决定目标集群
  - 将策略复制到集群命名空间（每个 managed cluster 一个）
  - 更新 root policy 的 status.placement 字段
  - 使用 mutex 锁防止并发修改同一策略
  
#### ReplicatedPolicyReconciler
- **文件**: `controllers/propagator/replicatedpolicy_controller.go`
- **功能**:
  - 处理集群命名空间中的复制策略
  - 解析 hub 模板（`{{hub ... hub}}` 语法）
  - 支持策略模板加密/解密
  - 管理策略覆盖（overrides）

#### RootPolicyStatusReconciler
- **文件**: `controllers/rootpolicystatus/`
- **功能**:
  - 从 managed clusters 收集合规性状态
  - 聚合状态到 root policy 的 status.status 字段
  - 计算整体合规性状态

#### PolicySetReconciler
- **文件**: `controllers/policyset/`
- **功能**:
  - 管理 PolicySet 资源，将多个策略组合在一起
  - 支持策略依赖关系和执行顺序

#### PolicyAutomationReconciler
- **文件**: `controllers/automation/`
- **功能**:
  - 基于策略违规触发 Ansible 作业
  - 实现自动化修复

### 2. Governance Policy Framework Addon (Managed 集群)

**位置**: `governance-policy-framework-addon`

**职责**: 在 Managed 集群上运行，同步策略并报告状态

**特点**: 使用双 Manager 模式

#### Spec Sync Controller
- **管理器**: Hub Manager
- **功能**:
  - 监视 Hub 集群上集群命名空间中的策略
  - 在 Managed 集群上创建/更新/删除复制的策略
  - 确保 managed cluster 上的策略规范与 hub 匹配

#### Status Sync Controller
- **管理器**: Managed Manager
- **功能**:
  - 监视 Managed 集群上的策略和事件
  - 使用动态监视器（kubernetes-dependency-watches）跟踪策略模板对象
  - 收集合规性历史（保留最近 10 个事件）
  - 更新 Hub 和 Managed 集群上的策略状态
  - 实现事件去重

#### Template Sync Controller
- **管理器**: Managed Manager
- **功能**:
  - 监视 Managed 集群上的策略
  - 创建/更新/删除 `spec.policy-templates` 中定义的对象
  - 验证模板并报告错误

#### Secret Sync Controller
- **管理器**: Hub Manager
- **功能**:
  - 从 Hub 同步 `policy-encryption-key` Secret
  - 支持加密策略模板

#### Gatekeeper Sync Controller
- **管理器**: 独立的第三个 Manager
- **功能**:
  - 动态启动/停止（基于 Gatekeeper 安装状态）
  - 将 Gatekeeper 约束违规同步到策略状态

### 3. Config Policy Controller (Managed 集群)

**位置**: `config-policy-controller`

**职责**: 评估和执行 ConfigurationPolicy 资源

**功能**:
- 检查 Kubernetes 对象是否符合期望状态
- 支持三种合规类型：
  - **musthave**: 对象必须存在且匹配指定字段（部分匹配）
  - **mustnothave**: 对象不能存在
  - **mustonlyhave**: 对象必须完全匹配（精确匹配）
- 支持两种模式：
  - **inform**: 仅报告合规性状态
  - **enforce**: 自动创建/更新/删除对象以符合策略
- 支持 Go 模板和 Sprig 函数
- 支持 Hub 模板（当配置时）
- 内置模板函数：
  - `fromSecret`: 从 Secret 读取数据
  - `fromConfigMap`: 从 ConfigMap 读取数据
  - `fromClusterClaim`: 从 ClusterClaim 读取值
  - `lookup`: 通用查找函数
  - `skipObject`: 条件性跳过对象创建

**评估流程**:
1. 策略根据评估间隔或监视事件被触发
2. 解析模板（如果存在）
3. 命名空间选择器确定目标命名空间
4. 对于每个对象模板：
   - 确定期望的对象
   - 与集群实际状态比较
   - 如果是 enforce 模式：创建/更新/删除对象
   - 如果是 inform 模式：仅报告合规性状态
5. 更新状态并发送事件

### 4. 其他 Policy Controllers

#### Cert Policy Controller
**位置**: `cert-policy-controller`
- 检查证书过期时间
- 验证证书链

#### IAM Policy Controller
**位置**: `iam-policy-controller`
- 管理 IAM 策略
- 验证用户权限和角色绑定

#### Operator Policy Controller
**位置**: `config-policy-controller/controllers/operatorpolicy_controller.go`
- 管理 OLM (Operator Lifecycle Manager) 操作符
- 处理 Subscription、OperatorGroup
- 监控操作符部署和 CSV 健康状况

## 核心 API 类型

### Policy (v1)

**文件**: `governance-policy-propagator/api/v1/policy_types.go`

```go
type Policy struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    
    Spec   PolicySpec
    Status PolicyStatus
}

type PolicySpec struct {
    Disabled           bool                // 禁用策略
    CopyPolicyMetadata *bool               // 是否复制元数据
    RemediationAction  RemediationAction   // inform/enforce
    PolicyTemplates    []*PolicyTemplate   // 策略模板数组
    Dependencies       []PolicyDependency  // 依赖关系
    HubTemplateOptions *HubTemplateOptions // Hub 模板选项
}

type PolicyTemplate struct {
    ObjectDefinition  runtime.RawExtension // 策略引擎资源
    ExtraDependencies []PolicyDependency   // 额外依赖
    IgnorePending     bool                 // 忽略 Pending 状态
}

type PolicyStatus struct {
    // ROOT POLICY 字段（仅在 Hub）
    Placement []*Placement                   // 放置信息
    Status    []*CompliancePerClusterStatus  // 每个集群的合规状态
    
    // REPLICATED POLICY 字段（仅在 Managed）
    ComplianceState ComplianceState      // 合规状态
    Details         []*DetailsPerTemplate // 每个模板的详细信息
}
```

**合规状态**:
- `Compliant`: 符合策略
- `NonCompliant`: 不符合策略
- `Pending`: 待评估

### PlacementBinding (v1)

**文件**: `governance-policy-propagator/api/v1/placementbinding_types.go`

```go
type PlacementBinding struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    
    PlacementRef     PlacementSubject  // 引用 Placement/PlacementRule
    Subjects         []Subject         // 绑定的 Policy/PolicySet
    BindingOverrides BindingOverrides  // 覆盖配置
    SubFilter        SubFilter         // 子过滤器
    Status           PlacementBindingStatus
}

type Subject struct {
    APIGroup string  // policy.open-cluster-management.io
    Kind     string  // Policy 或 PolicySet
    Name     string  // 策略名称
}

type PlacementSubject struct {
    APIGroup string  // cluster.open-cluster-management.io 或 apps.open-cluster-management.io
    Kind     string  // Placement 或 PlacementRule (已弃用)
    Name     string  // Placement 名称
}
```

### ConfigurationPolicy (v1)

**文件**: `config-policy-controller/api/v1/configurationpolicy_types.go`

```go
type ConfigurationPolicy struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    
    Spec   ConfigurationPolicySpec
    Status ConfigurationPolicyStatus
}

type ConfigurationPolicySpec struct {
    Severity           string              // low/medium/high
    RemediationAction  RemediationAction   // inform/enforce
    NamespaceSelector  NamespaceSelector   // 命名空间选择器
    ObjectTemplates    []*ObjectTemplate   // 对象模板
    EvaluationInterval EvaluationInterval  // 评估间隔
}

type ObjectTemplate struct {
    ComplianceType   string               // musthave/mustnothave/mustonlyhave
    ObjectDefinition runtime.RawExtension // Kubernetes 对象
}
```

## 策略分发流程

### 1. 策略创建阶段

```
用户创建 Policy → User Namespace (Hub)
        ↓
创建 Placement → 选择目标集群
        ↓
创建 PlacementBinding → 绑定 Policy 和 Placement
```

### 2. 策略传播阶段

```
RootPolicyReconciler (Hub)
        ↓
检查 PlacementBinding
        ↓
获取 Placement 决策（目标集群列表）
        ↓
为每个目标集群创建/更新 Replicated Policy → Cluster Namespace (Hub)
        ↓
更新 Root Policy Status (placement 信息)

        ↓

Spec Sync Controller (Managed)
        ↓
监视 Hub 上的 Cluster Namespace
        ↓
同步 Policy 到 Managed Cluster
```

### 3. 策略执行阶段

```
Template Sync Controller (Managed)
        ↓
读取 Policy.spec.policy-templates
        ↓
创建策略引擎对象 (ConfigurationPolicy, CertificatePolicy 等)

        ↓

Policy Controllers (Managed)
        ↓
评估策略规则
        ↓
[inform 模式] 报告合规状态
[enforce 模式] 创建/更新/删除 Kubernetes 资源
        ↓
更新策略状态和发送事件
```

### 4. 状态聚合阶段

```
Status Sync Controller (Managed)
        ↓
收集策略事件和状态
        ↓
更新 Managed Cluster 上的 Policy Status
        ↓
同步状态到 Hub Cluster

        ↓

RootPolicyStatusReconciler (Hub)
        ↓
从所有 Cluster Namespaces 收集状态
        ↓
聚合到 Root Policy Status.status
        ↓
计算整体合规状态
```

## 关键特性

### 1. Hub 模板

支持在策略中使用 `{{hub ... hub}}` 语法引用 Hub 集群资源：

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-example
spec:
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        spec:
          object-templates:
            - complianceType: musthave
              objectDefinition:
                apiVersion: v1
                kind: ConfigMap
                data:
                  hubValue: '{{hub fromConfigMap "default" "hub-config" "key" hub}}'
```

**解析位置**: Hub 集群的 ReplicatedPolicyReconciler
**权限控制**: 通过 `spec.hubTemplateOptions.serviceAccountName` 配置

### 2. 模板函数

**文件**: `config-policy-controller/controllers/configurationpolicy_controller.go`

在 ConfigurationPolicy 中支持：
- Go 模板语法
- Sprig 函数库
- 自定义函数：`fromSecret`, `fromConfigMap`, `fromClusterClaim`, `lookup`
- 上下文变量：`.Object`, `.ObjectName`, `.ObjectNamespace`

示例：
```yaml
objectDefinition:
  apiVersion: v1
  kind: ConfigMap
  data:
    secret-value: '{{ fromSecret "test" "mysecret" "password" | base64dec }}'
    cluster-id: '{{ fromClusterClaim "id.k8s.io" }}'
    service-ip: '{{ (lookup "v1" "Service" "default" "myservice").spec.clusterIP }}'
```

### 3. 策略依赖

支持策略间依赖关系：

```yaml
spec:
  dependencies:
    - apiVersion: policy.open-cluster-management.io/v1
      kind: Policy
      name: parent-policy
      namespace: default
      compliance: Compliant
```

只有当依赖策略达到指定合规状态时，当前策略才会被应用。

### 4. 策略加密

**文件**: 
- `governance-policy-propagator/controllers/propagator/encryption.go`
- `governance-policy-framework-addon/controllers/secretsync/`

支持加密敏感的策略模板：
- 使用 AES 加密
- 加密密钥存储在 `policy-encryption-key` Secret 中
- 自动密钥轮换（默认 30 天）
- SecretSyncController 同步密钥到 managed clusters

### 5. PolicySet

将多个策略组合为一组，支持：
- 批量部署
- 依赖顺序
- 统一管理

### 6. PolicyAutomation

基于策略违规自动触发 Ansible 作业：

```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: PolicyAutomation
metadata:
  name: auto-remediate
spec:
  policyRef: my-policy
  mode: once  # 或 everyEvent
  automationDef:
    type: AnsibleJob
    name: remediation-job
```

### 7. 动态监视（kubernetes-dependency-watches）

**使用位置**:
- Status Sync Controller
- Template Sync Controller

自动管理对策略引用资源的监视：
- 无需预先知道资源类型
- 动态添加/移除监视器
- 提高性能，减少不必要的协调

### 8. 指标（Metrics）

**文件**: `governance-policy-propagator/controllers/policymetrics/`

暴露 Prometheus 指标：
- 策略合规状态
- 传播失败计数
- 评估持续时间

## 开发和测试

### 本地开发环境设置

所有仓库都支持使用 KinD (Kubernetes in Docker) 进行本地开发：

```bash
# 创建 KinD 集群（Hub + Managed）
make kind-bootstrap-cluster-dev

# 构建镜像
make build-images

# 部署控制器
make kind-deploy-controller-dev

# 运行测试
make e2e-test
```

### 测试框架

- **单元测试**: Go 标准测试 + Gomega
- **E2E 测试**: Ginkgo/Gomega
- **测试环境**: KinD 集群（Hub + Managed）
- **覆盖率工具**: go test -cover

### 代码生成

所有仓库使用 Kubebuilder：
```bash
# 生成 CRD
make manifests

# 生成 DeepCopy 方法
make generate

# 生成部署 YAML
make generate-operator-yaml
```

## 技术栈

### 核心依赖

- **Controller Runtime**: v0.19.0 (Kubernetes controller framework)
- **Kubernetes**: v0.31.x (client-go, apimachinery)
- **OCM API**: Open Cluster Management API 库
- **go-template-utils**: v7 (Hub 模板解析)
- **kubernetes-dependency-watches**: 动态资源监视
- **Sprig**: v3 (模板函数库)

### 语言和工具

- **Go**: 1.24
- **Docker**: 容器构建
- **KinD**: 本地 Kubernetes 集群
- **Kustomize**: 部署清单管理
- **Ginkgo/Gomega**: 测试框架

## 安全考虑

1. **RBAC**: 所有控制器使用最小权限原则
2. **加密**: 支持策略模板加密
3. **密钥轮换**: 自动轮换加密密钥
4. **命名空间隔离**: 策略按命名空间隔离
5. **ServiceAccount**: Hub 模板可配置 ServiceAccount 限制权限

## 总结

stolostron Policy 框架是一个完整的多集群治理解决方案：

**优点**:
- 架构清晰，组件职责明确
- 支持多种策略引擎（可扩展）
- 强大的模板系统
- 自动化修复能力
- 完善的测试和文档

**设计亮点**:
- Hub-Spoke 架构实现中心化管理
- 双 Manager 模式实现跨集群操作
- 动态监视优化性能
- 策略复制机制支持大规模部署
- 状态聚合提供全局视图

**适用场景**:
- 多集群配置管理
- 合规性检查和报告
- 自动化修复
- 安全策略执行
- 证书管理
- 操作符生命周期管理

该框架是 Open Cluster Management (OCM) 生态系统的核心组件之一，为 Kubernetes 多集群环境提供了强大的治理能力。
