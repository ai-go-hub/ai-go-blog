# 从 PHP 到 AI + Golang，程序员自救转型手记（三十）：实现管理员注销接口、API 请求函数命名规范

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “token 过期清理、获取实例的方法名称规范统一”，本期将完成：实现管理员注销接口、API 请求函数命名规范

# 实现管理员注销接口

在迁移后台布局时，我们已经准备好了管理员注销按钮，和注销接口请求函数：

```ts
/**
 * 管理员注销请求
 */
export function adminLogout() {
    return request({
        url: '/admin/logout',
        method: 'POST',
    })
}
```

这里我们直接要求 CC：实现服务端 `/admin/logout` 接口

中途 AI 提供了几个选项：问我从哪里获取 token，提供了比如：
1. `logout` Handler 内单独从 `header` 解析出 `token`
2. 认证中间件解析到 `token` 后，额外注入 `token` 字段到请求上下文（之前只是认证，未记录 token）
3. ......

但我认为，我们之前已经注入了一个 `admin` 用于存储管理员信息，`token` 本来就是属于管理员信息的一部分，直接将 `token` 合并到 `admin` 中即可，聚合在一个对象里比两个独立 `context key` 更合理。

所以认证中间件内，将 `token` 解析出来后，合并放到 `admin` 模型中，最终注入到上下文中，然后注销时获取即可，不需要单独注入 `token`。

后续又考虑到 `model.Admin` 是 `GORM` 实体模型，映射 `admins` 表。`token` 是请求级别的临时数据，不属于 `admin` 实体，把它们绑在一起不合适，额外定义一个 `AdminSession struct` 作为 `extractAdmin` 的新返回值类型。

最终代码如下：

**internal\middleware\admin_auth.go：**

```go
// AdminSession 管理员会话信息，由认证中间件注入到请求上下文
type AdminSession struct {
	Admin *model.Admin
	Token string
}

// GetToken 从上下文中取出当前令牌字符串，未登录时返回空
func GetToken(c *gin.Context) string {
	session, ok := c.Get(CtxAdminKey)
	if !ok {
		return ""
	}
	return session.(*AdminSession).Token
}

// 其他略......
```

**internal\handler\admin\admin.go：**

```go
// Logout 管理员退出登录
func (h *AdminHandler) Logout(c *gin.Context) {
	if err := h.svc.Logout(c, middleware.GetToken(c)); err != nil {
		response.Fail(c)
		return
	}

	response.Success(c)
}

// 路由注册（此路由要求登录）
group.POST("/logout", middleware.AdminAuth(), h.Logout)
```

**internal\service\admin\admin.go：** 对应服务层调用 token 驱动删除 token 数据

```go
// Logout 注销当前管理员令牌
func (s *AdminService) Logout(c *gin.Context, tokenStr string) error {
	return token.Manager().Delete(c.Request.Context(), tokenStr)
}
```

### 意外 bug

后续对注销接口进行测试时，从数据库内直接删除掉 token 再发起注销，结果提示 `身份认证令牌失效，请重新登录`，这里逻辑不对，找到注销方法，加了以下几行代码：

```go
admin := middleware.GetAdmin(c)
if admin == nil {
    // 未登录，直接生成成功响应，其意图已自然完成
    // 不能执行任何 token 删除逻辑，因为管理员未认证
    response.Success(c)
    return
}
```

# API 请求函数命名规范

目前项目虽然没几个 `API` 请求函数（`web/src/api/` 内的函数），但是命名方式比较随意：

1. `adminLogin`、`adminLogout`
2. `getCaptchaConfig`、`checkClickCaptcha`
3. `postClearCache`

这里让 AI 帮我们统一一个命名规范：

豆包建议：`get` 和 `check` 开头的保持，`adminLogin` 这种没有前缀的加上一个 `do` 前缀，`post` 开头的也改为 `do` 前缀。

也听取了 `CC` 内的 `GLM-5.2` 的建议，差异在于：`adminLogin` 这种没有前缀也保持现状，因为我们项目只使用 `GET` 和 `POST` 两种请求方式，非 `GET` 自然就是 `POST`（兼容 cdn 等云服务，大多数 CDN/全站加速 服务对 PUT、DELETE 等兼容性差）；`post` 开头的则去掉前缀

最终我决定让 AI 帮我：
1. 去掉 `postClearCache` 的 `post` 前缀
2. `adminLogin` 改为 `login`，`adminLogout` 改为 `logout`（都在 `admin.ts` 文件以内，其实是无需前缀的）
3. `getCaptchaConfig` 改为 `getLoginConfig`

后续都以类似的方式进行命名：`get` 请求函数加前缀，普通的 `post` 无需前缀。