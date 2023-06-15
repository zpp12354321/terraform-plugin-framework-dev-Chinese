# 实现 Logging
在本教程中，您将在 Provider 中记录日志消息并从日志输出中过滤特殊值。
然后您将在执行 Terraform 时查看这些日志信息。为此，您将：
1. 添加日志消息
2. 添加结构化日志字段
3. 添加日志过滤
4. 在运行命令时查看所有 Terraform 日志输出
5. 在运行命令时将 Terraform 日志输出保存到文件
6. 查看特定的 Terraform 日志输出

## 前置
* Go 1.19+
* Terraform v1.0.3+
* Docker and Docker Compose

## 实现日志消息（log messages）
Providers 通过 `tflog` 包下的 `github.com/hashicorp/terraform-plugin-log` 模块支持日志功能。
这个包实现了结构化的日志记录和过滤功能。

打开 `internal/provider/provider.go` 文件，更新 `Configure` 方法
```go
func (p *hashicupsProvider) Configure(ctx context.Context, req provider.ConfigureRequest, resp *provider.ConfigureResponse) {
    tflog.Info(ctx, "Configuring HashiCups client")

    // Retrieve provider data from configuration
    var config hashicupsProviderModel
    /* ... */
```
更新 `import`
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
    "github.com/hashicorp/terraform-plugin-log/tflog"
)
```

## 实现结构化日志字段（structured log fields）
`tflog` 包支持向日志里添加额外的键值对以实现链路追踪。可以使用 `tflog.SetField()` 方法为 Provider 的请求添加键值对，也可以将其作为记录日志时的最后一个参数。

打开 `internal/provider/provider.go` 文件

在创建客户端前 `hashicups.NewClient` 添加以下日志信息
```go
    /* ... */
    if resp.Diagnostics.HasError() {
        return
    }

    ctx = tflog.SetField(ctx, "hashicups_host", host)
    ctx = tflog.SetField(ctx, "hashicups_username", username)
    ctx = tflog.SetField(ctx, "hashicups_password", password)

    tflog.Debug(ctx, "Creating HashiCups client")

    // Create a new HashiCups client using the configuration values
    client, err := hashicups.NewClient(&host, &username, &password)
    /* ... */

```
在 Configure 方法的最后添加如下日志
```go
    /* ... */
    // Make the HashiCups client available during DataSource and Resource
    // type Configure methods.
    resp.DataSourceData = client
    resp.ResourceData = client

    tflog.Info(ctx, "Configured HashiCups client", map[string]any{"success": true})
}
```

## 实现日志过滤（log filtering）
实现一个过滤器，在执行 `tflog.Debug(ctx, "Creating HashiCups client")` 方法前屏蔽掉 password 信息
```go
    /* ... */
    ctx = tflog.SetField(ctx, "hashicups_host", host)
    ctx = tflog.SetField(ctx, "hashicups_username", username)
    ctx = tflog.SetField(ctx, "hashicups_password", password)
    ctx = tflog.MaskFieldValuesWithFieldKeys(ctx, "hashicups_password")

    tflog.Debug(ctx, "Creating HashiCups client")
    /* ... */
