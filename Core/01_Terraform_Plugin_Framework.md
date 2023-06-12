# Terraform Plugin Framework
HashiCorp 推荐在 [protocol version 6](https://developer.hashicorp.com/terraform/plugin/terraform-plugin-protocol#protocol-version-6) 
或 [protocol version 5](https://developer.hashicorp.com/terraform/plugin/terraform-plugin-protocol#protocol-version-5) 
协议上通过 Terraform Plugin Framework 来开发 Provider 插件

推荐使用 Terraform Plugin Framework 来开发新的 Provider，因为该框架相比于 [Terraform Plugin SDKv2](https://developer.hashicorp.com/terraform/plugin/sdkv2) 具有明显的优势。
同时建议尽可能将现有的 Provider 迁移到该框架

可以参考 [Plugin Framework Benefits](https://developer.hashicorp.com/terraform/plugin/framework-benefits) 文档，以了解该框架如何简化程序开发；
参考 [Plugin Framework Features](https://developer.hashicorp.com/terraform/plugin/framework/migrating/benefits) 文档，以了解该框架与 SDKv2 之间的详细功能差异


## 开始
* 参考 [Terraform Plugin Framework 开发教程](https://developer.hashicorp.com/terraform/tutorials/providers-plugin-framework)
* 克隆 [terraform-provider-scaffolding-framework](https://github.com/hashicorp/terraform-provider-scaffolding-framework) 模板仓库

## 关键概念
* [Provider Servers](https://developer.hashicorp.com/terraform/plugin/framework/provider-servers) 封装了 Terraform 插件的详细信息，
并通过实现 Terraform 插件协议（ [Terraform Plugin Protocol](https://developer.hashicorp.com/terraform/plugin/how-terraform-works#terraform-plugin-protocol) ）来处理对 Provider、Resource 和 Datasource 操作的调用。
以二进制的方式存在，支持被 Terraform CLI 下载、启动和停止
    * 个人理解：Provider Servers 就一个 GRPC 的 Server，用来接收 Terraform CLI 的请求。收到请求后，调用 Provider 具体的 CRUD 方法
* [Provider](https://developer.hashicorp.com/terraform/plugin/framework/providers) 是定义 Resource 和 Datasource 的顶级抽象
* [Schemas](https://developer.hashicorp.com/terraform/plugin/framework/handling-data/schemas) 为 provider、resource 和配置块（provisioner configuration block）定义可用字段，并为 Terraform 提供有关这些字段的元数据。
* [Resource](https://developer.hashicorp.com/terraform/plugin/framework/resources) 是 Terraform 管理的基础设施的抽象实现，例如用 Terraform 管理计算实例、访问策略或磁盘
Providers 充当 Terraform 和 API 之间的转换层，支持在 HCL 配置文件中定义一个或者多个 Resource 模块
* [Datasource](https://developer.hashicorp.com/terraform/plugin/framework/data-sources) 是 Terraform 访问外部数据的抽象实现
Providers 中定义的 Datasource 告诉 Terraform 如何请求外部数据以及如何转换响应

## 测试与发布
* 学习为 Provider 编写[验收测试](https://developer.hashicorp.com/terraform/plugin/framework/acctests)
* 学习将 Provider [发布到 Terraform Registry](https://developer.hashicorp.com/terraform/registry/providers/publishing)

## 合并与转换
* 使用 [protocol version 5](https://developer.hashicorp.com/terraform/plugin/how-terraform-works#protocol-version-5) 协议将 Provider 与其他 SDKv2 Providers [结合](https://developer.hashicorp.com/terraform/plugin/mux/combining-protocol-version-5-providers)
* 使用 [protocol version 6](https://developer.hashicorp.com/terraform/plugin/how-terraform-works#protocol-version-6) 协议将 Provider 与其他框架实现的 Providers [结合](https://developer.hashicorp.com/terraform/plugin/how-terraform-works#protocol-version-6)