# 实现资源创建和读取
在本教程中，您将向 Provider 的 `order` resource 添加创建和读取功能，该 Provider 将会与名为 Hashicups 的虚构咖啡店应用程序的 API 进行交互。
为此，您将：
1. 定义 Resource 类型
2. 添加 Resource 到 Provider
3. 在 Resource 中实现 HashiCups 客户端
4. 定义 Resource Schema
5. 定义 Resource 数据模型
6. 实现 Resource 创建逻辑
7. 实现 Resource 读取逻辑
8. 验证 Resource 的行为

## 前置
* Go 1.19+
* Terraform v1.0.3+
* Docker and Docker Compose

## 定义 Resource 类型
Resource 需要实现 `resource.Resource` 接口，该接口需要实现
1. Metadata Method：定义 Resource 的类型名称
2. Schema Method：定义 Resource Schema，用于 Resource 块的配置
3. Create Method：定义资源创建逻辑，并更新 Terraform State 状态
4. Read Method：定义资源读取逻辑
5. Update Method：定义资源更新逻辑，并更新 Terraform State 状态
6. Delete Method：定义资源删除逻辑

创建 `internal/provider/order_resource.go` 文件
```go
package provider

import (
    "context"

    "github.com/hashicorp/terraform-plugin-framework/resource"
    "github.com/hashicorp/terraform-plugin-framework/resource/schema"
)

// Ensure the implementation satisfies the expected interfaces.
var (
    _ resource.Resource = &orderResource{}
)

// NewOrderResource is a helper function to simplify the provider implementation.
func NewOrderResource() resource.Resource {
    return &orderResource{}
}

// orderResource is the resource implementation.
type orderResource struct{}

// Metadata returns the resource type name.
func (r *orderResource) Metadata(_ context.Context, req resource.MetadataRequest, resp *resource.MetadataResponse) {
    resp.TypeName = req.ProviderTypeName + "_order"
}

// Schema defines the schema for the resource.
func (r *orderResource) Schema(_ context.Context, _ resource.SchemaRequest, resp *resource.SchemaResponse) {
    resp.Schema = schema.Schema{}
}

// Create creates the resource and sets the initial Terraform state.
func (r *orderResource) Create(ctx context.Context, req resource.CreateRequest, resp *resource.CreateResponse) {
}

// Read refreshes the Terraform state with the latest data.
func (r *orderResource) Read(ctx context.Context, req resource.ReadRequest, resp *resource.ReadResponse) {
}

// Update updates the resource and sets the updated Terraform state on success.
func (r *orderResource) Update(ctx context.Context, req resource.UpdateRequest, resp *resource.UpdateResponse) {
}

// Delete deletes the resource and removes the Terraform state on success.
func (r *orderResource) Delete(ctx context.Context, req resource.DeleteRequest, resp *resource.DeleteResponse) {
}
```

## 添加 Resource 到 Provider
打开 `internal/provider/provider.go` 文件，并更新
```go
// Resources defines the resources implemented in the provider.
func (p *hashicupsProvider) Resources(_ context.Context) []func() resource.Resource {
    return []func() resource.Resource{
        NewOrderResource,
    }
}
```

## 在 Resource 中实现 HashiCups 客户端
打开 internal/provider/order_resource.go 文件，并更新代码
```go
// orderResource is the resource implementation.
type orderResource struct {
    client *hashicups.Client
}
```

更新 import
```go
import (
    "context"

    "github.com/hashicorp-demoapp/hashicups-client-go"
    "github.com/hashicorp/terraform-plugin-framework/resource"
    "github.com/hashicorp/terraform-plugin-framework/resource/schema"
)
```
Resource 资源需要实现 ResourceWithConfigure 接口
```go
// Ensure the implementation satisfies the expected interfaces.
var (
    _ resource.Resource              = &orderResource{}
    _ resource.ResourceWithConfigure = &orderResource{}
)
```
实现 Configure 方法
```go
// Configure adds the provider configured client to the resource.
func (r *orderResource) Configure(_ context.Context, req resource.ConfigureRequest, resp *resource.ConfigureResponse) {
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

    r.client = client
}
```

## 定义 Resource Schema
Resource 使用 Schema 方法来定义支持的状态属性名称和类型。
本次实现的 order resource 需要将具有各种属性的咖啡列表保存到 Terraform State 中。

