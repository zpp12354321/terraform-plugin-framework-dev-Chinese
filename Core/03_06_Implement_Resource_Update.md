# 实现资源更新
在本教程中，您将为上一节课开发的 provider 的 order resource 资源添加更新功能。具体包括
1. 验证 schema 和数据模型  
验证 last_updated 属性。每当更新order resource 资源时，provider 都会将此属性更新为当前日期时间
2. 实现资源更新
3. 使用 plan modifier  
在资源更新时，保持 id 属性不变，来消除 Terraform plan 带来的差异性
4. 验证更新功能  
确保资源更新符合预期

## 前置
* Go 1.19+
* Terraform v1.0.3+
* Docker and Docker Compose

## 验证 schema 和数据模型
验证 `Schema` 方法包含 `last_updated` 属性
```go
func (r *orderResource) Schema(_ context.Context, _ resource.SchemaRequest, resp *resource.SchemaResponse) {
    resp.Schema = schema.Schema{
        Attributes: map[string]schema.Attribute{
            "id": schema.StringAttribute{
                Computed: true,
            },
            "last_updated": schema.StringAttribute{
                Computed: true,
            },
            "items": schema.ListNestedAttribute{
                // ...
```
验证资源模型 `orderResourceModel` 包含 `last_updated` 属性
```go
type orderResourceModel struct {
    ID          types.String     `tfsdk:"id"`
    Items       []orderItemModel `tfsdk:"items"`
    LastUpdated types.String     `tfsdk:"last_updated"`
}
```

## 实现资源更新
Provider 使用 Update 方法更新现有资源，具体包含以下几步：
1. 从 plan 中获取数据，并将其转换为 `orderResourceModel` 模型。该模型包括订单的 id 属性，该属性指定要更新的订单。
2. 根据 plan 的结果构建 API 的请求体，调用 API 来更新订单信息
3. 将响应内容更新到 Resource Schema 中
4. 更新 LastUpdated 属性
5. 将更新后的订单设置到 Terraform State 中

打开 `internal/provider/order_resource.go` 文件，替换以下内容
```go
func (r *orderResource) Update(ctx context.Context, req resource.UpdateRequest, resp *resource.UpdateResponse) {
    // Retrieve values from plan
    var plan orderResourceModel
    diags := req.Plan.Get(ctx, &plan)
    resp.Diagnostics.Append(diags...)
    if resp.Diagnostics.HasError() {
        return
    }

    // Generate API request body from plan
    var hashicupsItems []hashicups.OrderItem
    for _, item := range plan.Items {
        hashicupsItems = append(hashicupsItems, hashicups.OrderItem{
            Coffee: hashicups.Coffee{
                ID: int(item.Coffee.ID.ValueInt64()),
            },
            Quantity: int(item.Quantity.ValueInt64()),
        })
    }

    // Update existing order
    _, err := r.client.UpdateOrder(plan.ID.ValueString(), hashicupsItems)
    if err != nil {
        resp.Diagnostics.AddError(
            "Error Updating HashiCups Order",
            "Could not update order, unexpected error: "+err.Error(),
        )
        return
    }

    // Fetch updated items from GetOrder as UpdateOrder items are not
    // populated.
    order, err := r.client.GetOrder(plan.ID.ValueString())
    if err != nil {
        resp.Diagnostics.AddError(
            "Error Reading HashiCups Order",
            "Could not read HashiCups order ID "+plan.ID.ValueString()+": "+err.Error(),
        )
        return
    }

    // Update resource state with updated items and timestamp
    plan.Items = []orderItemModel{}
    for _, item := range order.Items {
        plan.Items = append(plan.Items, orderItemModel{
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
    plan.LastUpdated = types.StringValue(time.Now().Format(time.RFC850))

    diags = resp.State.Set(ctx, plan)
    resp.Diagnostics.Append(diags...)
    if resp.Diagnostics.HasError() {
        return
    }
}
```
构建并安装
```
go install .
```

## 验证更新方法
进入 examples/order 目录
```
cd examples/order
```
更新 `examples/order/main.tf` 文件
```hcl
resource "hashicups_order" "edu" {
  items = [{
    coffee = {
      id = 3
    }
    quantity = 2
    },
    {
      coffee = {
        id = 2
      }
      quantity = 3
  }]
}
```
运行 Terraform plan
```
$ terraform plan
hashicups_order.edu: Refreshing state... [id=1]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

   hashicups_order.edu will be updated in-place
  ~ resource "hashicups_order" "edu" {
      ~ id           = "1" -> (known after apply)
      ~ items        = [
          ~ {
              ~ coffee   = {
                  + description = (known after apply)
                    id          = 3
                  ~ image       = "/vault.png" -> (known after apply)
                  ~ name        = "Vaulatte" -> (known after apply)
                  ~ price       = 200 -> (known after apply)
                  ~ teaser      = "Nothing gives you a safe and secure feeling like a Vaulatte" -> (known after apply)
                }
                 (1 unchanged attribute hidden)
            },
          ~ {
              ~ coffee   = {
                  + description = (known after apply)
                  ~ id          = 1 -> 2
                  ~ image       = "/hashicorp.png" -> (known after apply)
                  ~ name        = "HCP Aeropress" -> (known after apply)
                  ~ price       = 200 -> (known after apply)
                  ~ teaser      = "Automation in a cup" -> (known after apply)
                }
              ~ quantity = 2 -> 3
            },
        ]
      ~ last_updated = "Thursday, 09-Feb-23 11:32:05 EST" -> (known after apply)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```
需要注意的是，id 属性在这里显示出了差异性，从已知属性变成未知属性 `(known after apply)`
```
      ~ id           = "1" -> (known after apply)
```
在更新操作中，这个属性不应该被改变。为了防止这个操作，接下来，我们将更新 id 属性，防止在 terraform update 的时候发生更新

## plan modifier
在 Terraform Plugin Framework 中，如果属性不能配置、并且在更新时不显示差异性，需要实现 `UseStateForUnknown()` plan modifier

(这里的不能配置，我个人理解即使在 resource 块中不能写，例如我们不能在 resource 块中手动指定 id 信息)
```go
            "id": schema.StringAttribute{
                Computed: true,
                PlanModifiers: []planmodifier.String{
                    stringplanmodifier.UseStateForUnknown(),
                },
            },
```
更新 import
```go
import (
    "context"
    "fmt"
    "strconv"
    "time"

    "github.com/hashicorp-demoapp/hashicups-client-go"
    "github.com/hashicorp/terraform-plugin-framework/resource"
    "github.com/hashicorp/terraform-plugin-framework/resource/schema"
    "github.com/hashicorp/terraform-plugin-framework/resource/schema/planmodifier"
    "github.com/hashicorp/terraform-plugin-framework/resource/schema/stringplanmodifier"
    "github.com/hashicorp/terraform-plugin-framework/types"
)
```
构建并安装
```
go install .
```

## 验证更新方法
进入 examples/order 目录
```
cd examples/order
```
执行 terraform apply，这个时候不会再显示 id 将会为 `(known after apply)`

provider 将会更新订单的属性，同时设置 `last_updated` 属性
```
$ terraform apply -auto-approve
#...

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.

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
        "id" = 2
        "image" = "/packer.png"
        "name" = "Packer Spiced Latte"
        "price" = 350
        "teaser" = "Packed with goodness to spice up your images"
      }
      "quantity" = 3
    },
  ])
  "last_updated" = "Thursday, 09-Feb-23 11:39:35 EST"
}
```