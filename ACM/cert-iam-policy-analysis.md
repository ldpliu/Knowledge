# Certificate Policy 和 IAM Policy 详细分析

## Certificate Policy Controller

### 功能概述

Certificate Policy Controller 是用于监控 Kubernetes 集群中证书过期状态的策略控制器。它确保指定命名空间中的所有证书不会在给定时间内过期。

### 核心职责

1. **证书过期监控**: 检查证书的有效期，确保证书在到期前得到更新
2. **CA 证书检查**: 对签名证书（CA）使用不同的过期阈值
3. **证书时长验证**: 确保证书的创建时长不超过指定限制
4. **SAN 模式验证**: 验证证书的主题备用名称（Subject Alternative Name）符合规范

### API 定义

#### CertificatePolicy CRD

```go
type CertificatePolicy struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    
    Spec   CertificatePolicySpec
    Status CertificatePolicyStatus
}
```

#### Spec 字段详解

```go
type CertificatePolicySpec struct {
    // 修复操作：目前只支持 "Inform"（通知）
    RemediationAction RemediationAction
    
    // 命名空间选择器：确定要检查的命名空间
    NamespaceSelector Target
    
    // 标签选择器：限制检查特定标签的 Secret
    LabelSelector map[string]NonEmptyString
    
    // 严重性级别：low, medium, high, critical
    Severity string
    
    // 最小有效期：证书剩余时间少于此值视为不合规
    MinDuration *metav1.Duration  // 默认 100h
    
    // 最大有效期：证书总时长超过此值视为不合规
    MaxDuration *metav1.Duration
    
    // CA 证书最小有效期：专门用于 CA 证书
    MinCADuration *metav1.Duration
    
    // CA 证书最大有效期：专门用于 CA 证书
    MaxCADuration *metav1.Duration
    
    // 允许的 SAN 模式：正则表达式，SAN 必须匹配
    AllowedSANPattern string
    
    // 禁止的 SAN 模式：正则表达式，SAN 不能匹配
    DisallowedSANPattern string
}
```

#### Status 字段详解

```go
type CertificatePolicyStatus struct {
    // 合规状态
    ComplianceState ComplianceState  // Compliant / NonCompliant
    
    // 合规详情：按命名空间组织
    CompliancyDetails map[string]CompliancyDetails
    
    // 历史记录：最近 10 条合规消息
    History []HistoryEvent
}

type CompliancyDetails struct {
    // 不合规证书数量
    NonCompliantCertificates uint
    
    // 不合规证书列表：证书名 -> 证书详情
    NonCompliantCertificatesList map[string]Cert
    
    // 人类可读的消息
    Message string
}

type Cert struct {
    Secret     string        // Secret 名称
    Expiration string        // 过期时间（UTC RFC 3339 格式）
    Expiry     time.Duration // 距离过期的时长
    CA         bool          // 是否为 CA 证书
    Duration   time.Duration // 证书总时长
    Sans       []string      // SAN 列表
}
```

### 实现原理

#### 1. 证书发现

**源码位置**: `controllers/certificatepolicy_utils.go`

控制器通过以下方式发现证书：

```
监视 Secret 资源
    ↓
应用 NamespaceSelector 过滤命名空间
    ↓
应用 LabelSelector 过滤 Secret
    ↓
从 Secret 中提取证书数据
    ↓
默认使用 'tls.crt' key
    ↓
可通过 'certificate_key_name' label 指定其他 key
```

#### 2. 证书解析

对每个 Secret 中的证书数据：

1. **解码证书**: 使用 Go 的 `crypto/x509` 库解析 PEM 格式证书
2. **提取信息**:
   - NotBefore / NotAfter: 证书有效期
   - Subject: 证书主体
   - Issuer: 签发者
   - IsCA: 是否为 CA 证书
   - DNSNames / IPAddresses: SAN 列表
3. **计算时长**:
   - Expiry = NotAfter - 当前时间
   - Duration = NotAfter - NotBefore

#### 3. 合规性评估

**源码位置**: `controllers/certificatepolicy_controller.go`

对每个证书进行多维度检查：

