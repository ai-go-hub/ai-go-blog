# 从 PHP 到 AI + Golang，程序员自救转型手记（二十八）：管理员登录态保持相关优化

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “管理员登录验证码开关”，本期将完成：管理员登录态保持优化、验权中间件

# 管理员登录态保持相关优化

管理员身份 token 的有效期，默认是 3 天，勾选保持登录时，有效期是 30 天，有效期目前是硬编码在源代码中的，现在将其改为动态配置：

1. 增加管理员 `token` 过期时间配置项，默认值：记住登录 30 天，否则为 3 天
2. 于 `@internal/service/admin/admin.go` 中应用这些配置项
3. 配置项加在 `@config/config.yaml` 的 `token` 配置之下，增加有效期字段

AI 写好的核心代码如下，人工 `review` 没发现啥问题：

```yaml
# config\config.yaml 文件

token:
  driver: database
  expire:
    admin: 72           # 管理员 token 有效期，单位：小时（3 天）
    admin_remember: 720 # 管理员 token 有效期（记住登录），单位：小时（30 天）
```

登录接口内，计算过期时间的逻辑修改：

```go
// 计算过期时间
expireCfg := config.Get().Token.Expire
expiredAt := time.Now().Add(time.Duration(expireCfg.Admin) * time.Hour)
if req.Remember {
    expiredAt = time.Now().Add(time.Duration(expireCfg.AdminRemember) * time.Hour)
}
```

由于我们已经将 `token` 有效期改为动态配置，前端登录页面的 “记住 30 天” 的选项就不合理了，将其改为 “记住我” 即可。

最后对 AI 写的代码进行实测，直接从数据库检查勾选记住我之后的 token 过期时间是否正确。

# 验权中间件

接下来完成 `token` 的接受和验证，按照社区惯例，我选择使用中间件来完成这项工作，先让 CC 做个初版出来看看效果再决定后续步骤：

帮我实现 `admin_auth` 中间件，接受 `Authorization: Bearer token` 格式的 `token`，并使用 `token` 管理器（ @internal/infra/token/token.go ）获取用户信息，保存到请求上下文中。

```bash
● 编译已通过。internal/middleware/admin_auth.go 实现了：

  - 提取 — 从 Authorization: Bearer <token> 头解析 token
  - 校验 — 通过 token.Instance().Get() 查询数据库，验证令牌存在且未过期（SHA256 哈希在 manager 层自动处理）
  - 注入 — 将 user_id 和 token_type 存入 gin.Context，下游通过常量取值：

  // 使用示例
  r.Group("/admin").Use(middleware.AdminAuth())

  // handler 中读取
  userID := c.GetUint(middleware.CtxUserID)
  tokenType := c.GetString(middleware.CtxTokenType)

✻ Brewed for 47s
```

### 必要的常量定义

这里首先发现了一个题外的问题，token 管理器中的 `token_type` 应该使用常量存储了，比如这次做验权肯定会使用它，四处硬编码并不合理，这里先在 `token` 管理器内加上了 `token type` 常量：

目前就这一种 `token`，不过未来大概率会增加

```go
// internal\infra\token\token.go 文件

// 令牌类型标识
const (
	TypeAdminLogin = "admin_login" // 管理员登录令牌
)
```

### 两个验权中间件

1. `AdminAuth` 中间件：正常的验权中间件，它解析 token > 没有 token 提示需要登录，token 验证失败也需要登录，白说了，只要使用了该中间件，不登录就是不行的
2. `AdminAuthOptional` 中间件：可选的认证中间件，有 token 则解析并注入管理员信息，没有 token 也继续放行

这样设计的目的是：
1. 比如有的接口不登录也能访问，但是如果你登录了，我会响应更多数据。
2. 比如获取登录配置的接口，已登录的我就直接跳转到后台，不需要重复登录。
3. ......

### 向请求上下文注入更多信息

目前 AI 给的方案是：将登录用户的 `user_id` 和 `token_type` 存入 `gin.Context`，下游通过常量取值：

```go
userID := c.GetUint(middleware.CtxUserID)
tokenType := c.GetString(middleware.CtxTokenType)
```

