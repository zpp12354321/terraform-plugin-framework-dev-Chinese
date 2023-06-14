# 配置 Provider Client
在本教程中，您将通过 Terraform Provider 的配置文件或环境变量初始化 HashiCups API 客户端，该客户端可在全局使用。
为此，您将：

1. 定义 Provider Schema  
通过 Terraform 配置文件来初始化 Client 所需的身份认证和 Host 信息
2. 定义 Provider 数据模型  
通过 Go 代码来定义模型
3. 实现 Provider 的 Configure 方法  
通过数据模型或者环境变量读取 Terraform 的配置信息来初始化 Client，如果缺少配置，需要抛出错误。
创建好的 Client 可用于 Datasource 和 Resource
4. 验证配置  
确保 Provider 的配置符合预期

## 前置
* Go 1.19+
* Terraform v1.0.3+
* Docker and Docker Compose

## 实现 Provider Schema
Terraform Plugin Framework 使用 provider 的 Schema 方法来定义可接受的配置属性名称和类型。

HashiCups 客户端需要主机、用户名和密码才能正确配置

打开 `internal/provider/provider.go` 文件，替换 Schema 为如下内容
```go
// Schema defines the provider-level schema for configuration data.
func (p *hashicupsProvider) Schema(_ context.Context, _ provider.SchemaRequest, resp *provider.SchemaResponse) {
    resp.Schema = schema.Schema{
        Attributes: map[string]schema.Attribute{
            "host": schema.StringAttribute{
                Optional: true,
            },
            "username": schema.StringAttribute{
                Optional: true,
            },
            "password": schema.StringAttribute{
                Optional:  true,
                Sensitive: true,
            },
        },
    }
}
```

## 实现 Provider 数据模型
Terraform Plugin Framework 使用 `tfsdk` 结构字段标记的 Go 结构体来映射 Schema 中定义模型。
结构中的类型必须与我们之前定义的 Schema 中类型一致。

将以下内容添加到 internal/provider/provider.go
```go
// hashicupsProviderModel maps provider schema data to a Go type.
type hashicupsProviderModel struct {
    Host     types.String `tfsdk:"host"`
    Username types.String `tfsdk:"username"`
    Password types.String `tfsdk:"password"`
}
```
注意 import "github.com/hashicorp/terraform-plugin-framework/types"

## 实现 Configure 方法
Provider 使用 Configure 方法从 Terraform 配置或环境变量中读取 HashiCups API 客户端所需的参数。
验证成功之后，将会在此初始化 HashiCups API Client。

Configure 方法的实现需要遵循以下步骤：
* 从配置中获取目标配置参数，并将其转换为 providerModel 数据结构
* 检查配置参数，避免客户端被错误配置
* 从环境变量中获取参数，如果参数已经被 Terraform 配置文件所配置，就忽略
* 创建 HashiCups API 客户端
* 将创建好的 HashiCups API 客户端存储，以供 Datasource 和 Resource 使用