```go
// 伪代码示例
func evaluateCertificate(cert *x509.Certificate, spec CertificatePolicySpec) bool {
    isCA := cert.IsCA
    expiry := cert.NotAfter.Sub(time.Now())
    duration := cert.NotAfter.Sub(cert.NotBefore)
    
    // 检查 1: 最小有效期
    minDuration := spec.MinCADuration
    if !isCA || minDuration == nil {
        minDuration = spec.MinDuration
    }
    if expiry < minDuration {
        return false  // 不合规：即将过期
    }
    
    // 检查 2: 最大有效期
    maxDuration := spec.MaxCADuration
    if !isCA || maxDuration == nil {
        maxDuration = spec.MaxDuration
    }
    if maxDuration != nil && duration > maxDuration {
        return false  // 不合规：证书时长过长
    }
    
    // 检查 3: SAN 模式
    sans := getAllSANs(cert)
    if spec.AllowedSANPattern != "" {
        if !allMatch(sans, spec.AllowedSANPattern) {
            return false  // 不合规：SAN 不匹配允许模式
        }
    }
    if spec.DisallowedSANPattern != "" {
        if anyMatch(sans, spec.DisallowedSANPattern) {
            return false  // 不合规：SAN 匹配禁止模式
        }
    }
    
    return true  // 合规
}
```

#### 4. 状态更新

```
评估所有证书
    ↓
按命名空间分组结果
    ↓
构建 CompliancyDetails
    ↓
更新 CertificatePolicy Status
    ↓
发送 Kubernetes Event
    ↓
添加历史记录（限制 10 条）
```

#### 5. 协调循环

**源码位置**: `controllers/certificatepolicy_controller.go`

```go
func (r *CertificatePolicyReconciler) Reconcile(ctx context.Context, req ctrl.Request) {
    // 1. 获取 CertificatePolicy
    policy := &v1.CertificatePolicy{}
    err := r.Get(ctx, req.NamespacedName, policy)
    
    // 2. 获取目标命名空间列表
    namespaces := getSelectedNamespaces(policy.Spec.NamespaceSelector)
    
    // 3. 遍历每个命名空间
    for _, ns := range namespaces {
        // 4. 列出符合条件的 Secrets
        secrets := listSecrets(ns, policy.Spec.LabelSelector)
        
        // 5. 评估每个 Secret 中的证书
        for _, secret := range secrets {
            cert := parseCertificate(secret)
            compliant := evaluateCertificate(cert, policy.Spec)
            // 记录结果
        }
    }
    
    // 6. 更新策略状态
    updatePolicyStatus(policy)
    
    // 7. 发送事件
    recordEvent(policy)
}
```

### 使用示例

#### 基础证书监控

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: CertificatePolicy
metadata:
  name: cert-expiration-policy
  namespace: default
spec:
  namespaceSelector:
    include: ["default", "kube-*"]
    exclude: ["kube-system"]
  remediationAction: inform
  minimumDuration: 100h  # 证书过期前 100 小时开始告警
  severity: high
```

#### 高级 CA 证书检查

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: CertificatePolicy
metadata:
  name: ca-cert-policy
  namespace: default
spec:
  namespaceSelector:
    include: ["*"]
  labelSelector:
    cert-type: "ca"
  remediationAction: inform
  minimumDuration: 168h      # 普通证书：7 天
  minimumCADuration: 720h    # CA 证书：30 天
  maximumDuration: 8760h     # 最长 1 年
  maximumCADuration: 87600h  # CA 最长 10 年
  severity: critical
```

#### SAN 验证

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: CertificatePolicy
metadata:
  name: san-validation-policy
spec:
  namespaceSelector:
    include: ["default"]
  remediationAction: inform
  minimumDuration: 100h
  # 只允许 .example.com 或 .internal 域名
  allowedSANPattern: '^.*\.(example\.com|internal)$'
  # 禁止使用 localhost
  disallowedSANPattern: '^localhost$'
