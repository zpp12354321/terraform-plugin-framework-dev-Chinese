# 实现资源导入
在本教程中，您将为 provider 的 order resource 添加导入功能。为此，您将：

1. 实现资源导入  
通过 terraform import 命令获取给定的订单 ID，使 Terraform 能够开始管理现有订单
2. 验证导入功能  
确保资源导入功能按预期工作

## 前置
* Go 1.19+
* Terraform v1.0.3+
* Docker and Docker Compose

## 实现导入功能
provider 使用 `ImportState` 方法导入现有资源，具体步骤为：

使用 `resource.ImportStatePassthroughID()` 方法从 `terraform import` 命令中获取 ID 值并将其保存到 id 属性。

**如果没有发生错误，Terraform 将自动调用资源的 Read 方法来导入其余的 Terraform 属性状态**（重点关注！！）。

由于 Read 方法只需要依赖 id 属性，我们不需要在对该方法做额外的修改。

打开 internal/provider/order_resource.go 文件，添加一个新的 ImportState 方法来实现在 order_resource.go 中导入资源，如下所示。
```go
func (r *orderResource) ImportState(ctx context.Context, req resource.ImportStateRequest, resp *resource.ImportStateResponse) {
    // Retrieve import ID and save to id attribute
    resource.ImportStatePassthroughID(ctx, path.Root("id"), req, resp)
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
    "github.com/hashicorp/terraform-plugin-framework/path"
    "github.com/hashicorp/terraform-plugin-framework/resource"
    "github.com/hashicorp/terraform-plugin-framework/resource/schema"
    "github.com/hashicorp/terraform-plugin-framework/resource/schema/planmodifier"
    "github.com/hashicorp/terraform-plugin-framework/resource/schema/stringplanmodifier"
    "github.com/hashicorp/terraform-plugin-framework/types"
)
```
确保资源实现 `ResourceWithImportState` 接口
```go
// Ensure the implementation satisfies the expected interfaces.
var (
    _ resource.Resource                = &orderResource{}
    _ resource.ResourceWithConfigure   = &orderResource{}
    _ resource.ResourceWithImportState = &orderResource{}
)
```
构建和安装
```
$ go install .
```

## 验证导入功能
进入 `examples/order` 目录
```
cd examples/order
```
执行 `terraform apply`，确保资源创建成功
```
$ terraform apply -auto-approve
#...
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

edu_order = {
  "id" = "2"
  "items" = tolist([
    {
#...
```
从 tf.state 文件中获取资源的 id 属性
```
$ terraform show
# hashicups_order.edu:
resource "hashicups_order" "edu" {
    id           = "2"
    items        = [
         (2 unchanged elements hidden)
    ]
    last_updated = "Wednesday, 14-Dec-22 11:18:20 CST"
}
#...
```
将现有订单从 tf 中移除（并不是删除，而是不需要通过 tf 管理）
```
$ terraform state rm hashicups_order.edu
Removed hashicups_order.edu
Successfully removed 1 resource instance(s).
```
导入现有资源
```
$ terraform import hashicups_order.edu 2
hashicups_order.edu: Importing from ID "2"...
hashicups_order.edu: Import prepared!
  Prepared hashicups_order for import
hashicups_order.edu: Refreshing state... [id=2]

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.
```
确保资源导入成功
```
$ terraform show
# hashicups_order.edu:
resource "hashicups_order" "edu" {
    id    = "2"
    items = [
         (2 unchanged elements hidden)
    ]
}

##...
```