# Knowledge Base - stolostron Policy Framework

这个知识库包含 stolostron (Open Cluster Management) Policy 框架的详细技术分析文档。

## 📚 文档列表

### [Policy 框架实现分析](acm-policy.md)
- **内容**: Policy 框架的整体架构和核心组件
- **涵盖主题**:
  - 系统架构图和组件关系
  - Policy Propagator（策略传播器）
  - Framework Addon（框架插件）
  - Config Policy Controller（配置策略控制器）
  - 策略分发和状态聚合流程
  - Hub 模板、依赖关系、加密等高级特性
  - 开发和测试指南

### [Certificate & IAM Policy 详细分析](cert-iam-policy-analysis.md)
- **内容**: Certificate Policy 和 IAM Policy 控制器的深度分析
- **涵盖主题**:
  
  **Certificate Policy Controller**:
  - 证书过期监控机制
  - CA 证书和普通证书的差异化检查
  - SAN (Subject Alternative Name) 验证
  - API 定义和实现原理
  - 使用示例和最佳实践
  
  **IAM Policy Controller**:
  - ClusterRoleBinding 监控
  - 特权用户数量限制
  - 权限审计和合规检查
  - API 定义和实现原理
  - 使用示例和最佳实践

## 🎯 快速导航

### 初次学习
1. 从 [Policy 框架实现分析](acm-policy.md) 开始，了解整体架构
2. 阅读 [Certificate & IAM Policy 分析](cert-iam-policy-analysis.md) 了解具体控制器实现

### 查找特定信息
- **架构设计** → [acm-policy.md - 系统架构](acm-policy.md#系统架构)
- **策略分发流程** → [acm-policy.md - 策略分发流程](acm-policy.md#策略分发流程)
- **证书监控** → [cert-iam-policy-analysis.md - Certificate Policy](cert-iam-policy-analysis.md#certificate-policy-controller)
- **权限审计** → [cert-iam-policy-analysis.md - IAM Policy](cert-iam-policy-analysis.md#iam-policy-controller)

## 🔧 核心概念

### Policy 框架组件

| 组件 | 位置 | 功能 |
|------|------|------|
| **Policy Propagator** | Hub 集群 | 策略分发和状态聚合 |
| **Framework Addon** | Managed 集群 | 策略同步和模板处理 |
| **Config Policy Controller** | Managed 集群 | Kubernetes 对象合规检查 |
| **Cert Policy Controller** | Managed 集群 | 证书过期监控 |
| **IAM Policy Controller** | Managed 集群 | 权限绑定审计 |

### 策略类型

- **Policy**: 策略容器，包装其他策略引擎资源
- **ConfigurationPolicy**: 检查 Kubernetes 对象状态
- **CertificatePolicy**: 监控证书过期
- **IamPolicy**: 审计特权账户数量
- **OperatorPolicy**: 管理 OLM 操作符生命周期

## 🚀 实际应用场景

### 证书管理
使用 Certificate Policy 在生产环境中提前预警证书过期，确保服务不中断。

### 安全合规
使用 IAM Policy 限制 cluster-admin 数量，防止权限泛滥。

### 配置标准化
使用 Config Policy 确保所有集群的配置符合企业标准。

## 📖 相关资源

- [Open Cluster Management 官方文档](https://open-cluster-management.io/)
- [stolostron GitHub 组织](https://github.com/stolostron)
- [Policy 框架仓库](https://github.com/stolostron/governance-policy-framework)

## 🤝 贡献

欢迎贡献更多分析文档和使用案例！

## 📝 许可

文档内容基于对开源项目的分析，项目本身遵循 Apache 2.0 许可证。