注意 Required 和 Optional 为 true 的，才是 Schema 中支持输入的参数
Computed 为 true 的参数为计算出来的参数
```go
// Schema defines the schema for the resource.
func (r *orderResource) Schema(_ context.Context, _ resource.SchemaRequest, resp *resource.SchemaResponse) {
    resp.Schema = schema.Schema{
        Attributes: map[string]schema.Attribute{
            "id": schema.StringAttribute{
                Computed: true,
            },
            "last_updated": schema.StringAttribute{
                Computed: true,
            },
			// 咖啡列表
            "items": schema.ListNestedAttribute{
                Required: true,
                NestedObject: schema.NestedAttributeObject{
                    Attributes: map[string]schema.Attribute{
                        "quantity": schema.Int64Attribute{
                            Required: true,  // 可输入的参数
                        },
                        "coffee": schema.SingleNestedAttribute{
                            Required: true,
                            Attributes: map[string]schema.Attribute{
                                "id": schema.Int64Attribute{
                                    Required: true,  // 可输入的参数
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
                            },
                        },
                    },
                },
            },
        },
    }
}
```

## 实现 Resource 资源模型
使用 go 结构体描述 TF 文件的定义
```go
// orderResourceModel maps the resource schema data.
type orderResourceModel struct {
    ID          types.String     `tfsdk:"id"`
    Items       []orderItemModel `tfsdk:"items"`
    LastUpdated types.String     `tfsdk:"last_updated"`
}

// orderItemModel maps order item data.
type orderItemModel struct {
    Coffee   orderItemCoffeeModel `tfsdk:"coffee"`
    Quantity types.Int64          `tfsdk:"quantity"`
}

// orderItemCoffeeModel maps coffee order item data.
type orderItemCoffeeModel struct {
    ID          types.Int64   `tfsdk:"id"`
    Name        types.String  `tfsdk:"name"`
    Teaser      types.String  `tfsdk:"teaser"`
    Description types.String  `tfsdk:"description"`
    Price       types.Float64 `tfsdk:"price"`
    Image       types.String  `tfsdk:"image"`
}
```
## 实现 Resource Create
Create 方法的实现需要遵循以下步骤
1. 检查 API Client 是否正确配置。如果没有配置，抛出错误
2. 从 Plan 中获取值，并将其转换为 `orderResourceModel`
3. 根据获取的值，将其转换为 API 的请求体 `hashicups.OrderItem`
4. 创建一个新的 order
5. 将 API 的响应转换为 Resource Schema
6. 更新 Terraform State 的状态
```go
// Create a new resource.
func (r *orderResource) Create(ctx context.Context, req resource.CreateRequest, resp *resource.CreateResponse) {
    // Retrieve values from plan
    var plan orderResourceModel
    diags := req.Plan.Get(ctx, &plan)
    resp.Diagnostics.Append(diags...)
    if resp.Diagnostics.HasError() {
        return
    }

    // Generate API request body from plan
    var items []hashicups.OrderItem
    for _, item := range plan.Items {
        items = append(items, hashicups.OrderItem{
            Coffee: hashicups.Coffee{
                ID: int(item.Coffee.ID.ValueInt64()),
            },
            Quantity: int(item.Quantity.ValueInt64()),
        })
    }

    // Create new order
    order, err := r.client.CreateOrder(items)
    if err != nil {
        resp.Diagnostics.AddError(
            "Error creating order",
            "Could not create order, unexpected error: "+err.Error(),
        )
        return
    }

    // Map response body to schema and populate Computed attribute values
    plan.ID = types.StringValue(strconv.Itoa(order.ID))
    for orderItemIndex, orderItem := range order.Items {
        plan.Items[orderItemIndex] = orderItemModel{
            Coffee: orderItemCoffeeModel{
                ID:          types.Int64Value(int64(orderItem.Coffee.ID)),
                Name:        types.StringValue(orderItem.Coffee.Name),
                Teaser:      types.StringValue(orderItem.Coffee.Teaser),
                Description: types.StringValue(orderItem.Coffee.Description),
                Price:       types.Float64Value(orderItem.Coffee.Price),
                Image:       types.StringValue(orderItem.Coffee.Image),
            },
            Quantity: types.Int64Value(int64(orderItem.Quantity)),
        }
    }
    plan.LastUpdated = types.StringValue(time.Now().Format(time.RFC850))

    // Set state to fully populated data
    diags = resp.State.Set(ctx, plan)
    resp.Diagnostics.Append(diags...)
    if resp.Diagnostics.HasError() {
        return
    }
}
```
更新 import 方法
```go
import (
    "context"
    "fmt"
    "strconv"
    "time"

    "github.com/hashicorp-demoapp/hashicups-client-go"
    "github.com/hashicorp/terraform-plugin-framework/resource"
    "github.com/hashicorp/terraform-plugin-framework/resource/schema"
    "github.com/hashicorp/terraform-plugin-framework/types"
)
```

