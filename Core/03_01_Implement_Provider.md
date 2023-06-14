# 通过 Terraform Plugin Framework 实现 Provider
在本教程中，您将使用 Terraform Plugin Framework 针对名为 HashiCups 的虚构咖啡店应用程序的 API 编写自定义 Provider。
通过该过程，您将了解如何创建 Datasource、向 HashiCups 客户端提供身份验证、provider 如何将目标 API 映射到 Terraform 以实现资源的创建、读取、更新和删除

在本教程中，您将配置 Terraform Provider 开发环境，并创建一个可以与 Terraform 通信的 Provider。为此，您将会：
1. 初始化开发环境  
克隆 terraform-provider-scaffolding-framework 仓库并添加额外的测试文件。该仓库含通用 Terraform Provider 的脚手架
2. 定义 provider 类型  
实现 Resource 和 Datasource
3. 定义 provider server  
Provider 在启动时与 Terraform 进行通信

## 前置
* Go 1.19+
* Terraform v1.0.3+
* Docker and Docker Compose

## 初始化开发环境
克隆 terraform-provider-scaffolding-framework 代码仓库
```
git clone https://github.com/hashicorp/terraform-provider-scaffolding-framework
```
文件夹重命名
```
mv terraform-provider-scaffolding-framework terraform-provider-hashicups-pf
```
进入文件夹
```
cd terraform-provider-hashicups-pf
```
重命名 go.mod
```
go mod edit -module terraform-provider-hashicups-pf
```
go mod tidy
```
go mod tidy
```
更换 main.go 里的 import
```
import (
    "context"
    "flag"
    "log"

    "github.com/hashicorp/terraform-plugin-framework/providerserver"

    "terraform-provider-hashicups-pf/internal/provider"
)

```
创建 docker_compose 文件夹
```
mkdir docker_compose
```
创建 docker_compose/conf.json 文件
```
{
  "db_connection": "host=db port=5432 user=postgres password=password dbname=products sslmode=disable",
  "bind_address": "0.0.0.0:9090",
  "metrics_address": "localhost:9102"
}
```
创建 docker_compose/docker-compose.yml 文件
```yaml
version: '3.7'
services:
  api:
    image: "hashicorpdemoapp/product-api:v0.0.22"
    ports:
      - "19090:9090"
    volumes:
      - ./conf.json:/config/config.json
    environment:
      CONFIG_FILE: '/config/config.json'
    depends_on:
      - db
  db:
    image: "hashicorpdemoapp/product-api-db:v0.0.22"
    ports:
      - "15432:5432"
    environment:
      POSTGRES_DB: 'products'
      POSTGRES_USER: 'postgres'
      POSTGRES_PASSWORD: 'password'
```

## 定义 provider
Provider 需要实现 `provider.Provider` 接口

实现此接口需要：

1. Metadata Method：定义 Provider 类型名称，用以包含在每个数据源和资源类型名称中。例如，名为 hashicups_order 的 Resource 的 Provider 类型名称为 hashicups。
(说白了，也就是为所有 Resource 和 Datasource 定义一个通用前缀)
2. Schema Method：定义 provider-level 的配置 Schema。在这些教程的后面，您将更新此方法以支持 HashiCups API Token 和 Endpoint
3. Configure Method：用于为 Resource 和 Datasource 配置共享客户端
4. Datasource Method：定义 DataSource
5. Resource Method：定义 Resource

进入 internal/provider 目录，该目录将包含除 provider server 之外所有 Go 代码
打开 internal/provider/provider.go 文件并将现有代码替换为以下内容
```go
package provider

import (
    "context"

    "github.com/hashicorp/terraform-plugin-framework/datasource"
    "github.com/hashicorp/terraform-plugin-framework/provider"
    "github.com/hashicorp/terraform-plugin-framework/provider/schema"
    "github.com/hashicorp/terraform-plugin-framework/resource"
)

// Ensure the implementation satisfies the expected interfaces.
var (
    _ provider.Provider = &hashicupsProvider{}
)

// New is a helper function to simplify provider server and testing implementation.
func New(version string) func() provider.Provider {
    return func() provider.Provider {
        return &hashicupsProvider{
            version: version,
        }
    }
}

// hashicupsProvider is the provider implementation.
type hashicupsProvider struct {
    // version is set to the provider version on release, "dev" when the
    // provider is built and ran locally, and "test" when running acceptance
    // testing.
    version string
}

// Metadata returns the provider type name.
func (p *hashicupsProvider) Metadata(_ context.Context, _ provider.MetadataRequest, resp *provider.MetadataResponse) {
    resp.TypeName = "hashicups"
    resp.Version = p.version
}

// Schema defines the provider-level schema for configuration data.
func (p *hashicupsProvider) Schema(_ context.Context, _ provider.SchemaRequest, resp *provider.SchemaResponse) {
    resp.Schema = schema.Schema{}
}

// Configure prepares a HashiCups API client for data sources and resources.
func (p *hashicupsProvider) Configure(ctx context.Context, req provider.ConfigureRequest, resp *provider.ConfigureResponse) {
}

// DataSources defines the data sources implemented in the provider.
func (p *hashicupsProvider) DataSources(_ context.Context) []func() datasource.DataSource {
    return nil
}

// Resources defines the resources implemented in the provider.
func (p *hashicupsProvider) Resources(_ context.Context) []func() resource.Resource {
    return nil
}
```