```

---

## IAM Policy Controller

### 功能概述

IAM Policy Controller 监控 Kubernetes 集群中的 IAM 集群角色绑定（ClusterRoleBinding），检测具有特定集群角色绑定的用户数量，并报告策略是否合规。

### 核心职责

1. **监控特权访问**: 追踪有多少用户具有 cluster-admin 或其他高权限角色
2. **限制管理员数量**: 确保管理员账户数量在可控范围内
3. **安全基线检查**: 作为安全基线的一部分，防止权限过度分配

### API 定义

#### IamPolicy CRD

```go
type IamPolicy struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    
    Spec   IamPolicySpec
    Status IamPolicyStatus
}
```

#### Spec 字段详解

```go
type IamPolicySpec struct {
    // 忽略的 ClusterRoleBinding 模式
    // 默认忽略所有以 "system:" 开头的绑定
    IgnoreClusterRoleBindings []NonEmptyString
    
    // 修复操作：目前只支持 "Inform"
    RemediationAction RemediationAction
    
    // 命名空间选择器（已废弃，不起作用）
    NamespaceSelector Target
    
    // 标签选择器
    LabelSelector map[string]string
    
    // 最大用户数：超过此值视为不合规
    MaxClusterRoleBindingUsers int  // 最小值为 1
    
    // 要检查的集群角色名称
    // 默认为 "cluster-admin"
    ClusterRole string
    
    // 严重性级别：low, medium, high, critical
    Severity string
}
```

#### Status 字段详解

```go
type IamPolicyStatus struct {
    // 合规状态
    ComplianceState ComplianceState  // Compliant / NonCompliant / UnknownCompliancy
    
    // 合规详情：集群角色 -> 用户列表
    CompliancyDetails map[string]CompliancyDetail
}

// CompliancyDetail: 用户类型 -> 用户名列表
type CompliancyDetail map[string][]string
```

### 实现原理

#### 1. ClusterRoleBinding 发现

**源码位置**: `controllers/iampolicy_controller.go`

```
监视 ClusterRoleBinding 资源
    ↓
过滤出引用目标 ClusterRole 的绑定
    ↓
应用 IgnoreClusterRoleBindings 过滤规则
    ↓
收集绑定的 Subjects（用户、组、ServiceAccount）
```

#### 2. 用户统计

对每个 ClusterRoleBinding：

```go
// 伪代码
func countUsers(bindings []rbacv1.ClusterRoleBinding, spec IamPolicySpec) map[string][]string {
    users := map[string][]string{
        "User":           {},
        "Group":          {},
        "ServiceAccount": {},
    }
    
    for _, binding := range bindings {
        // 1. 检查是否应该忽略
        if shouldIgnore(binding.Name, spec.IgnoreClusterRoleBindings) {
            continue
        }
        
        // 2. 检查是否引用目标角色
        if binding.RoleRef.Name != spec.ClusterRole {
            continue
        }
        
        // 3. 收集所有 subjects
        for _, subject := range binding.Subjects {
            switch subject.Kind {
            case "User":
                users["User"] = append(users["User"], subject.Name)
            case "Group":
                users["Group"] = append(users["Group"], subject.Name)
            case "ServiceAccount":
                sa := subject.Namespace + "/" + subject.Name
                users["ServiceAccount"] = append(users["ServiceAccount"], sa)
            }
        }
    }
    
    return users
}
```

#### 3. 合规性评估

**源码位置**: `controllers/iampolicy_utils.go`

```go
func evaluateCompliance(users map[string][]string, maxUsers int) (ComplianceState, string) {
    totalUsers := len(users["User"]) + len(users["Group"]) + len(users["ServiceAccount"])
    
    if totalUsers > maxUsers {
        msg := fmt.Sprintf(
            "Found %d cluster role bindings (max allowed: %d). "+
            "Users: %v, Groups: %v, ServiceAccounts: %v",
            totalUsers, maxUsers,
            users["User"], users["Group"], users["ServiceAccount"],
        )
        return NonCompliant, msg
    }
    
    return Compliant, fmt.Sprintf(
        "Found %d cluster role bindings (within limit of %d)",
        totalUsers, maxUsers,
    )
}
```

#### 4. 协调循环

```go
func (r *IamPolicyReconciler) Reconcile(ctx context.Context, req ctrl.Request) {
    // 1. 获取 IamPolicy
    policy := &v1.IamPolicy{}
    err := r.Get(ctx, req.NamespacedName, policy)
    
    // 2. 获取目标集群角色（默认 cluster-admin）
    targetRole := policy.Spec.ClusterRole
    if targetRole == "" {
        targetRole = "cluster-admin"
    }
    
    // 3. 列出所有 ClusterRoleBindings
    bindings := &rbacv1.ClusterRoleBindingList{}
    err = r.List(ctx, bindings)
    
    // 4. 统计用户
    users := countUsers(bindings.Items, policy.Spec)
    
    // 5. 评估合规性
    state, message := evaluateCompliance(users, policy.Spec.MaxClusterRoleBindingUsers)
    
    // 6. 更新状态
    policy.Status.ComplianceState = state
    policy.Status.CompliancyDetails = map[string]CompliancyDetail{
        targetRole: users,
    }
    
    // 7. 发送事件
    recordEvent(policy, message)
    
    return ctrl.Result{}, nil
}
```

#### 5. 忽略规则处理

```go
func shouldIgnore(bindingName string, ignorePatterns []NonEmptyString) bool {
    // 默认忽略 system: 开头的绑定
    if strings.HasPrefix(bindingName, "system:") {
        return true
    }
    
    // 检查用户自定义的忽略模式
    for _, pattern := range ignorePatterns {
        matched, _ := regexp.MatchString(string(pattern), bindingName)
        if matched {
            return true
        }
    }
    
    return false
}
```

### 使用示例

#### 基础管理员数量限制

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: IamPolicy
metadata:
  name: limit-cluster-admins
  namespace: default
spec:
  remediationAction: inform
  severity: high
  # 限制最多 5 个 cluster-admin
  maxClusterRoleBindingUsers: 5
  # 默认检查 cluster-admin 角色
```

