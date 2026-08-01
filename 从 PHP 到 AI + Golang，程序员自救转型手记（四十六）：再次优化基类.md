# 从 PHP 到 AI + Golang，程序员自救转型手记（四十六）：再次优化基类

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “前端远程下拉输入组件”，本期将完成：再次优化基类

# 再次优化基类

> 基类严格来讲不是 go 里边的概念，在本 blog 中使用它范指 基控制器、基服务、基仓储

## 列表数据有序保证

很多项目都没有做这个，甚至都不知道这个概念，这其实是一个数据库（含 PostgreSQL、MySQL）使用的小坑，在未指定排序子句 `ORDER BY` 时：**数据库并不保证排序，即先查到哪行就输出哪行且不保证多次相同查询中的数据输出顺序**

也就是说，只要是列表查询，开发者必需传递 `ORDER BY` 子句，如果不传递，你看的有序只是临时的，下次未必还是这个顺序，这在分页等场景可能是灾难性的。

实践中，我们以用户传递的排序字段优先，如果没有传递，默认读取主键字段做有序保证，**且就算有传递排序字段，依旧以主键字段托底** 即可（防止用户排序字段值有一样的，此时又会丢失有序保证）。

## 基控制器封装 BuildSerOpts 方法，用于构建服务层选项数据

比如 `Get` 方法，现在组装服务层选项是这样写的：

```go
entity, err := h.svc.Get(c, service.Options{
    Omit:       h.cfg.Omit.Get,
    Select:     h.cfg.Select.Get,
    PrimaryKey: c.Param("pk"),
})
```

增加 `BuildSerOpts` 方法后，可以简化为：

```go
opts := h.BuildSerOpts(c, "Get", Request{
    PrimaryKeyValue: c.Param("pk"),
})

entity, err := h.svc.Get(c, opts)
```

即：无需再单独传递 `Omit` 和 `Select` 选项；而且在 `List` 方法中收益更大，代码可以由原来的 `7` 行改为单行：`opts := h.BuildSerOpts(c, "List", req.Request)`。

## 基类增加自定义扩展参数解析函数

目前，我们可以为基类配置 `OmitFields`（各动作的 Omit 黑名单字段）、`SelectFields`（设置各动作的 Select 白名单字段）、`Adapter`（数据适配器）。

以上都是固定的参数，但在实际业务场景中，很多时候需要能够向底层传递一些 `自定义的扩展参数`，规划如下：

1. 仓储层与自定义扩展参数无关，最多传递到服务层即可
2. 服务层额外接受一个 `Extension` 的 `Options`，类型为 `any`，增加参数后完整的 `Options` 如下：

    ```go
    // Options 通用服务操作选项
    // 每个方法所需要的选项都可以在此找到，但并非每个方法都会使用全部的选项
    type Options struct {
        OmitFields       []string // 排除出入库字段，会传递给仓储层的 Omit 方法
        SelectFields     []string // 选择出入库字段，会传递给仓储层的 Select 方法
        Wheres           []Where  // 查询条件，用于构建 WhereScopes，然后传递给仓储层的 Scopes 方法
        SortField        string   // 排序字段，用于构建 OrderScope
        SortOrder        string   // 排序方式
        Page             int      // 页码，用于构建 PaginateScope
        Limit            int      // 每页条数
        PrimaryKeyValue  string   // 主键值，目前可供 Get、Update 方法获取数据行
        PrimaryKeyValues []string // 主键切片，目前可供 Delete 方法批量删除行
        Extension        any      // 任意自定义扩展参数
    }
    ```

3. 控制器层不仅仅是接受一个 `Extension any`，更合理的方式是接受一个函数 `ExtensionResolver func(c *gin.Context) any`，可以将它称之为 `扩展数据解析器`，函数返回值将被赋值给服务层的 `Extension any`（直接于之前增加的 `BuildSerOpts`（构建服务层选项数据）方法中调用一下解析器再赋值即可）。

以上是需求描述，也是给 AI 的提示词（Blog 会比实际发送的更加详细很多），AI 帮忙实现以上需求后，我特意实际使用了一下：

比如我们需要向服务层传递一个自定义的 `AdminSession` 参数，又不想重写控制器层的方法，首先在服务层定义好结构体（字段列表和类型完全自定义）：

```go
// AuthAdminRuleExtension 规则列表扩展参数
type AuthAdminRuleExtension struct {
	AdminSession *dto.AdminSession
}
```

控制器 `handler.NewHandler` 时，额外使用 `WithExtension` 传递一个 `扩展数据解析器` 即可：

```go
// NewAuthAdminRuleHandler 创建菜单和权限规则管理控制器实例
func NewAuthAdminRuleHandler(svc *svcAuth.AuthAdminRuleService) *AuthAdminRuleHandler {
	return &AuthAdminRuleHandler{
		Handler: handler.NewHandler(svc,
            // 传递 `扩展数据解析器`
			handler.WithExtension(func(c *gin.Context) any {
				return &svcAuth.AuthAdminRuleExtension{
                    // 传递 AdminSession
					AdminSession: middleware.GetAdmin(c),
				}
			}),
		),
		svc: svc,
	}
}
```

服务层，读取和使用扩展数据：

```go
// 从控制器传来的 `管理员信息` 扩展数据
extension, ok := opts.Extension.(*AuthAdminRuleExtension)
if !ok || extension.AdminSession == nil {
    return nil, errors.New("参数错误，缺少 AdminSession 扩展数据")
}

// opts.xxxx 参数，基本每个方法都有接受
// 此时就可以使用 extension.AdminSession 了
```

## Create 默认忽略传递的主键字段入库

我们已经设计了 `OmitFields` 选项，用于配置忽略入库的字段，但是执行创建动作时，主键字段总是应该被忽略，我们在很早之前就写好了获取当前模型主键的方法，这里直接配合它，在 `未设定 OmitFields 选项时，默认忽略掉主键字段`。
