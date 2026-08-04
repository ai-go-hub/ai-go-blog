# 从 PHP 到 AI + Golang，程序员自救转型手记（五十一）：管理员和角色组的关联

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “增加管理员角色组管理”，本期将完成：管理员和角色组的关联

# 管理员和角色组的关联

目前系统内，还没有一处使用了关联表，这里就以 `管理员` 和 `角色组` 来跑路关联的流程，先直接让 AI 提供建议，切换为 `plan` 模式：接下来需要完成 `Admin` 和 `AdminGroupAccess` 模型的关联，它们定义于 `@internal/model/admin.go`，用什么方式关联比较好？同时，我需要为基类增加快速配置关联的功能，在什么地方配置，配置名叫什么比较好？

## GORM 预加载复习

以上提示词说的比较笼统，甚至使用什么关联模式都是让 AI 提供方案，在它分析的时候，这里我也是赶紧打开 GORM 的学习笔记，查看一下各种关联方式的预加载写法。

> 我要做不只是写好管理员和角色组的预加载（这直接重写仓储层就行），我的目的是在控制器配置一个字段，仓储层就能给我加载好供我使用，即 基类 对关联预加载的支持。

预加载的写法有：

### Preload

```go

type User struct {
  gorm.Model
  Username string
  Orders   []Order
}

type Order struct {
  gorm.Model
  UserID uint
  Price  float64
}

// 预加载 Order 数据，参数二可以传递一个 func(db gorm.PreloadBuilder) error 闭包函数，里边拼接查询条件、排序方式等
.Preload("Order", nil)

// 条件预加载
.Preload("Orders", "state NOT IN (?)", "cancelled")

// 嵌套
.Preload("Orders.OrderItems.Product")
```

### Joins 预加载

还是上面的模型定义，使用 Joins 预加载是这样写的：

```go
// 参数二同样可以传递一个闭包，签名是: func(db gorm.JoinBuilder, joinTable clause.Table, curTable clause.Table) error
.Joins(clause.JoinTarget{Association: "Order"}, nil)
```

Joins 预加载的性能无疑是最好的，因为它在 `单条 SQL 中` 使用左连接来加载关联数据。

## AI 初版完成

提供的方案一般般，在刚刚复习完笔记的我来看，甚至有点 `low`，首先是控制器加了个可选参数函数 `WithPreloads`，然后将设置的 `Preloads` 传递到仓储层，于仓储层直接：

```go
// 预加载关联
for _, p := range opts.Preloads {
    q = q.Preload(p, func(pb gorm.PreloadBuilder) error { return nil })
}
```

`Preload` 写法，然后加载是加载了，闭包函数是固定的 `return nil`，以下是找到的问题或改善点。

### 参数和 GORM 的 Preloads 同步

> 控制器层，定义预加载选项 `Preloads` 时，支持到第二个参数，其类型和 `GORM` 的 `Preloads` 第二个参数同步。

```go
// Preload 预加载关联配置
type Preload struct {
	Name  string                          // 关联名称
	Query func(gorm.PreloadBuilder) error // 可选的自定义查询条件，为 nil 时仅按名称预加载
}
```

AI 上来就定义了一个 `type Preload struct`，第一个字段为 `Name string`，我对比了一下 `Preload` 的签名，它第一个参数是 `association string`，这个 AI 不严谨啊，让它先改正。

### 控制器层关联配置优化

```go
// PreloadFields 声明对应方法的预加载关联配置
type PreloadFields struct {
	Get    []repository.Preload
	List   []repository.Preload
	Update []repository.Preload
	Create []repository.Preload
}
```

这里是 AI 模仿的 `ActionFields`，给 `Get、List、Update、Create` 各提供了一个定义 `Preload` 的位置，其中在 `Update、Create` 中这东西基本没用，直接去掉，`List` 几乎是主要使用方，`Get` 使用频率低且就算额外查了 `List` 的 `Preload` 也没关系。

总而言之，简化为单个 `Preloads []repository.Preload` 配置即可，不需要区分方法。

### 仓储层 Preload 调用优化

AI 原本是这样写的：

```go
for _, p := range opts.Preloads {
    query := p.Query
    if query == nil {
        query = func(pb gorm.PreloadBuilder) error { return nil }
    }
    q = q.Preload(p.Association, query)
}
```

实际上，直接写 `q = q.Preload(p.Association, nil)` 都是可以的，并不需要 `nil` 守卫，直接去除。

### 关联筛选查询

我们的表格是支持多字段筛选的，比如使用关联模型 `group` 的 `name` 字段模糊查询，前端几乎无需改动，表格组件和表格管家的公共搜索组件自然就能传递这种数据至服务端：

```ts
{
    wheres: 
    [
        {
            field: 'group.name',
            operator: 'ILIKE',
            value: '1',
        },
        {
            field: 'group.title',
            operator: 'ILIKE',
            value: '1',
        },
    ],
    or: true
}
```

问题在于，服务端在何处应用这些筛选数据，首先肯定是考虑使用 `Preload(association string, query func(db gorm.PreloadBuilder)` 的第二个闭包函数，但这个理解是错误的：

```go
gorm.G[Admin](db).Preload("Group", func(pb PreloadBuilder) error {
    pb.Where("name LIKE ?", "%foo%")
    return nil
}).Find(ctx)
```

产生两条 SQL：

```sql
-- 1. 主查询：所有 Admin，全部返回
SELECT * FROM admins;
-- 2. 预加载：只加载 name LIKE 'foo' 的 group
SELECT * FROM admin_groups WHERE id IN (?, ?, ?) AND name LIKE '%foo%';
```

结果是：所有 `Admin` 都会在结果里，只是部分行（不匹配的）的 `Group` 数据将被置为 `nil`。

我们要的筛选语义显然是：只返回 `group.name` 匹配的 `Admin` 的行。这需要主查询过滤，`Preload` 帮不上。

唯一干净的方式：子查询，顺着关联链层层套 `EXISTS/IN` 子查询即可。

```sql
SELECT * FROM admins WHERE admins.group_id IN (
    SELECT id FROM admin_groups WHERE name LIKE '%foo%'
);
```

就按子查询的方式，让 AI 实现出来，在已有的 `BuildWhereScopes、BuildWhereExpr` 完成关联表字段名检查等；这里实际上的实现比较复杂，而且人工 `review` 了很久，不过好在是实现出来了，而且很优雅。

现在前端可以直接发送任意表的筛选数据，最终组装好对应的 SQL 进行查询，比如主表的 `name LIKE '%foo%'`，关联表的 `group.name LIKE '%foo%'`。