## 实现 Provider Server
Terraform Provider 本质上是服务端进程。Terraform 对 Resource 和 Datasource 的处理都是通过与 Terraform Provider 之间的交互实现的

实现 Provider Server 需要遵循以下步骤：

启动 Provider Server 进程。 在 Go 语言中通过 `main` 函数启动一个服务端，用来监听 Terraform 的请求

Terraform Plugin Framework 还支持一些其他功能，例如启用对调试工具的支持。 您不会在这些教程中实现此功能

打开 terraform-provider-hashicups-pf 仓库根目录中的 `main.go` 文件，并将 `main` 函数替换为以下内容
```go
func main() {
    var debug bool

    flag.BoolVar(&debug, "debug", false, "set to true to run the provider with support for debuggers like delve")
    flag.Parse()

    opts := providerserver.ServeOpts{
        // NOTE: This is not a typical Terraform Registry provider address,
        // such as registry.terraform.io/hashicorp/hashicups. This specific
        // provider address is used in these tutorials in conjunction with a
        // specific Terraform CLI configuration for manual development testing
        // of this provider.
        Address: "hashicorp.com/edu/hashicups-pf",
        Debug:   debug,
    }

    err := providerserver.Serve(context.Background(), provider.New(version), opts)

    if err != nil {
        log.Fatal(err.Error())
    }
}
```

## 验证 Provider
手动运行
```go
$ go run main.go
This binary is a plugin. These are not meant to be executed directly.
Please execute the program that consumes these plugins, which will
load any plugins automatically
exit status 1
```
这将返回一条错误消息，因为这不是 Terraform 通常启动 Provider 的方式，但该错误表明 Go 能够编译并运行您的 Provider

## 本地安装准备
当您运行 `terraform init` 时，Terraform 会安装并校验 Provider。Terraform 将从 Provider Registry 或本地 Registry 下载 Provider。
在本地开发 Provider 时，您需要针对 Provider 的本地开发版本测试 Terraform 配置，但是本地开发的版本没有相关的版本号或者一个官方的 checksums，导致无法校验成功

为此，Terraform 允许在 `.terraformrc` 配置文件中设置 `dev_overrides` 块来使用本地 Provider 构建

Terraform 将在您的用户主目录（home directory）中搜索 `.terraformrc` 文件并应用

Mac
首先，找到 Go 安装二进制文件的 `GOBIN` 路径
```
$ go env GOBIN
/Users/<Username>/go/bin
```
如果未设置 `GOBIN` 环境变量，请使用默认路径 `/Users/<Username>/go/bin`

在您的用户主目录 (`~`) 中创建一个名为 `.terraformrc` 的新文件，然后在下面添加 `dev_overrides` 块。
将 `<PATH>` 更改为从上面的 `go env GOBIN` 命令返回的值
```
provider_installation {

  dev_overrides {
      "hashicorp.com/edu/hashicups-pf" = "<PATH>"
  }

  # For all other providers, install them directly from their origin provider
  # registries as normal. If you omit this, Terraform will _only_ use
  # the dev_overrides block, and so no other providers will be available.
  direct {}
}
```

## 本地安装并验证
您的 Terraform CLI 现在已准备好使用 `GOBIN` 路径中本地安装的 Provider。 
在仓库根目录下执行 go install 命令将提供 Provider 编译为二进制文件并将其安装在您的 `GOBIN` 路径中
```
go install .
```
创建一个 examples/provider-install-verification 目录
```
mkdir examples/provider-install-verification && cd "$_"
```
创建 main.tf 文件
```hcl
terraform {
  required_providers {
    hashicups = {
      source = "hashicorp.com/edu/hashicups-pf"
    }
  }
}

provider "hashicups" {}

data "hashicups_coffees" "example" {}
```
此目录中的 `main.tf` 使用了 Provider 尚不支持的 `hashicups_coffees` Datasource。您将在以后的教程中将实现此数据源

运行 Terraform Plan 将显示 Provider 被覆盖的警告，以及有关缺少 Datasource 的错误。
即使出现错误，这也验证了 Terraform 能够成功启动本地安装的 Provider 并在开发环境中与其交互
```
$ terraform plan
╷
│ Warning: Provider development overrides are in effect
│
│ The following provider development overrides are set in the CLI
│ configuration:
│  - hashicorp.com/edu/hashicups-pf in /Users/<Username>/go/bin
│
│ The behavior may therefore not match any released version of the provider and
│ applying changes may cause the state to become incompatible with published
│ releases.
╵
╷
│ Error: Invalid data source
│
│   on main.tf line 11, in data "hashicups_coffees" "example":
│   11: data "hashicups_coffees" "example" {}
│
│ The provider hashicorp.com/edu/hashicups-pf does not support data source
│ "hashicups_coffees".
╵
```

Notice: 无需执行 terraform init，
dev_overrides 参数告诉 Terraform 命令（例如 terraform apply）忽略由 terraform init 填充的依赖项锁定文件和 Provider 缓存目录，并改用您指定的路径。
它不会影响 terraform init 本身的行为

参考：https://stackoverflow.com/questions/74402085/terraform-cannot-init-custom-provider-written-in-go