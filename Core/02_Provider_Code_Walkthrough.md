# 代码初步走读
**Terraform 通过 Terraform Provider 与云提供商、SaaS 服务提供商等第三方通信。而 Terraform 与 Terraform Provider 之间的通信则使用 gRPC。Terraform 作为 gRPC 客户端运行，Provider 作为 gRPC 服务器运行**

Terraform 通过 Provider 定义的 Resource 来管理基础设施、通过定义的 Datasource 来读取数据。
使用者编写 HCL 配置来定义资源后， 由Terraform 将此配置传达给 Provider，再由 Provider 创建基础设施

如下的示例将会展示 Provider 各个组件之间的关系。 通常来说，Resource 和 Datasource 通过 API 与云提供商进行通信，
但本示例使用静态数据进行演示

## Provider 核心组件
Terraform Provider 至少需要以下组件：
* Provider Server
* Provider
* Resource or Datasource

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
`provider.New()` 返回一个函数，该函数的返回值需要实现 `provider.Provider` 接口。
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

## Provider
Provider 封装了云厂商、SaaS 厂商或与其他 API 交互的 Resource 和 Datasource

在这个例子中，Provider 封装了一个 Resource 和一个 Datasource

`New()` 返回一个函数，该函数的返回需要实现 `provider.Provider` 接口

`provider.Provider` 接口定义了如下方法：
* `Schema`：此函数返回一个 `schema.Schema` 结构，用以指定 Terraform 配置块（Terraform configuration blocks）
* `Configure`：该函数允许配置 provider-level 的数据或客户端。 这些配置值可能由使用者通过 schema、环境变量或其他方式所定义
* `Resource`：返回所有实现的 Resource
* `Datasource`：返回所有实现的 Datasource
* `Metadata`：`Metadata` 函数返回 provider 的元数据，例如 `TypeName` 和 `Version`。`TypeName` 用于命名资源和数据源的前缀
```go
package provider

import (
	"context"

	"github.com/hashicorp/terraform-plugin-framework/datasource"
	"github.com/hashicorp/terraform-plugin-framework/provider"
	"github.com/hashicorp/terraform-plugin-framework/resource"
)

var _ provider.Provider = (*exampleProvider)(nil)

type exampleProvider struct{}

func New() func() provider.Provider {
	return func() provider.Provider {
		return &exampleProvider{}
	}
}

func (p *exampleProvider) Configure(ctx context.Context, req provider.ConfigureRequest, resp *provider.ConfigureResponse) {
}

func (p *exampleProvider) Metadata(ctx context.Context, req provider.MetadataRequest, resp *provider.MetadataResponse) {
	resp.TypeName = "example"
}

func (p *exampleProvider) DataSources(ctx context.Context) []func() datasource.DataSource {
	return []func() datasource.DataSource{
		NewDataSource,
	}
}

func (p *exampleProvider) Resources(ctx context.Context) []func() resource.Resource {
	return []func() resource.Resource{
		NewResource,
	}
}

func (p *exampleProvider) Schema(ctx context.Context, req provider.SchemaRequest, resp *provider.SchemaResponse) {
}
```

## Resource
Resource 通常用于管理基础设施对象，例如虚拟网络和计算实例。