## 实现读方法
读方法用来获取 Resource 的信息并更新 Terraform State 的状态。
Terraform Provider 在每次 Plan 之前调用 `Read` 方法，用来比较资源的当前状态与配置之间的差异。

(说白了，就是 Terraform Plan 显示出来的差异，就是读取出来的值，与你在 HCL 文件中定义值的不同之处)

(读取之后，会将数据写入到 tf.state 文件里)

1. 获取当前 Terraform State 里的数据
2. 通过该数据获取资源的 ID，在这里也就是 Order ID
3. 通过 API Client 获取 Order 的详情
4. 将 API 的响应转换为 Resource Schema
5. 更新 Terraform State 的状态
```go
// Read resource information.
func (r *orderResource) Read(ctx context.Context, req resource.ReadRequest, resp *resource.ReadResponse) {
    // Get current state
    var state orderResourceModel
    diags := req.State.Get(ctx, &state)
    resp.Diagnostics.Append(diags...)
    if resp.Diagnostics.HasError() {
        return
    }

    // Get refreshed order value from HashiCups
    order, err := r.client.GetOrder(state.ID.ValueString())
    if err != nil {
        resp.Diagnostics.AddError(
            "Error Reading HashiCups Order",
            "Could not read HashiCups order ID "+state.ID.ValueString()+": "+err.Error(),
        )
        return
    }

    // Overwrite items with refreshed state
    state.Items = []orderItemModel{}
    for _, item := range order.Items {
        state.Items = append(state.Items, orderItemModel{
            Coffee: orderItemCoffeeModel{
                ID:          types.Int64Value(int64(item.Coffee.ID)),
                Name:        types.StringValue(item.Coffee.Name),
                Teaser:      types.StringValue(item.Coffee.Teaser),
                Description: types.StringValue(item.Coffee.Description),
                Price:       types.Float64Value(item.Coffee.Price),
                Image:       types.StringValue(item.Coffee.Image),
            },
            Quantity: types.Int64Value(int64(item.Quantity)),
        })
    }

    // Set refreshed state
    diags = resp.State.Set(ctx, &state)
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

## 验证 Resource
创建 `examples/order` 文件夹
```
$ mkdir examples/order && cd "$_"
```
创建 `main.tf` 文件
```hcl
terraform {
  required_providers {
    hashicups = {
      source  = "hashicorp.com/edu/hashicups-pf"
    }
  }
  required_version = ">= 1.1.0"
}

provider "hashicups" {
  username = "education"
  password = "test123"
  host     = "http://localhost:19090"
}

resource "hashicups_order" "edu" {
  items = [{
    coffee = {
      id = 3
    }
    quantity = 2
    }, {
    coffee = {
      id = 1
    }
    quantity = 2
    }
  ]
}

output "edu_order" {
  value = hashicups_order.edu
}
```
执行 terraform 命令
```
$ terraform apply

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

   hashicups_order.edu will be created
  + resource "hashicups_order" "edu" {
      + id           = (known after apply)
      # ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + edu_order = {
      + id           = (known after apply)
      + items        = [
          + {
#...

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

hashicups_order.edu: Creating...
hashicups_order.edu: Creation complete after 0s [id=21]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

edu_order = {
  "id" = "1"
  "items" = tolist([
    {
      "coffee" = {
        "description" = ""
        "id" = 3
        "image" = "/vault.png"
        "name" = "Vaulatte"
        "price" = 200
        "teaser" = "Nothing gives you a safe and secure feeling like a Vaulatte"
      }
      "quantity" = 2
    },
    {
      "coffee" = {
        "description" = ""
        "id" = 1
        "image" = "/hashicorp.png"
        "name" = "HCP Aeropress"
        "price" = 200
        "teaser" = "Automation in a cup"
      }
      "quantity" = 2
    },
  ])
  "last_updated" = "Thursday, 09-Feb-23 11:32:05 EST"
}
```
执行完成之后，Provider 将会在 Terraform state 文件中保存 Resource 的信息。可以通过以下命令查看详情
```
$ terraform state show hashicups_order.edu
# hashicups_order.edu:
resource "hashicups_order" "edu" {
    id           = "1"
    items        = [
       (2 unchanged elements hidden)
    ]
    last_updated = "Thursday, 22-Jul-21 03:26:51 PDT"
}
```

## 验证 Order 创建
可以通过终端，查看 HashiCups 服务端打印的日志
```
api_1  | 2021-07-22T10:26:31.179Z [INFO]  Handle User | signin
api_1  | 2021-07-22T10:26:51.179Z [INFO]  Handle User | signin
api_1  | 2021-07-22T10:26:51.195Z [INFO]  Handle Orders | CreateOrder
```
