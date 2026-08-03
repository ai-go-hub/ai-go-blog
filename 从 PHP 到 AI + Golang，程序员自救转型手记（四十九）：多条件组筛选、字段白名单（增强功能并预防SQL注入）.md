# 从 PHP 到 AI + Golang，程序员自救转型手记（四十九）：多条件组筛选、字段白名单（增强功能并预防SQL注入）

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “菜单规则管理实现”，本期将完成：多条件组筛选、字段白名单

# 筛选字段白名单

白名单说白了就是先在模型定义好所有字段，然后检查前端传递的 `筛选字段名称`，是否在模型定义内，否则就算字段不存在，主要是为了安全着想，避免 SQL 注入等。

之前我们在仓储层已经做好了 `检查一个字段是否在模型内` 的方法，而且 `排序字段名` 已经做过白名单检查了，之前没有做筛选字段的检查，主要是因为筛选功能还不完善。

这里直接让 AI，在服务层，使用 `检查一个字段是否在模型内` 方法，检查所有筛选条件提供的 `Field` 即可。

# 筛选操作符号白名单

众所周知，一个筛选条件的基本结构是这样的：`Where("name = ?", "jinzhu")`；

我们已经完成了字段名的检查，字段值则是直接使用 `?` 这种插值的语法，无需担心注入问题，那么操作符号（`=、>、<、LIKE` 等）是否也需要检查呢？

答案是肯定需要过滤：比如前端传递的操作符号是 `= 1 or id > 0 -- `，SQL `status = 1 or id > 0 --  ?`。

接下来就是如何优雅的实现操作符白名单，我们之前已经写了一个 `GetOperatorByAlias` 方法，用于将前端 `eq` 转为 `=`、`ne` 转为 `!=` 等，所以直接利用它，将所有允许的操作符号列出来就行了，不允许的返回空字符或者 nil，拼接 where 时，去掉 nil 的筛选条件即可。

# 多条件组筛选

目前支持的筛选条件数据是这样的：

```go
type Where struct {
	Field    string
	Operator string
	Value    any
}
```

前端传递一个 `[]Where`，服务端将切片中的每一项组装为一个 `GORM` 的 `scope`，然后传递给仓储层使用，所以 `scopes` 直接的连接符号统一是 `AND`，比如：

```ts
[
    {
        field: 'a',
        operator: 'eq',
        value: '1',
    },
    {
        field: 'b',
        operator: 'eq',
        value: '2',
    },
]
```

将组装为两个 `scope`，最终 SQL 类似：`WHERE a=1 AND b=2`

我希望实现，前端传递：

```ts
[
    {wheres: []Where, or: true},
    {wheres: []Where, or: false},
]
```

`wheres: []Where` 字段代表以前版本的筛选条件数据，而且保持数组格式，可以传递多个，而这些条件之间的连接符号，取用 `or` 字段的值，`true` 则使用 `OR`，否则使用 `AND`，示例如下：

```ts
[
    {
        wheres: 
        [
            {
                field: 'a',
                operator: 'eq',
                value: '1',
            },
            {
                field: 'b',
                operator: 'eq',
                value: '2',
            },
        ],
        or: true
    },
    {
        wheres: 
        [
            {
                field: 'c',
                operator: 'eq',
                value: '3',
            },
            {
                field: 'd',
                operator: 'eq',
                value: '4',
            },
        ],
        or: false
    },
]
```

一个 `[]Where` 组装为一个 `GORM` 的 `scope`，最终 SQL 类似：`WHERE (a=1 OR b=2) AND (c=3 AND d=4)`

最终可以达成复杂查询下 `scopes` 减少，和单个 `scope` 内的多个筛选条件支持自定义 `OR / AND` 连接符号。

服务端支持了这种 `多条件组筛选` 后，前端的 `表格` 和 `远程下拉` 都会受益，因为它们就是基于服务端 `List` 方法获取数据的，而且不用担心安全问题，我们的字段名和操作符号都有白名单，值则是使用 `?` 插值语法天然防范 SQL 注入。