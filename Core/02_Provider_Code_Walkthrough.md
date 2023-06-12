# 代码走读
**Terraform 通过 Terraform Provider 与云提供商、SaaS 服务提供商等第三方通信。而 Terraform 与 Terraform Provider 之间的通信则使用 gRPC。Terraform 作为 gRPC 客户端运行，Provider 作为 gRPC 服务器运行**

Terraform 通过 Provider 定义的 Resource 来管理基础设施、通过定义的 Datasource 来读取数据。
使用者编写 HCL 配置来定义资源后， 由Terraform 将此配置传达给 Provider，再由 Provider 创建基础设施

如下的示例将会展示 Provider 各个组件之间的关系。 通常来说，Resource 和 Datasource 通过 API 与云提供商进行通信，
但本示例使用静态数据进行演示