```
构建和安装
```
$ go install .
```

## 查看日志输出
Terraform 的日志输出由环境变量控制，例如 TF_LOG 或以 TF_LOG_ 为前缀的环境变量

进入 `examples/coffees` 目录
```
$ cd examples/coffees
```
执行 Terraform plan 命令，查看日志输出
```
$ TF_LOG=TRACE terraform plan
##...
2022-09-19T09:33:34.487-0500 [INFO]  provider.terraform-provider-hashicups-pf: Configuring HashiCups client: tf_provider_addr=hashicorp.com/edu/hashicups-pf tf_req_id=UUID tf_rpc=ConfigureProvider @caller=PATH @module=hashicups_pf timestamp=2022-09-19T09:33:34.487-0500
2022-09-19T09:33:34.487-0500 [DEBUG] provider.terraform-provider-hashicups-pf: Creating HashiCups client: @module=hashicups_pf hashicups_password=*** tf_req_id=UUID @caller=PATH hashicups_host=http://localhost:19090 hashicups_username=education tf_provider_addr=hashicorp.com/edu/hashicups-pf tf_rpc=ConfigureProvider timestamp=2022-09-19T09:33:34.487-0500
2022-09-19T09:33:34.517-0500 [INFO]  provider.terraform-provider-hashicups-pf: Configured HashiCups client: tf_rpc=ConfigureProvider hashicups_password=*** hashicups_username=education success=true tf_provider_addr=hashicorp.com/edu/hashicups-pf tf_req_id=UUID @caller=PATH @module=hashicups_pf hashicups_host=http://localhost:19090 timestamp=2022-09-19T09:33:34.517-0500
##...
```

## 保存日志
运行 Terraform Plan 时设置 TF_LOG 和 TF_LOG_PATH 环境变量
```
$ TF_LOG=TRACE TF_LOG_PATH=trace.txt terraform plan
```
打开 `examples/coffees/trace.txt` 文件并验证日志信息
```
##...
2022-09-30T16:23:38.515-0500 [DEBUG] provider.terraform-provider-hashicups-pf: Calling provider defined Provider Configure: tf_provider_addr=hashicorp.com/edu/hashicups-pf tf_req_id=12541d93-279d-1ec0-eac1-d2d2fcfd4030 tf_rpc=ConfigureProvider @caller=/Users/YOU/go/pkg/mod/github.com/hashicorp/terraform-plugin-framework@v0.13.0/internal/fwserver/server_configureprovider.go:12 @module=sdk.framework timestamp=2022-09-30T16:23:38.515-0500
2022-09-30T16:23:38.515-0500 [INFO]  provider.terraform-provider-hashicups-pf: Configuring HashiCups client: tf_req_id=12541d93-279d-1ec0-eac1-d2d2fcfd4030 tf_rpc=ConfigureProvider @caller=/Users/YOU/code/terraform-provider-hashicups-pf/hashicups/provider.go:66 @module=hashicups_pf tf_provider_addr=hashicorp.com/edu/hashicups-pf timestamp=2022-09-30T16:23:38.515-0500
2022-09-30T16:23:38.515-0500 [DEBUG] provider.terraform-provider-hashicups-pf: Creating HashiCups client: hashicups_password=*** tf_req_id=12541d93-279d-1ec0-eac1-d2d2fcfd4030 tf_provider_addr=hashicorp.com/edu/hashicups-pf tf_rpc=ConfigureProvider @caller=/Users/YOU/code/terraform-provider-hashicups-pf/hashicups/provider.go:171 @module=hashicups_pf hashicups_host=http://localhost:19090 hashicups_username=education timestamp=2022-09-30T16:23:38.515-0500
2022-09-30T16:23:38.524-0500 [INFO]  provider.terraform-provider-hashicups-pf: Configured HashiCups client: @module=hashicups_pf hashicups_password=*** hashicups_username=education tf_req_id=12541d93-279d-1ec0-eac1-d2d2fcfd4030 tf_rpc=ConfigureProvider @caller=/Users/YOU/code/terraform-provider-hashicups-pf/hashicups/provider.go:190 hashicups_host=http://localhost:19090 success=true tf_provider_addr=hashicorp.com/edu/hashicups-pf timestamp=2022-09-30T16:23:38.524-0500
2022-09-30T16:23:38.524-0500 [DEBUG] provider.terraform-provider-hashicups-pf: Called provider defined Provider Configure: @module=sdk.framework tf_provider_addr=hashicorp.com/edu/hashicups-pf tf_req_id=12541d93-279d-1ec0-eac1-d2d2fcfd4030 tf_rpc=ConfigureProvider @caller=/Users/YOU/go/pkg/mod/github.com/hashicorp/terraform-plugin-framework@v0.13.0/internal/fwserver/server_configureprovider.go:20 timestamp=2022-09-30T16:23:38.524-0500
##...
```
移除 `examples/coffees/trace.txt` 文件
```
$ rm trace.txt
```

## 查看特定日志输出
前面的示例使用了 `TRACE` 日志记录级别，`TRACE` 是最详细级别的日志记录，可能包含大量信息，这些信息仅在您深入了解 Terraform 或 Terraform Plugin Framework 的某些内部组件时才有意义。
您可以改为将日志记录级别降低为 `DEBUG`、`INFO`、`WARN` 或 `ERROR`。

在 `TF_LOG` 环境变量设置为 `INFO` 的情况下运行 Terraform Plan
```
$ TF_LOG=INFO terraform plan
2022-09-30T16:27:38.446-0500 [INFO]  Terraform version: 1.3.0
2022-09-30T16:27:38.447-0500 [INFO]  Go runtime version: go1.19.1
2022-09-30T16:27:38.447-0500 [INFO]  CLI args: []string{"terraform", "plan"}
#...
```
您可以为某些组件启用日志输出，例如来支持 Provider 日志而不是 Terraform CLI 日志。

在 `TF_LOG_PROVIDER` 环境变量设置为 `INFO` 的情况下运行 Terraform Plan
```
$ TF_LOG_PROVIDER=INFO terraform plan
##...
2022-12-14T10:39:33.247-0600 [INFO]  provider.terraform-provider-hashicups-pf: Configuring HashiCups client: @caller=/Users/YOU/code/terraform-provider-hashicups-pf/hashicups/provider.go:61 @module=hashicups_pf tf_provider_addr=hashicorp.com/edu/hashicups-pf tf_req_id=45969718-ed46-42b3-2cb9-847635d5aebb tf_rpc=ConfigureProvider timestamp=2022-12-14T10:39:33.247-0600
2022-12-14T10:39:33.255-0600 [INFO]  provider.terraform-provider-hashicups-pf: Configured HashiCups client: success=true hashicups_host=http://localhost:19090 hashicups_username=education tf_provider_addr=hashicorp.com/edu/hashicups-pf tf_req_id=45969718-ed46-42b3-2cb9-847635d5aebb tf_rpc=ConfigureProvider @caller=/Users/YOU/code/terraform-provider-hashicups-pf/hashicups/provider.go:184 @module=hashicups_pf timestamp=2022-12-14T10:39:33.255-0600
data.hashicups_coffees.edu: Reading...
data.hashicups_coffees.edu: Read complete after 0s
##...
```