替换 `internal/provider/provider.go` 中的 `Configure` 方法
```go
func (p *hashicupsProvider) Configure(ctx context.Context, req provider.ConfigureRequest, resp *provider.ConfigureResponse) {
    // Retrieve provider data from configuration
    var config hashicupsProviderModel
    diags := req.Config.Get(ctx, &config)
    resp.Diagnostics.Append(diags...)
    if resp.Diagnostics.HasError() {
        return
    }

    // If practitioner provided a configuration value for any of the
    // attributes, it must be a known value.

    if config.Host.IsUnknown() {
        resp.Diagnostics.AddAttributeError(
            path.Root("host"),
            "Unknown HashiCups API Host",
            "The provider cannot create the HashiCups API client as there is an unknown configuration value for the HashiCups API host. "+
                "Either target apply the source of the value first, set the value statically in the configuration, or use the HASHICUPS_HOST environment variable.",
        )
    }

    if config.Username.IsUnknown() {
        resp.Diagnostics.AddAttributeError(
            path.Root("username"),
            "Unknown HashiCups API Username",
            "The provider cannot create the HashiCups API client as there is an unknown configuration value for the HashiCups API username. "+
                "Either target apply the source of the value first, set the value statically in the configuration, or use the HASHICUPS_USERNAME environment variable.",
        )
    }

    if config.Password.IsUnknown() {
        resp.Diagnostics.AddAttributeError(
            path.Root("password"),
            "Unknown HashiCups API Password",
            "The provider cannot create the HashiCups API client as there is an unknown configuration value for the HashiCups API password. "+
                "Either target apply the source of the value first, set the value statically in the configuration, or use the HASHICUPS_PASSWORD environment variable.",
        )
    }

    if resp.Diagnostics.HasError() {
        return
    }

    // Default values to environment variables, but override
    // with Terraform configuration value if set.

    host := os.Getenv("HASHICUPS_HOST")
    username := os.Getenv("HASHICUPS_USERNAME")
    password := os.Getenv("HASHICUPS_PASSWORD")

    // 如果 config 里存在，就用 config 里的，而不用环境变量里的
    if !config.Host.IsNull() {
        host = config.Host.ValueString()
    }

    if !config.Username.IsNull() {
        username = config.Username.ValueString()
    }

    if !config.Password.IsNull() {
        password = config.Password.ValueString()
    }

    // If any of the expected configurations are missing, return
    // errors with provider-specific guidance.

    if host == "" {
        resp.Diagnostics.AddAttributeError(
            path.Root("host"),
            "Missing HashiCups API Host",
            "The provider cannot create the HashiCups API client as there is a missing or empty value for the HashiCups API host. "+
                "Set the host value in the configuration or use the HASHICUPS_HOST environment variable. "+
                "If either is already set, ensure the value is not empty.",
        )
    }

    if username == "" {
        resp.Diagnostics.AddAttributeError(
            path.Root("username"),
            "Missing HashiCups API Username",
            "The provider cannot create the HashiCups API client as there is a missing or empty value for the HashiCups API username. "+
                "Set the username value in the configuration or use the HASHICUPS_USERNAME environment variable. "+
                "If either is already set, ensure the value is not empty.",
        )
    }

    if password == "" {
        resp.Diagnostics.AddAttributeError(
            path.Root("password"),
            "Missing HashiCups API Password",
            "The provider cannot create the HashiCups API client as there is a missing or empty value for the HashiCups API password. "+
                "Set the password value in the configuration or use the HASHICUPS_PASSWORD environment variable. "+
                "If either is already set, ensure the value is not empty.",
        )
    }

    if resp.Diagnostics.HasError() {
        return
    }

    // Create a new HashiCups client using the configuration values
    client, err := hashicups.NewClient(&host, &username, &password)
    if err != nil {
        resp.Diagnostics.AddError(
            "Unable to Create HashiCups API Client",
            "An unexpected error occurred when creating the HashiCups API client. "+
                "If the error is not clear, please contact the provider developers.\n\n"+
                "HashiCups Client Error: "+err.Error(),
        )
        return
    }

    // Make the HashiCups client available during DataSource and Resource
    // type Configure methods.
    resp.DataSourceData = client
    resp.ResourceData = client
}
```
替换 import 
```go
import (
       "context"
       "os"

       "github.com/hashicorp-demoapp/hashicups-client-go"
       "github.com/hashicorp/terraform-plugin-framework/datasource"
       "github.com/hashicorp/terraform-plugin-framework/path"
       "github.com/hashicorp/terraform-plugin-framework/provider"
       "github.com/hashicorp/terraform-plugin-framework/provider/schema"
       "github.com/hashicorp/terraform-plugin-framework/resource"
       "github.com/hashicorp/terraform-plugin-framework/types"
)
```
更新 go 依赖
```go
$ go mod tidy
```
安装 Provider
```go
$ go install .
```
Notice: 这里也是一样，你是什么厂商的 Provider，就初始化自己厂商的 Client 就可以了

## 在本地启动 HashiCups 服务
这一步主要是确保刚刚初始化的 Client 可以链接上

导航到 `docker_compose` 文件夹
```
cd docker_compose
```
启动 docker 容器，端口为 19090
```
docker-compose up
```
保持终端窗口打开，HashiCup 服务端将会在终端窗口打印日志信息。
通过以下命令验证 HashiCup 服务是否正常
```
$ curl localhost:19090/health/readyz
ok
```

## 创建一个 HashiCups 用户
HashiCups 需要用户名和密码来生成 JWT 用于身份验证。您需要使用这个用户来向 HashiCups 验证自己的行为。

创建一个用户，名称为 `education`，密码为 `test123`
```
$ curl -X POST localhost:19090/signup -d '{"username":"education", "password":"test123"}'
```
设置 `HASHICUPS_TOKEN` 环境变量为你取到的 Token，之后课程可能会用到这个环境变量
```
$ export HASHICUPS_TOKEN=ey...
```
终端将会记录此次日志
```
api_1  | 2020-12-10T09:19:50.601Z [INFO]  Handle User | signup
```

## 实现临时的 Datasource
实现一个临时的 Datasource 来帮助我们验证客户端的初始化。不用深究实现细节，之后的课程会讲到