`exampleResource` 结构体实现了 `resource.Resource` 接口。 该接口定义了以下函数：
* `Metadata`：返回资源的全名 (TypeName)。 全名在 Terraform 配置中用作 `resource <full name> <alias>`
* `Schema`：定义 Resource 资源的 Schema。它定义了 Resource 配置块具有哪些字段，并为 Terraform 提供这些字段的元数据信息。
例如，定义一个字段是否是必需的
* `Create`：创建此类型的新资源
* `Read`：读取资源值以更新状态
* `Update`：更新资源和状态
* `Delete`：删除资源
```go
package provider

import (
	"context"

	"github.com/hashicorp/terraform-plugin-framework/resource"
	"github.com/hashicorp/terraform-plugin-framework/resource/schema"
	"github.com/hashicorp/terraform-plugin-framework/resource/schema/planmodifier"
	"github.com/hashicorp/terraform-plugin-framework/resource/schema/stringplanmodifier"
	"github.com/hashicorp/terraform-plugin-framework/types"
	"github.com/hashicorp/terraform-plugin-log/tflog"
)

var _ resource.Resource = (*exampleResource)(nil)

type exampleResource struct {
    provider exampleProvider
}

func NewResource() resource.Resource {
    return &exampleResource{}
}

func (e *exampleResource) Metadata(_ context.Context, req resource.MetadataRequest, resp *resource.MetadataResponse) {
    resp.TypeName = req.ProviderTypeName + "_resource"
}

func (e *exampleResource) Schema(ctx context.Context, req resource.SchemaRequest, resp *resource.SchemaResponse) {
    resp.Schema = schema.Schema{
        Attributes: map[string]schema.Attribute{
            "configurable_attribute": schema.StringAttribute{
                Optional:            true,
            },
            "id": schema.StringAttribute{
                Computed:            true,
                PlanModifiers: []planmodifier.String{
                    stringplanmodifier.UseStateForUnknown(),
                },
            },
        },
    }
}

type exampleResourceData struct {
    ConfigurableAttribute types.String `tfsdk:"configurable_attribute"`
    Id                    types.String `tfsdk:"id"`
}

func (e *exampleResource) Create(ctx context.Context, req resource.CreateRequest, resp *resource.CreateResponse) {
    var data exampleResourceData

    diags := req.Config.Get(ctx, &data)
    resp.Diagnostics.Append(diags...)

    if resp.Diagnostics.HasError() {
        return
    }

    // Create resource using 3rd party API.

    data.Id = types.StringValue("example-id")

    tflog.Trace(ctx, "created a resource")

    diags = resp.State.Set(ctx, &data)
    resp.Diagnostics.Append(diags...)
}

func (e *exampleResource) Read(ctx context.Context, req resource.ReadRequest, resp *resource.ReadResponse) {
    var data exampleResourceData

    diags := req.State.Get(ctx, &data)
    resp.Diagnostics.Append(diags...)

    if resp.Diagnostics.HasError() {
        return
    }

    // Read resource using 3rd party API.

    diags = resp.State.Set(ctx, &data)
    resp.Diagnostics.Append(diags...)
}

func (e *exampleResource) Update(ctx context.Context, req resource.UpdateRequest, resp *resource.UpdateResponse) {
    var data exampleResourceData

    diags := req.Plan.Get(ctx, &data)
    resp.Diagnostics.Append(diags...)

    if resp.Diagnostics.HasError() {
        return
    }

    // Update resource using 3rd party API.

    diags = resp.State.Set(ctx, &data)
    resp.Diagnostics.Append(diags...)
}

func (e *exampleResource) Delete(ctx context.Context, req resource.DeleteRequest, resp *resource.DeleteResponse) {
    var data exampleResourceData

    diags := req.State.Get(ctx, &data)
    resp.Diagnostics.Append(diags...)

    if resp.Diagnostics.HasError() {
        return
    }

    // Delete resource using 3rd party API.
}
```

## Datasource
Datasource 通常用于提供基础结构对象的只读视图

`exampleDataSource` 结构实现了 `datasource.DataSource` 接口。 该接口定义了以下函数：
* `Metadata`：此函数返回 Datasource 的全名 (TypeName)。 全名在 Terraform 配置中用作 `data <full name> <alias>`
* `Schema`：定义了 Datasource Terraform 配置块具有哪些字段。例如，定义一个字段是否可选
* `Read`：允许提供 provider 读取 Datasource 的值以更新状态
```go
package provider

import (
	"context"

	"github.com/hashicorp/terraform-plugin-framework/datasource"
	"github.com/hashicorp/terraform-plugin-framework/datasource/schema"
	"github.com/hashicorp/terraform-plugin-framework/types"
	"github.com/hashicorp/terraform-plugin-log/tflog"
)

var _ datasource.DataSource = (*exampleDataSource)(nil)

type exampleDataSource struct {
	provider exampleProvider
}

func NewDataSource() datasource.DataSource {
	return &exampleDataSource{}
}

func (e *exampleDataSource) Metadata(ctx context.Context, req datasource.MetadataRequest, resp *datasource.MetadataResponse) {
	resp.TypeName = req.ProviderTypeName + "_datasource"
}

func (e *exampleDataSource) Schema(ctx context.Context, req datasource.SchemaRequest, resp *datasource.SchemaResponse) {
	resp.Schema = schema.Schema{
		Attributes: map[string]schema.Attribute{
			"configurable_attribute": schema.StringAttribute{
				MarkdownDescription: "Example configurable attribute",
				Optional:            true,
			},
			"id": schema.StringAttribute{
				MarkdownDescription: "Example identifier",
				Computed:            true,
			},
		},
	}
}

type exampleDataSourceData struct {
	ConfigurableAttribute types.String `tfsdk:"configurable_attribute"`
	Id                    types.String `tfsdk:"id"`
}

func (e *exampleDataSource) Read(ctx context.Context, req datasource.ReadRequest, resp *datasource.ReadResponse) {
	var data exampleDataSourceData

	diags := req.Config.Get(ctx, &data)
	resp.Diagnostics.Append(diags...)

	if resp.Diagnostics.HasError() {
		return
	}

	// Interact with 3rd party API to read data source.

	data.Id = types.StringValue("example-id")

	tflog.Trace(ctx, "read a data source")

	diags = resp.State.Set(ctx, &data)
	resp.Diagnostics.Append(diags...)
}
```

## Terraform 使用配置
### Resource 配置
```hcl
resource "example_resource" "example" {
  configurable_attribute = "some-value"
}
```
`configurable_attribute` 在 Resource Schema 中定义为字符串类型属性
### Datasource 配置
```hcl
data "example_datasource" "example" {
  configurable_attribute = "some-value"
}
```
`configurable_attribute` 在 Datasource Schema 中定义为字符串类型属性