我们的中间件名为：`AdminAuth`，明显是管理员专用的，这里直接让它在中间件里边检查数据库内的 `token` 的 `type` 为 `TypeAdminLogin`，然后直接将对应管理员的**整行数据**获取到，完整注入到上下文中。

### 使用 AdminAuthOptional 中间件

直接于 `获取登录配置接口（/admin/login-config）` 中，用上 `AdminAuthOptional` 中间件，这是可选登录中间件，如果用户传递了 token，并且认证成功，`Handler` 内就可以直接告诉用户不用登录了，前端跳转到后台首页。

### token 检查不完善

```go
// 校验令牌
tk, err := token.Instance().Get(c.Request.Context(), parts[1])
if err != nil || tk == nil {
    return nil, "身份认证令牌失效，请重新登录"
}
```

AI 错误的认为 `Get` 方法已经对 token 的有效期进行了检查，但实际上没有，额外加上有效期检查逻辑：

```go
// 校验令牌
tk, err := token.Instance().Get(c.Request.Context(), parts[1])
if err != nil || tk == nil || time.Now().After(tk.ExpiredAt) {
    return nil, "身份认证令牌失效，请重新登录"
}
```

这里现场于数据内修改 token 有效期字段进行了测试，确定 token 有效期检查逻辑无误、有效。

### 完整代码

我们能完美的描述需求和找到的问题，AI解决起来也更加得心应手，比如从 header 提取 token、数据 `Set(常量 key, value)` 到请求上下文等都没有问题，中间件的代码如下：

```go
// internal\middleware\admin_auth.go

package middleware

import (
	"net/http"
	"strings"
	"time"

	"ai-go-mall/internal/infra/token"
	"ai-go-mall/internal/model"
	repoAdmin "ai-go-mall/internal/repository/admin"
	"ai-go-mall/internal/response"

	"github.com/gin-gonic/gin"
)

const (
	// 上下文中存储管理员信息的键
	CtxAdminKey = "admin"
)

// 标识常量
const (
	FlagNeedLogin = "need_login" // 需要登录
	FlagLoggedIn  = "logged_in"  // 已经登录
)

// AdminAuth 管理员认证中间件，未登录时阻断请求
func AdminAuth() gin.HandlerFunc {
	return func(c *gin.Context) {
		admin, msg := extractAdmin(c)
		if admin != nil {
			c.Set(CtxAdminKey, admin)
			c.Next()
		} else {
			response.Fail(c,
				response.WithMessage(msg),
				response.WithCode(http.StatusUnauthorized),
				response.WithData(gin.H{"type": FlagNeedLogin}),
			)
			c.Abort()
		}
	}
}

// AdminAuthOptional 可选管理员认证中间件，有 token 则注入管理员信息，没有也放行
func AdminAuthOptional() gin.HandlerFunc {
	return func(c *gin.Context) {
		if admin, _ := extractAdmin(c); admin != nil {
			c.Set(CtxAdminKey, admin)
		}
		c.Next()
	}
}

// GetAdmin 从上下文中取出管理员信息，未登录时返回 nil
func GetAdmin(c *gin.Context) *model.Admin {
	admin, ok := c.Get(CtxAdminKey)
	if !ok {
		return nil
	}
	return admin.(*model.Admin)
}

// extractAdmin 提取并验证 token，返回 (管理员信息, 错误消息)
func extractAdmin(c *gin.Context) (*model.Admin, string) {
	// 提取请求 token
	authHeader := c.GetHeader("authorization")

	parts := strings.SplitN(authHeader, " ", 2)
	if len(parts) != 2 || !strings.EqualFold(parts[0], "Bearer") {
		return nil, "请先登录"
	}

	// 校验令牌
	tk, err := token.Instance().Get(c.Request.Context(), parts[1])
	if err != nil || tk == nil || time.Now().After(tk.ExpiredAt) {
		return nil, "身份认证令牌失效，请重新登录"
	}

	// 仅允许管理员登录类型的令牌
	if tk.Type != token.TypeAdminLogin {
		return nil, "请重新登录"
	}

	// 查询管理员信息
	adminRepo := repoAdmin.NewAdminRepository()
	admin, err := adminRepo.Get(c, tk.UserID)
	if err != nil {
		return nil, "请重新登录"
	}
	return admin, ""
}
```