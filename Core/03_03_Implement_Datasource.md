# 实现 Datasource
在本教程中，将实现一个从 HashiCups API 读取咖啡列表并将其保存在 Terraform 状态中的 Datasource。
为此，您将： 

1. 定义 Datasource 的类型  
2. 将 Datasource 添加到 Provider
3. 在 Datasource 中实现 HashiCups 客户端  
从 Provider 中获取 HashiCups 客户端，并在 Datasource 中使用
4. 定义 Datasource Schema  
5. 定义 Datasource 数据模型  
6. 定义 Datasource 读取逻辑  
通过 HashiCups 客户端调用 API 并使用数据更新到 Terraform State
7. 验证 Datasource
确保 Datasource 的行为符合预期

## 前置
* Go 1.19+
* Terraform v1.0.3+
* Docker and Docker Compose

## 定义 Datasource 的类型
Datasource 需要实现 `datasource.DataSource` 接口，才能将其添加到 Provider

该接口定义了以下方法：
1. Metadata Method。定义了 Datasource 类型名称，可以通过该名称在 Terraform 配置中引用 Datasource
2. Schema Method。定义了 Datasource 中存在哪些字段
3. Read Method。实现读逻辑，并更新 Terraform state

创建一个 `internal/provider/coffees_data_source.go` 文件
```go
package provider

import (
    "context"

    "github.com/hashicorp/terraform-plugin-framework/datasource"
    "github.com/hashicorp/terraform-plugin-framework/datasource/schema"
)

// Ensure the implementation satisfies the expected interfaces.
var (
    _ datasource.DataSource = &coffeesDataSource{}
)

// NewCoffeesDataSource is a helper function to simplify the provider implementation.
func NewCoffeesDataSource() datasource.DataSource {
    return &coffeesDataSource{}
}

// coffeesDataSource is the data source implementation.
type coffeesDataSource struct{}

// Metadata returns the data source type name.
func (d *coffeesDataSource) Metadata(_ context.Context, req datasource.MetadataRequest, resp *datasource.MetadataResponse) {
    resp.TypeName = req.ProviderTypeName + "_coffees"
}

// Schema defines the schema for the data source.
func (d *coffeesDataSource) Schema(_ context.Context, _ datasource.SchemaRequest, resp *datasource.SchemaResponse) {
    resp.Schema = schema.Schema{}
}

// Read refreshes the Terraform state with the latest data.
func (d *coffeesDataSource) Read(ctx context.Context, req datasource.ReadRequest, resp *datasource.ReadResponse) {
}

```

## 将 Datasource 添加到 Provider
更新 `internal/provider/provider.go` 文件
```go
// DataSources defines the data sources implemented in the provider.
func (p *hashicupsProvider) DataSources(_ context.Context) []func() datasource.DataSource {
    return []func() datasource.DataSource {
        NewCoffeesDataSource,
    }
}
```

## 在 Datasource 中实现 HashiCups 客户端
Datasource 可以选择实现 Configure 方法来从 Provider 获取已经配置好的客户端并持有

编辑 `internal/provider/coffees_data_source.go` 文件

将 `coffeesDataSource` 替换为以下内容，允许持有对 HashiCups 客户端的引用
```go
// coffeesDataSource is the data source implementation.
type coffeesDataSource struct {
    client *hashicups.Client
}
```
替换 `import` 的内容
```go
import (
       "context"

       "github.com/hashicorp-demoapp/hashicups-client-go"
       "github.com/hashicorp/terraform-plugin-framework/datasource"
       "github.com/hashicorp/terraform-plugin-framework/datasource/schema"
)
```
确保 Datasource 实现 `DataSourceWithConfigure` 接口
```go
// Ensure the implementation satisfies the expected interfaces.
var (
    _ datasource.DataSource              = &coffeesDataSource{}
    _ datasource.DataSourceWithConfigure = &coffeesDataSource{}
)
```
实现 Configure 方法，从 Provider 获取 HashiCups 客户端
```go
// Configure adds the provider configured client to the data source.
func (d *coffeesDataSource) Configure(_ context.Context, req datasource.ConfigureRequest, resp *datasource.ConfigureResponse) {
    if req.ProviderData == nil {
        return
    }

    client, ok := req.ProviderData.(*hashicups.Client)
    if !ok {
        resp.Diagnostics.AddError(
            "Unexpected Data Source Configure Type",
            fmt.Sprintf("Expected *hashicups.Client, got: %T. Please report this issue to the provider developers.", req.ProviderData),
        )

        return
    }

    d.client = client
}
```

## 实现 Datasource Schema
Schema 方法定义了 Datasource 块的配置的名称与属性

Coffees Datasource 需要保存 Coffee 列表到 Terraform State 中

