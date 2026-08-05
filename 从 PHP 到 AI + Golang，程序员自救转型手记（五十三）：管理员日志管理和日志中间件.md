# 从 PHP 到 AI + Golang，程序员自救转型手记（五十三）：管理员日志管理和日志中间件，日志标题读取的极限优化

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “管理员权限检查中间件”，本期将完成：管理员日志管理和日志中间件

# 管理员日志中间件

一个全局中间件，对所有 `/admin/` 开头的请求生效，之所以不作为分组中间件注册，因为我们路由注册时，可能会建立多个类似下方的分组：

```go
group := r.Group("/admin/auth/admin")
group := r.Group("/admin/auth/group")
```

即，我们的路由都是子模块自行负责注册的，不是统一注册，所以使用分组级中间件不方便。

> 后续有改动：未来会在制作 “自定义后台入口” 功能时，统一注册 admin 路由的 group，并将日志中间件直接注册到该 group 上，就不再需要全局注册了。

## 日志标题读取

### 从权限规则表读取

用户请求 `/admin/auth/admin/update` 接口，日志中不仅要记录 `请求用户、请求URL、请求体` 等，更重要的是 `日志标题` 数据，最好能一眼就看出管理员干了啥。

我们刚好有 `admin_rules` 表，管理员的每个请求都经过了验权，能验权自然于该表有权限数据：

- `/auth/admin` 从规则表可查出标题为 `管理员账号管理` 的权限规则
- `/auth/admin/update` 从规则表可查出标题为 `更新` 的权限规则

最终组装为 `管理员账号管理 - 更新` 作为日志标题，看起来很完美，但是：一个后台请求中，我们已经读了 `tokens`、验权读了 `admin_rules`，现在为了个日志标题，又加两个 SQL 查询，这非常不合适，所有这里让 AI 做了个在单条 SQL 查询中准备好标题的办法，核心代码是这样的：

```go
fmt.Sprintf(
    "SELECT c.title, COALESCE(p.title, '') AS parent_title FROM %s AS c LEFT JOIN %s AS p ON p.id = c.pid WHERE c.name = ?",
    tableName, tableName,
)
```

### 控制器层、服务层指定标题

这一点非常有必要，比如管理员登录接口 `login`，这个接口压根不在权限规则表有记录，而这个接口的日志，又不得不记录，那么我们可以在控制器里边：

```go
c.Set(middleware.CtxAdminLogTitleKey, "登录")
```

向请求上下文设定一个自定义的日志标题，日志中间件里边只需要先调用 `c.Next()`，让控制器的代码先跑，即可在写入日志时读取到自定义标题数据：

```go
// 检查控制器层是否已设置自定义日志标题
customTitle, hasCustomTitle := c.Get(CtxAdminLogTitleKey)

// 异步写入日志
// 先在外面把变量提前捕获好，在开 goroutine 慢慢处理和入库，避免挂在请求上下文的数据随请求结束丢失等
go func() {
    // ......
}
```

## 排除

日志也不用每条请求都做记录，有以下排除规则：

### 请求排除

1. 仅记录管理后台请求
2. 仅记录 `POST` 请求
3. 排除 `chunked` 请求
4. 额外排除 `list` 请求（因为 `list` 请求也可能是 `POST`，筛选、排序等数据 `GET` 不够，且 `GET` 直接发送 `json` 数据不方便）

### 敏感字段过滤或排除

我们可以利用 GORM 的模型的创建前钩子，在日志写入前，对 `password、salt、token` 等敏感字段进行过滤。

日志是异步写入的，在模型钩子中处理，可以避免解析请求体等动作拖慢用户侧响应速度。

## 非超管只能看自己的日志

在控制器和服务层直接通过拼接 `Where` 进行限制，避免任何可能的越权。

```go
// 非超管则只查询自己的日志
if !super {
    opts.Wheres = append(opts.Wheres, service.WhereGroup{
        Wheres: []service.Where{{
            Field:    "admin_id",
            Operator: "eq",
            Value:    extension.AdminSession.ID,
        }},
    })
}
```

# 管理员日志管理

## 前端请求体展示优化

到这里一般的后台就是直接在 `日志详情` 直接显示 `JSON` 了，比如：`{"username": "admin", "password": "****", ......}`，不过我还是在这里用了个 `el-tree`，并写了工具函数将 `JSON` 构建为该组件可以合理使用的结构，最终效果如下：

![](./images/53/1.png)

## 双击打开详情

表格的默认设计是双击打开编辑，稍微改一下即可；另外直接将双击行的数据作为 `详情弹窗` 的数据，无需额外加载。

自定义一个表格行级操作按钮：

```ts
let optButtons: OptButton[] = [
    {
        render: 'tip',
        name: 'info',
        title: '详情',
        text: '',
        type: 'primary',
        icon: 'lucide-info',
        class: 'table-row-info',
        click: (row: TableRow) => {
            openInfo(row)
        },
    },
]
```

利用双击单元格前钩子重写双击操作：

```ts
tableManager.opts.before!.columnDblclick = ({ row }) => {
    openInfo(row)
    return false
}
```

接下来在 `openInfo` 函数打开详情弹窗，主要是 `tableManager.form.operate = 'info'`，详情弹窗那边再 `<Info v-if="tableManager.form.operate == 'info'" />` 就行了

```ts
/**
 * 点击查看详情按钮响应
 */
const openInfo = (row: TableRow) => {
    if (!row) return
    // 数据来自表格数据，未重新请求 API，深克隆，不然可能会影响表格
    let rowClone = cloneDeep(row)

    rowClone.data = rowClone.data ? [{ label: '点击展开', children: buildJsonToElTreeData(JSON.parse(rowClone.data)) }] : []
    tableManager.form.extend!['info'] = rowClone
    tableManager.form.operate = 'info'
}
```