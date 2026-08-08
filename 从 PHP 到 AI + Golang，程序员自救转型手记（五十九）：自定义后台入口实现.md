# 从 PHP 到 AI + Golang，程序员自救转型手记（五十九）：前后端自定义后台入口实现

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作（欢迎 star~），全程记录分享。

在上一期，我们进行了 “后台系统配置管理实现”，本期将完成：自定义后台入口实现

# 自定义后台入口实现

作为一个后台管理系统，开发者能够自定义入口还算比较重要，比如可以将 `域名/admin` 进入后台改为 `域名/dfwef1dki`，能够实现隐藏入口的目的。

## 服务端

先直接问问AI：我要实现 "自定义后台入口" 的功能，前端注册路由我已有方案，现在的问题是：gin 这边注册的 /admin 路由，在什么地方修改合适？能否走现有的文件配置系统？还是只能使用固定常量，所有注册 `/admin` 的路由的地方，都使用该常量作为前缀？是否有更好的办法？

> 其实我个人也是优先考虑文件配置方案，这里之所以提供常量选项，是因为配置总是需要先读取了才有，而我们的路由注册是利用了 `init` 实现自动发现，使用配置或许会不太方便？

AI 首先给出的方案也是使用 `文件配置系统` 实现，增加一个 `app.admin_path` 配置项，它说常量方案技术上可行但有一个限制：改常量需要重新编译。实际上使用配置也是需要重新编译的，因为就算配置重载了，路由也不会重新注册，要实现路由重新注册那就麻烦了，还得清理旧的路由等，没必要。

**也就是说，重新编译是不可避免的，而使用文件配置系统，或者常量方式都可以实现需求。**

那当然是文件配置系统，常量没有配置项方便和易管理，不过配置名由 AI 生成的 `app.admin_path` 改为 `server.admin_base_route_path`，路由路径属于 `server` 层面的配置，不应该放在 `app` 下，让 AI 开始实现。

`review` 时看懂了 AI 的实现思路：

以往我们的路由注册，是子模块在 `init()` 内调用路由注册函数，如下

```go
func init() {
	registry.Register(func(r *gin.Engine) {
		repo := repoCommon.NewConfigRepository()
		svc := svcRoutine.NewConfigService(repo)
		h := handlerRoutine.NewConfigHandler(svc, repo)

		group := r.Group("/admin/routine/config")
		h.RegisterRoutes(group)
	})
}
```

`init()` 函数会在包载入时被自动执行，而 `registry.Register` 函数会将传入的 `路由注册函数` 全部收集起来，然后系统的入口文件内会统一调用 `router.Setup` 完成所有路由的注册：

```go
// Setup 遍历所有已注册的路由模块，传入 Engine 完成注册
func Setup(r *gin.Engine) {
	for _, fn := range registry.Routes {
		fn(r)
	}
}
```

AI 的自定义入口思路是：先定义一个 `registry.RegisterAdmin`，它不是接受 `Engine`，而是接受一个 `gin 的路由 group`，我们只需要将后台的路由注册函数由 `registry.Register` 换为 `registry.RegisterAdmin` 即可：

```go
func init() {
	registry.RegisterAdmin(func(group *gin.RouterGroup) {
		repo := repoAdmin.NewAdminRuleRepository()
		svc := svcAuth.NewAuthAdminRuleService(repo)
		h := handlerAuth.NewAuthAdminRuleHandler(svc)

		subGroup := group.Group("/auth/rule")
		h.RegisterRoutes(subGroup)
	})
}
```

`router.Setup` 现在会单独注册全部的后台路由，并且为它传递 `adminGroup`，如下：

```go
// Setup 遍历所有已注册的路由模块，传入 Engine 完成注册
func Setup(r *gin.Engine) {
	for _, fn := range registry.Routes {
		fn(r)
	}

	// 后台路由分组
	adminGroup := r.Group(config.Get().Server.AdminBaseRoutePath)

	// 注册全部后台路由
	for _, fn := range registry.AdminRoutes {
		fn(adminGroup)
	}
}
```

额外还发现了以下细节点需额外处理：

### AdminLog 中间件可以移动注册位置

以前 `AdminLog` 中间件是在 `serve.go` 中全局注册的：

```go
// 注册管理员操作日志中间件
engine.Use(middleware.AdminLog())
```

现在可以直接改到 `router.Setup` 的 `后台路由分组` 上。

```go
func Setup(r *gin.Engine) {
	for _, fn := range registry.Routes {
		fn(r)
	}

	adminGroup := r.Group(config.Get().Server.AdminBaseRoutePath)

    // 注册管理员操作日志中间件
	adminGroup.Use(middleware.AdminLog())

	for _, fn := range registry.AdminRoutes {
		fn(adminGroup)
	}
}
```

### BuildCheckPath 的改动优化

`BuildCheckPath` 的内部会先去掉传入字符串的 `/admin/` 前缀，相当于以前是写死的 `admin 路由前缀字面量`，现在我们要支持自定义，此处自然也应该使用变量替换，即 `config.Get().Server.AdminBaseRoutePath+"/"` 替换 `/admin/`。

但 AI 实现的是 `BuildCheckPath` 额外接受一个 `adminPath` 参数，而不是自己去读 `config` 数据：

```go
func BuildCheckPath(fullPath string, adminPath string)
```

这里完全可以改为函数内自行读取，因为此函数本来就在 `infra` 文件夹内，不是 `pkg` 之类的公共包。

以上只是一个案例，这里让 AI 将所有适合自行读取 `admin 路由前缀配置` 的，全部都自行读取，不要加参数去传。

## 前端

前端自定义后台入口是本来就支持的，我们的 `admin` 路由前缀全部都是使用的 `常量` ，需要修改时，调整常量值就行了。

### 前端请求函数修改

前端请求函数还是在请求原本的 `/admin`，并没有使用 `admin` 路由前缀常量，而在 `API` 请求函数里边该常量其实非常不方便，全局替换的方式更加合理，在请求封装 `src\utils\request.ts` 的拦截器中：

```ts
// 自定义后台入口支持：将请求 URL 中的 /admin 前缀替换为实际配置的后台路径
// opts.replaceCustomAdminPath 默认为 true
if (opts.replaceCustomAdminPath !== false && adminBaseRoutePath !== '/admin' && /^\/admin\//.test(config.url!)) {
    config.url = config.url!.replace(/^\/admin\//, adminBaseRoutePath + '/')
}
```

> 如上，请求接口和注册路由时，前端终究需要知道自定义的后台入口值是多少，所以避免不了有泄露的可能，所以我开篇只是将此功能描述为：还算比较重要