#### 自定义角色检查

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: IamPolicy
metadata:
  name: limit-edit-users
  namespace: default
spec:
  remediationAction: inform
  severity: medium
  # 检查 edit 角色
  clusterRole: edit
  # 最多 10 个用户
  maxClusterRoleBindingUsers: 10
  # 忽略特定的绑定
  ignoreClusterRoleBindings:
    - "^system:.*"           # 系统绑定
    - "^oidc:.*"             # OIDC 相关
    - "^pipeline-.*"         # 流水线相关
```

#### 严格的管理员控制

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: IamPolicy
metadata:
  name: strict-admin-policy
  namespace: default
spec:
  remediationAction: inform
  severity: critical
  clusterRole: cluster-admin
  # 只允许 1 个紧急管理员账户
  maxClusterRoleBindingUsers: 1
  ignoreClusterRoleBindings:
    - "^system:.*"
    - "^kube-.*"
```

---

## 工作流程对比

### Certificate Policy 工作流程

```
1. CertificatePolicy 创建
    ↓
2. Template Sync 在 Managed Cluster 创建 CertificatePolicy
    ↓
3. Certificate Policy Controller 启动
    ↓
4. 监视 Secret 资源
    ↓
5. 定期扫描（默认每 10 分钟）
    ↓
6. 解析证书并评估
    ↓
7. 更新 CertificatePolicy Status
    ↓
8. Status Sync 同步到 Hub
    ↓
9. 显示在 Root Policy
```

### IAM Policy 工作流程

```
1. IamPolicy 创建
    ↓
2. Template Sync 在 Managed Cluster 创建 IamPolicy
    ↓
3. IAM Policy Controller 启动
    ↓
4. 监视 ClusterRoleBinding 资源
    ↓
5. 当 ClusterRoleBinding 变化时触发
    ↓
6. 统计用户数量并评估
    ↓
7. 更新 IamPolicy Status
    ↓
8. Status Sync 同步到 Hub
    ↓
9. 显示在 Root Policy
```

---

## 技术实现细节

### Certificate Policy Controller

#### 控制器架构

**文件**: `controllers/certificatepolicy_controller.go`

```go
type CertificatePolicyReconciler struct {
    client.Client
    Scheme         *runtime.Scheme
    Recorder       events.EventRecorder
    InstanceName   string
    TargetK8sClient kubernetes.Interface
}

func (r *CertificatePolicyReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&policyv1.CertificatePolicy{}).
        Watches(
            &source.Kind{Type: &corev1.Secret{}},
            handler.EnqueueRequestsFromMapFunc(r.secretToPolicy),
        ).
        Complete(r)
}
```

#### 关键功能

1. **证书解析**: 使用 `crypto/x509` 和 `encoding/pem`
2. **正则表达式验证**: 使用 `regexp` 包验证 SAN
3. **时间计算**: 使用 `time` 包计算过期时间
4. **事件记录**: 使用 Kubernetes Events API

#### 性能优化

- 使用 Watch 机制而非轮询
- 缓存证书解析结果
- 批量处理 Secret
- 限制历史记录数量（10 条）