使用如下代码替换 Schema 方法
```go
// Schema defines the schema for the data source.
func (d *coffeesDataSource) Schema(_ context.Context, _ datasource.SchemaRequest, resp *datasource.SchemaResponse) {
    resp.Schema = schema.Schema{
        Attributes: map[string]schema.Attribute{
            "coffees": schema.ListNestedAttribute{
                Computed: true,
                NestedObject: schema.NestedAttributeObject{
                    Attributes: map[string]schema.Attribute{
                        "id": schema.Int64Attribute{
                            Computed: true,
                        },
                        "name": schema.StringAttribute{
                            Computed: true,
                        },
                        "teaser": schema.StringAttribute{
                            Computed: true,
                        },
                        "description": schema.StringAttribute{
                            Computed: true,
                        },
                        "price": schema.Float64Attribute{
                            Computed: true,
                        },
                        "image": schema.StringAttribute{
                            Computed: true,
                        },
                        "ingredients": schema.ListNestedAttribute{
                            Computed: true,
                            NestedObject: schema.NestedAttributeObject{
                                Attributes: map[string]schema.Attribute{
                                    "id": schema.Int64Attribute{
                                        Computed: true,
                                    },
                                },
                            },
                        },
                    },
                },
            },
        },
    }
}

```

## 定义 Datasource 数据模型
```go
// coffeesDataSourceModel maps the data source schema data.
type coffeesDataSourceModel struct {
    Coffees []coffeesModel `tfsdk:"coffees"`
}

// coffeesModel maps coffees schema data.
type coffeesModel struct {
    ID          types.Int64               `tfsdk:"id"`
    Name        types.String              `tfsdk:"name"`
    Teaser      types.String              `tfsdk:"teaser"`
    Description types.String              `tfsdk:"description"`
    Price       types.Float64             `tfsdk:"price"`
    Image       types.String              `tfsdk:"image"`
    Ingredients []coffeesIngredientsModel `tfsdk:"ingredients"`
}

// coffeesIngredientsModel maps coffee ingredients data
type coffeesIngredientsModel struct {
    ID types.Int64 `tfsdk:"id"`
}
```
更新 import
```go
import (
    "context"
    "fmt"

    "github.com/hashicorp-demoapp/hashicups-client-go"
    "github.com/hashicorp/terraform-plugin-framework/datasource"
    "github.com/hashicorp/terraform-plugin-framework/datasource/schema"
    "github.com/hashicorp/terraform-plugin-framework/types"
)
```

## 实现读取方法
Datasource 调用 Read 方法来更新基于 Schema 的 Terraform State 状态信息。
本次我们适配的 hashicups_coffees Datasource 将使用 HashiCups client 调用 HashiCups API 来获取 Coffee 列表并保存到 Terraform State 中。

读取的方法需要遵循以下步骤：
1. 调用客户端的 `GetCoffees` 方法获取所有 Coffee 列表
2. 将 API 的响应，映射到 Terraform Schema 上
3. 更新 Terraform State 的状态
```go
// Read refreshes the Terraform state with the latest data.
func (d *coffeesDataSource) Read(ctx context.Context, req datasource.ReadRequest, resp *datasource.ReadResponse) {
    var state coffeesDataSourceModel

    coffees, err := d.client.GetCoffees()
    if err != nil {
        resp.Diagnostics.AddError(
            "Unable to Read HashiCups Coffees",
            err.Error(),
        )
        return
    }

    // Map response body to model
    for _, coffee := range coffees {
        coffeeState := coffeesModel{
            ID:          types.Int64Value(int64(coffee.ID)),
            Name:        types.StringValue(coffee.Name),
            Teaser:      types.StringValue(coffee.Teaser),
            Description: types.StringValue(coffee.Description),
            Price:       types.Float64Value(coffee.Price),
            Image:       types.StringValue(coffee.Image),
        }

        for _, ingredient := range coffee.Ingredient {
            coffeeState.Ingredients = append(coffeeState.Ingredients, coffeesIngredientsModel{
                ID: types.Int64Value(int64(ingredient.ID)),
            })
        }

        state.Coffees = append(state.Coffees, coffeeState)
    }

    // Set state
    diags := resp.State.Set(ctx, &state)
    resp.Diagnostics.Append(diags...)
    if resp.Diagnostics.HasError() {
        return
    }
}

```
构建和安装
```
$ go install .
```

## 验证 Datasource
```
cd examples/coffees
```
修改 main.tf 文件
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

output "edu_coffees" {
  value = data.hashicups_coffees.edu
}
```
执行 Terraform plan
```
$ terraform plan
##...
data.hashicups_coffees.edu: Reading...
data.hashicups_coffees.edu: Read complete after 0s

Changes to Outputs:
  + edu_coffees = {
      + coffees = [
          + {
              + description = ""
              + id          = 1
              + image       = "/hashicorp.png"
              + ingredients = [
                  + {
                      + id = 6
                    },
                ]
              + name        = "HCP Aeropress"
              + price       = 200
              + teaser      = "Automation in a cup"
            },
##...
You can apply this plan to save these new output values to the Terraform state,
without changing any real infrastructure.

───────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't
guarantee to take exactly these actions if you run "terraform apply" now.
```