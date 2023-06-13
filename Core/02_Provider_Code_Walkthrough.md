# 代码走读
**Terraform 通过 Terraform Provider 与云提供商、SaaS 服务提供商等第三方通信。而 Terraform 与 Terraform Provider 之间的通信则使用 gRPC。Terraform 作为 gRPC 客户端运行，Provider 作为 gRPC 服务器运行**

Terraform 通过 Provider 定义的 Resource 来管理基础设施、通过定义的 Datasource 来读取数据。
使用者编写 HCL 配置来定义资源后， 由Terraform 将此配置传达给 Provider，再由 Provider 创建基础设施

如下的示例将会展示 Provider 各个组件之间的关系。 通常来说，Resource 和 Datasource 通过 API 与云提供商进行通信，
但本示例使用静态数据进行演示

## Provider 核心组件
Terraform Provider 至少需要以下组件：
* [Provider Server]
* [Provider]
* [Resource] or [Datasource]

Provider 对 Resource(s) 和 Datasource(s) 进行了包装，并可配置一个客户端 Client，通过 API 与第三方服务通信。
Resource 用于管理基础设施对象；
Datasource 用于读取基础设施对象

## Provider Server
Terraform Provider 必须实现 gRPC server，用以在程序启动时支持与 Terraform-Core 的特定连接和握手处理。 
Terraform Provider 需要 Provider Server 才能：
* 给 Terraform Core 管理 Resources
* 给 Terraform Core 读取 Data sources

Notice: 这里重点理解，什么是 Terraform Core，Terraform Sever，Terraform Provider 以及 Resource 和 Datasource

个人理解：
1. Terraform Core 就是 Terraform 的核心程序，也就是我们安装的二进制运行程序。我们运行的 Terraform init/apply 命令，就是通过 Terraform Core 来执行的
2. Terraform Provider 就是各个厂商实现的插件，用来对资源进行增删改查。
当运行 Terraform 命令后，Terraform Core 会调用 Terraform Provider 与第三方 API 进行通信
3. Terraform Server 就是 Terraform Provider 中实现的一个 grpc Server。
Terraform Core 与 Terraform Provider 之间的通信走 grpc。Terraform Core 是 grpc Client，Terraform Provider 是 grpc Server
4. Resource 和 Datasource 就是 Terraform Provider 实现的资源

`main()` 函数用于定义 Terraform Provider Server。
`provider.New()` 返回一个实现了 `provider.provider` 接口的函数。
`provider.provider` 接口定义了用于从 Provider 获取 Resource 和 Datasource 的方法。
```go
package main

import (
    "context"
    "flag"
    "log"

    "github.com/hashicorp/terraform-plugin-framework/providerserver"

    "github.com/example_namespace/terraform-provider-example/internal/provider"
)

func main() {
    var debug bool

    flag.BoolVar(&debug, "debug", false, "set to true to run the provider with support for debuggers like delve")
    flag.Parse()

    opts := providerserver.ServeOpts{
        Address: "registry.terraform.io/example_namespace/example",
        Debug:   debug,
    }

    err := providerserver.Serve(context.Background(), provider.New(), opts)

    if err != nil {
        log.Fatal(err.Error())
    }
}

```