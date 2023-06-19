# 实现资源删除
在本教程中，您将为 order resource 实现资源的删除操作

## 前置
* Go 1.19+
* Terraform v1.0.3+
* Docker and Docker Compose

## 实现删除方法
provider 使用 `Delete` 方法来删除一个现存资源
1. 从 state 中获取数据
2. 删除现有的 order

打开 `internal/provider/order_resource.go` 文件
```go
func (r *orderResource) Delete(ctx context.Context, req resource.DeleteRequest, resp *resource.DeleteResponse) {
    // Retrieve values from state
    var state orderResourceModel
    diags := req.State.Get(ctx, &state)
    resp.Diagnostics.Append(diags...)
    if resp.Diagnostics.HasError() {
        return
    }

    // Delete existing order
    err := r.client.DeleteOrder(state.ID.ValueString())
    if err != nil {
        resp.Diagnostics.AddError(
            "Error Deleting HashiCups Order",
            "Could not delete order, unexpected error: "+err.Error(),
        )
        return
    }
}
```
构建并安装
```
go install .
```

## 验证删除
进入 examples/order 目录
```
cd examples/order
```
删除配置
```
$ terraform destroy -auto-approve
#...
Destroy complete! Resources: 1 destroyed.
```