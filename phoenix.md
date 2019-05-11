# Phoenix Router CheatSheet

运行 `mix phx.routes` 可以查看当前 phoenix 项目中的所有 routes.

使用 Router.Helpers.{xxx_path}(Myapp.Endpoint, {function}, {params}) 即可生成对应的路径.


# Phoenix Controller CheatSheet

## Controller

- 返回 json 数据:

```ex
def show(conn, %{"id" => id}) do
  json(conn, %{id: id})
end
```