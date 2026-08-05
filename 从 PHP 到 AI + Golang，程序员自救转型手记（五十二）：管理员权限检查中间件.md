# 从 PHP 到 AI + Golang，程序员自救转型手记（五十二）：管理员权限检查中间件，AI 随意放行差点闯大祸

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “管理员和角色组的关联”，本期将完成：管理员权限检查中间件

# 管理员权限检查中间件

之前我们已经完成了 `RBAC` 权限模型的开发，接下来建立一个 `权限检查中间件`，对管理员的每个操作进行检查。

我将此中间件取名为 `AdminPermission`（权限），和已有的 `AdminAuth`（认证）对应。

和 `BuildAdmin` 不同，本系统认证和权限检查，都是在中间件完成，中间件的使用方式：

```go
// 必须登录
group.GET("/init", middleware.AdminAuth(), h.Init)

// 可选登录（若传递了正确的 token，中间件将在请求上下文注入登录管理员信息，没传递 token 也会放行）
group.POST("/upload", middleware.AdminAuthOptional(), h.Upload)

// 必须登录，同时检查权限
// 权限检查中间件就是本文的目标，它必须和 middleware.AdminAuth() 中间件配套使用，毕竟没登录咋验权呢
group.POST("/list", middleware.AdminAuth(), middleware.AdminPermission(), h.List)
```

> AI 做出来的首版验权中间件，在管理员未登录时，直接放行并表示由 AdminAuth 拦截（实际上 AdminPermission 比 AdminAuth 后执行，放行是绝对错误的），未匹配到权限规则时，居然也会放行等，相当于 AI 把放行当做默认操作，如果没有人工 review 或测试，这将是一个巨大的安全漏洞。

接下来直接让 AI 给规划一下：我需要一个 `admin_permission.go（AdminPermission）`中间件，需要使用它对管理员的每次请求进行验权，你觉得如何写比较合适？要求和上下文如下：

1. 权限节点种子数据在 `cmd\migrate\migrations\000003_seed.up.sql` 里边有
2. 权限规则表对应的模型在 `internal\model\admin.go` 中的 `AdminRule` 模型，你可以先读取它们的字段类型定义
3. 从 `gin` 请求上下文读取当前匹配到的路由，从中解析出 `权限节点的 name`，然后调用 `permission.Check` 验权
4. 对于 `List` 和 `Get` 请求，可以检查 `路由路径/read` 权限节点

AI 完成了种子数据的读取，并完成了初版实现，人工 `review`，在里边发现了一下问题：

## 错误的：跳过验权的路由

AI 先定义了一个 `permissionSkipPaths` map，里边存储了需要跳过验权的路由：

PS：AI 将 `/admin/auth/admin_log/get/:pk` 这种路径称为 `路由模式`，因为它们是：匹配成功的已注册路由，带原样的 `/get/:pk` 字样，而不是用户请求时的 `/get/1` 这种

```go
// permissionSkipPaths AdminPermission 跳过验权的路由模式
var permissionSkipPaths = map[string]struct{}{
	"/admin/init":                   {},
	"/admin/clear-cache":            {},
	"/admin/logout":                 {},
	"/admin/auth/admin_log/list":    {},
	"/admin/auth/admin_log/get/:pk": {},
}
```

然后在验权中间件里边进行跳过：

```go
// 跳过列表中的路由不验权
if _, skip := permissionSkipPaths[c.FullPath()]; skip {
    c.Next()
    return
}
```

这种跳过方式是错误的，因为根据我们的设计目标，**不验权的，直接不使用 AdminPermission 中间件就行了**，应该直接删除这些多余的逻辑。

## 严重问题：意外的放行

如下代码提取片段，中间件内有多处放行：

```go
admin := GetAdmin(c)
if admin == nil {
    // 未登录，由 AdminAuth 拦截 401，放行
    c.Next()
    return
}

action, nodeSuffix := extractAction(fullPath)
if nodeSuffix == "" {
    // 无法识别动作，放行
    c.Next()
    return
}
```

首先未登录的直接放行就是严重的错误，应该直接拦截，并提示未登录。

其次 `AI` 写了一个从 `fullPath` 末尾提取当前动作的方法，如果提取失败，也放行，这与我们的设计不符，唯二放行规则应该是超管和存在匹配的节点。

## 改造权限规则管理器的 Check 方法

我们设计的权限节点，并不会区分 `list` 和 `get` 请求，我本是想统一检查 `read` 权限节点。

AI 写出来的版本中规中矩，就是先检查 `list` 节点，没有就继续检查 `read` 节点，但是这会查库两次，所以我们重新规划一下。

我们从 `permission.Check` 方法入手，让它支持在一次查询中，验证两个节点就行了，如下：

1. 中间件内无需超管短路，`permission.Check` 里边已经有了
2. `permission.Check` 的 `ruleName` 改为 `ruleNames []string`，额外增加一个 `op` 参数，值是 `OR` 或 `AND`（默认），然后将：`name` 字段的数据库查询操作符号改为 `IN`，`op=OR` 只要有结果就返回 `true`，`op=AND` 时需要全部有结果才返回 `true`
3. 检查路径末尾是 `list` 的请求时，同时向 `permission.Check` 传递 `read` 的 `Check path`，`list` 的在前，`op` 设为 `OR`，其余权限节点检查全部用 `AND`

对 `Check` 方法的改造总体很简单，人工 `review` 只找到两个细节问题：

**一、应该对 `op` 统一转小写后，再进行检查**

**二、操作符号限定在 `or` 和 `and` 之间**

```go
// 原来的
if op != "" {
    op = "and"
}

// 改为
if op != "or" {
    op = "and"
}
```

## 中间件逻辑重新整理

现在可以重新梳理整个中间件的逻辑，核心其实非常简单，被 AI 搞复杂了，就一件事而已：**根据路由模式构建权限节点的 name**，然后直接传递给 `permission.Check` 就行了。

在 `AI` 之前的规划中，我们得知，原来我们可以获取到当前请求的：路由模式（匹配成功的已注册路由，带原样的 `/get/:pk` 字样，而不是用户请求时的 `/get/1` 这种）。

如果是用户请求的 `URL`，要提取权限节点确实有点麻烦，因为参数数量是不固定的，但是现在从固定的 `路由模式` 提取节点名称，可就太简单了：

示例路由模式：`/admin/auth/admin-log/get/:pk`

1. 去掉开头的 `/admin/` 得到 `auth/admin-log/get/:pk`
2. 以 `/` 切割为 `[auth, admin-log, get, :pk]`，从第一个含有 `:` 符号的元素开始全部丢弃，剩下的用 `/` 再拼回 `string`，就是权限节点的 `name` 了

|路由模式|权限节点 name|
|--|--|
|/admin/auth/admin-log/get/:pk|auth/admin-log/get|
|/admin/auth/test/create/:pk/info|auth/test/create|
|/admin/auth/test/update|auth/test/update|

节点 name 是管理员自己在菜单规则管理中定义的，只要遵循以上规则，节点自然就能匹配到，匹配不到也没关系，程序不会放行（之前 AI 认为可以放行😂），管理员再自行修改就行了。