创建 `internal/provider/coffees_data_source.go` 文件
```go
package provider

import (
    "context"

    "github.com/hashicorp/terraform-plugin-framework/datasource"
    "github.com/hashicorp/terraform-plugin-framework/datasource/schema"
)

func NewCoffeesDataSource() datasource.DataSource {
    return &coffeesDataSource{}
}

type coffeesDataSource struct{}

func (d *coffeesDataSource) Metadata(_ context.Context, req datasource.MetadataRequest, resp *datasource.MetadataResponse) {
    resp.TypeName = req.ProviderTypeName + "_coffees"
}

func (d *coffeesDataSource) Schema(_ context.Context, _ datasource.SchemaRequest, resp *datasource.SchemaResponse) {
    resp.Schema = schema.Schema{}
}

func (d *coffeesDataSource) Read(ctx context.Context, req datasource.ReadRequest, resp *datasource.ReadResponse) {
}
```
在 `internal/provider/provider.go` 文件中，添加如下内容
```go
// DataSources defines the data sources implemented in the provider.
func (p *hashicupsProvider) DataSources(_ context.Context) []func() datasource.DataSource {
    return []func() datasource.DataSource {
        NewCoffeesDataSource,
    }

```
构建和安装 Provider
```
$ go install .
```

## 验证 Provider 的配置
导航到 `examples/provider-install-verification` 文件夹
```
$ cd examples/provider-install-verification
```
当前 main.tf 文件没有对 Provider 进行配置(provider "hashicups" {} 为空)
```
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
执行 Terraform plan，Terraform 将会显示缺失参数的警告
```
$ terraform plan
##...
╷
│ Error: Missing HashiCups API Host
│ 
│   with provider["hashicorp.com/edu/hashicups-pf"],
│   on main.tf line 9, in provider "hashicups":
│    9: provider "hashicups" {}
│ 
│ The provider cannot create the HashiCups API client as there is a missing or
│ empty value for the HashiCups API host. Set the host value in the
│ configuration or use the HASHICUPS_HOST environment variable. If either is
│ already set, ensure the value is not empty.
╵
╷
│ Error: Missing HashiCups API Username
│ 
│   with provider["hashicorp.com/edu/hashicups-pf"],
│   on main.tf line 9, in provider "hashicups":
│    9: provider "hashicups" {}
│ 
│ The provider cannot create the HashiCups API client as there is a missing or
│ empty value for the HashiCups API username. Set the username value in the
│ configuration or use the HASHICUPS_USERNAME environment variable. If either
│ is already set, ensure the value is not empty.
╵
╷
│ Error: Missing HashiCups API Password
│ 
│   with provider["hashicorp.com/edu/hashicups-pf"],
│   on main.tf line 9, in provider "hashicups":
│    9: provider "hashicups" {}
│ 
│ The provider cannot create the HashiCups API client as there is a missing or
│ empty value for the HashiCups API password. Set the password value in the
│ configuration or use the HASHICUPS_PASSWORD environment variable. If either
│ is already set, ensure the value is not empty.
╵
```
加上如下环境变量，再执行 Terraform plan
```
$ HASHICUPS_HOST=http://localhost:19090 \
  HASHICUPS_USERNAME=education \
  HASHICUPS_PASSWORD=test123 \
  terraform plan
```
这个时候，Terraform 已经可以正确读取到 Datasource 了
```
## ...
data.hashicups_coffees.example: Reading...
data.hashicups_coffees.example: Read complete after 0s

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and
found no differences, so no changes are needed.
```
HashiCups Server 的终端窗口显示有用户登陆了
```
api_1  | 2020-12-10T09:19:50.601Z [INFO]  Handle User | signin
```
通过手动在 Terraform 配置文件里添加 `host`, `username`, and `password` 参数来继续校验

创建一个 examples/coffees 目录
```
mkdir ../coffees && cd "$_"
```
创建一个 main.tf 文件，并设置 Terraform Provider 的配置参数
```
terraform {
  required_providers {
    hashicups = {
      source = "hashicorp.com/edu/hashicups-pf"
    }
  }
}

provider "hashicups" {
  host     = "http://localhost:19090"
  username = "education"
  password = "test123"
}

data "hashicups_coffees" "edu" {}

```
运行 Terraform Plan
```
$ terraform plan

##...
data.hashicups_coffees.edu: Reading...
data.hashicups_coffees.edu: Read complete after 0s
##...
```

## 移除临时的 Datasource
移除 `internal/provider/coffees_data_source.go` 文件
```
rm internal/provider/coffees_data_source.go
```
在 `internal/provider/provider.go` 文件中，修改 Datasource
```
// DataSources defines the data sources implemented in the provider.
func (p *hashicupsProvider) DataSources(_ context.Context) []func() datasource.DataSource {
    return nil
}
```
构建和安装 Provider
```
$ go install .
```
