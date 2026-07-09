# 从 PHP 到 AI + Golang，程序员自救转型手记（二十七）：管理员登录验证码开关

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “icon 组件封装”，本期将完成：管理员登录验证码开关

# 管理员登录验证码开关

管理员登录人机验证码有时候需要关闭，比如为了内网开发环境下的测试方便等，这里直接将需求描述给 CC：

```bash
@config/captcha.yaml 增加管理员登录的人机验证码开关配置项，前端登录前使用 GET 请求获取该配置，根据配置确定点选验证码是否弹出，服务端登录接口根据该配置决定是否进行验证码效验
```

● 所有修改已完成，编译通过。以下是变更总结：

1. `config/captcha.yaml` — 新增 `enabled` 配置项

```yaml
captcha:
    enabled: true   # ← 新增，是否启用人机验证码
```

2. `internal/infra/config/config.go` — `CaptchaConfig` 增加 `Enabled` 字段

```go
type CaptchaConfig struct {
    Enabled         bool         `mapstructure:"enabled"`  // ← 新增
    Elements        []string     `mapstructure:"elements"`
    // ...
}
```

3. `internal/service/admin/admin.go` — 登录时根据配置决定是否校验验证码

```go
if config.Get().Captcha.Enabled {
    if ok, err := captcha.Check(req.Captcha, true); !ok {
        return nil, fmt.Errorf("验证码错误：%w", err)
    }
}
```

4. `internal/handler/common/captcha.go` — 新增 `GET /common/captcha/config` 接口

返回 `{"code":200, "data":{"enabled":true}}`，供前端查询验证码开关状态。

5. `web/src/api/common.ts` — 新增 `getCaptchaConfig()` API 函数

6. `web/src/views/admin/login.vue` — 登录流程改造

- `onMounted` 时调用 GET `/common/captcha/config` 获取配置
  - `onSubmit` 时根据 `captchaEnabled` 决定：
    - 启用：弹出点选验证码（原流程不变）
    - 关闭：跳过验证码，直接提交登录（captcha 字段给空值）
  - 获取配置失败时默认启用验证码（安全兜底）

使用方式：将 `config/captcha.yaml` 中的 `enabled` 设为 `false` 即可关闭管理员登录的人机验证码。

---------------------------------------------------------

先简单看了下输出，确实加了配置项，叫做 `captcha.enabled`，但我们的 `config/captcha.yaml`，是全局验证码配置文件，而本次增加的开关只是管理员登录接口的验证码开关，所以这个配置名有点太广泛了，这里的规划明显就需要全局观了，未来可能还会增加会员登录接口的验证码开关，或者商户登录接口的，当前又是在 `config/captcha.yaml` 文件，最合理的应该是建立一个类似 `开关` 的数组，其内再存放各个模块的具体开关状态，我连变量名都懒得取，将这段思路发给 AI 即可，这波稳了一手，先开 `plan mode`。

```bash
  # 各接口人机验证码开关
  switches:
    # 管理员登录接口
    admin_login: true
```

AI 取出来一个 `switches.admin_login`，看来它理解了我的意思，下面先检查目录结构改动，然后对代码逐行 `review`：

### 验证码配置获取接口位置错误

获取验证码配置的接口，它放在了单独的公共接口之中，而我希望的是放在 `admin/login-cofig` 之类的独立接口中，登录页请求一下就行了。

```go
// internal\handler\common\captcha.go 文件

// Config 返回验证码配置（供前端判断是否启用人机验证）
func (h *CaptchaHandler) Config(c *gin.Context) {
	response.Success(c, response.WithData(gin.H{
		"admin_login": config.Get().Captcha.Switches.AdminLogin,
	}))
}
```

AI 调整后：

```go
// internal\handler\admin\admin.go 文件

// GetLoginConfig 返回管理员登录页配置
func (h *AdminHandler) GetLoginConfig(c *gin.Context) {
	response.Success(c, response.WithData(gin.H{
		"admin_login": config.Get().Captcha.Switches.AdminLogin,
	}))
}

// 路由注册增加一行

group.GET("/login-config", h.GetLoginConfig)
```

### 缺少合理封装

前端需要根据验证码开关状态确定是否弹出人机验证码，AI是这样写的：

```ts
if (captchaEnabled.value) {
    clickCaptcha((data) => {
        submitLoading.value = true
        adminLogin({ ...loginForm, captcha: data })
            .then((res) => {
                adminInfo.dataFill(res.data.data, false)
                router.push('/admin')
            })
            .finally(() => {
                submitLoading.value = false
            })
    })
} else {
    submitLoading.value = true
    adminLogin({ ...loginForm, captcha: { key: '', clicks: [], renderedWidth: 0, renderedHeight: 0 } })
        .then((res) => {
            adminInfo.dataFill(res.data.data, false)
            router.push('/admin')
        })
        .finally(() => {
            submitLoading.value = false
        })
}
```

很明显大部分代码可复用，冗余严重，改为：

```ts
const doLogin = (captcha?: ClickRequest) => {
    submitLoading.value = true
    adminLogin({ ...loginForm, captcha })
        .then((res) => {
            adminInfo.dataFill(res.data.data, false)
            router.push('/admin')
        })
        .finally(() => {
            submitLoading.value = false
        })
}

if (captchaEnabled.value) {
    clickCaptcha((data) => doLogin(data))
} else {
    doLogin()
}
```