### IAM Policy Controller

#### 控制器架构

**文件**: `controllers/iampolicy_controller.go`

```go
type IamPolicyReconciler struct {
    client.Client
    Scheme       *runtime.Scheme
    Recorder     events.EventRecorder
    InstanceName string
}

func (r *IamPolicyReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&policyv1.IamPolicy{}).
        Watches(
            &source.Kind{Type: &rbacv1.ClusterRoleBinding{}},
            handler.EnqueueRequestsFromMapFunc(r.clusterRoleBindingToPolicy),
        ).
        Complete(r)
}
```

#### 关键功能

1. **RBAC 检查**: 直接访问 ClusterRoleBinding 资源
2. **模式匹配**: 使用正则表达式过滤绑定
3. **用户分类**: 区分 User、Group、ServiceAccount
4. **去重**: 防止重复计数

#### 性能优化

- 使用 Watch 而非 List
- 缓存 ClusterRoleBinding 列表
- 快速过滤忽略的绑定
- 最小化 API 调用

---

## 对比总结

| 特性 | Certificate Policy | IAM Policy |
|------|-------------------|------------|
| **监控对象** | Secret (证书) | ClusterRoleBinding |
| **检查频率** | 定期扫描 + Secret 变更触发 | ClusterRoleBinding 变更触发 |
| **合规类型** | 基于时间和模式 | 基于数量 |
| **复杂度** | 高（证书解析、多维度检查） | 中（简单计数） |
| **修复操作** | 仅 Inform | 仅 Inform |
| **典型用例** | 证书管理、安全合规 | 权限审计、访问控制 |
| **性能影响** | 中等（证书解析开销） | 低（简单统计） |
| **配置灵活性** | 高（多个阈值、SAN 验证） | 中（主要是数量限制） |

---

## 实际应用场景

### Certificate Policy

1. **证书过期监控**: 在生产环境中提前预警证书过期
2. **合规性检查**: 确保证书符合企业安全标准（如最大有效期）
3. **自动化审计**: 定期检查所有命名空间的证书状态
4. **SAN 合规**: 验证证书的域名符合组织规范

### IAM Policy

1. **管理员账户审计**: 监控 cluster-admin 数量，防止权限泛滥
2. **安全基线**: 作为 CIS Kubernetes Benchmark 的一部分
3. **合规报告**: 生成特权账户的审计报告
4. **权限收敛**: 识别并清理过多的管理员权限

---

## 最佳实践

### Certificate Policy

1. **分层监控**: 为不同类型的证书设置不同的策略
   - CA 证书使用更长的 minimumCADuration
   - 应用证书使用标准的 minimumDuration

2. **命名空间隔离**: 使用 namespaceSelector 精确控制检查范围

3. **标签管理**: 使用 labelSelector 对证书进行分类管理

4. **告警分级**: 根据证书重要性设置不同的 severity

### IAM Policy

1. **最小权限原则**: 将 maxClusterRoleBindingUsers 设置为尽可能小的值

2. **定期审查**: 结合策略报告定期审查管理员账户

3. **自动化清理**: 基于策略结果触发自动化流程清理不必要的权限

4. **多层防护**: 为不同的高权限角色创建多个策略

---

## 集成示例

### 在 Policy 框架中使用

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: security-baseline
  namespace: policies
spec:
  remediationAction: inform
  disabled: false
  policy-templates:
    # 证书策略
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: CertificatePolicy
        metadata:
          name: cert-expiration-check
        spec:
          namespaceSelector:
            include: ["*"]
          remediationAction: inform
          minimumDuration: 168h
          severity: high
    
    # IAM 策略
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: IamPolicy
        metadata:
          name: admin-limit-check
        spec:
          remediationAction: inform
          severity: critical
          maxClusterRoleBindingUsers: 3
          clusterRole: cluster-admin
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: all-clusters
  namespace: policies
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: environment
              operator: Exists
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: security-baseline-binding
  namespace: policies
placementRef:
  name: all-clusters
  kind: Placement
  apiGroup: cluster.open-cluster-management.io
subjects:
  - name: security-baseline
    kind: Policy
    apiGroup: policy.open-cluster-management.io
```

这样配置后，证书和 IAM 检查将自动应用到所有匹配的集群，并将合规状态聚合到 Hub 集群。
