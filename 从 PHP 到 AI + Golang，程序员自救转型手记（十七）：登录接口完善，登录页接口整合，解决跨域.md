# 从 PHP 到 AI + Golang，程序员自救转型手记（十七）：登录接口完善，登录页接口整合，解决跨域

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “目录结构更新、完善 token 系统”，本期将完成：登录接口完善，登录页接口整合，解决跨域

# 登录接口完善

### 为什么使用有状态的 DB token 而不是 jwt？

1. jwt 无法即时作废是最大硬伤，就算发现泄露也无法立即吊销，**风险致命，仅这一点，它就不适合后台管理系统**
2. jwt 信息只是 Base64 编码，不是加密，任何人解码就能看到里面用户 ID 等数据
3. 续期逻辑别扭，要么滑动续期、要么双 Token（Access+Refresh）方案增加复杂度
4. 密钥管理不当一旦泄露，攻击者可伪造任意用户令牌

**来个服务器二次验证，额外维护黑名单？那就失去无状态的特性了，何必jwt？**

### 登录接口完善规划

1. @internal/service/admin/admin_service.go 中的 LoginRequest 增加 remember（记住我）字段
2. 管理员登录成功使用 `github.com/google/uuid` 的 `NewV7` 方法生成 `token` 值，并使用 token 管理器（@internal\infra\token\token.go）创建 token
3. remember 为真时，token 有效期为 30 天，否则 token 有效期为 3 天（未来考虑增加配置项，可自定义修改 token 有效期）

将规划发给 cc，这次的需求简单明确，实现的核心代码如下：

1. 增加了 `github.com/google/uuid` 依赖
2. `internal\service\admin\admin_service.go` 文件内，管理员登录成功后增加了生成 token 的代码：

```go
// 增加导入
import (
    "time"

	"ai-go-mall/internal/infra/token"
    
    "github.com/google/uuid"
)

// 登录方法中，登录成功后增加生成令牌的代码
tokenStr := uuid.Must(uuid.NewV7()).String()

// 计算过期时间：记住登录为 30 天，否则为 3 天
expiredAt := time.Now().Add(3 * 24 * time.Hour)
if req.Remember {
    expiredAt = time.Now().Add(30 * 24 * time.Hour)
}

// 创建令牌
tk := &model.Token{
    Token:     tokenStr,
    Type:      "admin_login",
    UserID:    admin.ID,
    CreatedAt: time.Now(),
    ExpiredAt: expiredAt,
}
if err := token.Instance().Create(c.Request.Context(), tk); err != nil {
    return nil, errors.New("保存令牌失败")
}
```

# 登录页接口整合

### 完成登录页接口的整合
1. 于 `web/src/api/admin` 文件夹合理位置建立登录页的登录请求函数
2. `@web/src/views/admin/login.vue` 完成登录请求的发送
3. 登录请求响应结构体是 `@internal/service/admin/admin_service.go` 的 `LoginResponse`，请求后将响应的数据填充到 `adminInfo` 状态商店
4. 若登录请求成功，跳转至 `/admin`

AI 工作时，顺便手动操作数据库增加一个管理员以便后续测试，密码密文可以让另外的 AI 生成，提示词如：golang 中，验证密码使用 bcrypt.CompareHashAndPassword，帮我生成一个密码为 "123456" 的密文

```bash
登录接口整合已完成，以下是变更总结：

1. 新增 web/src/api/admin/login.ts — 登录 API 请求函数

- 定义了 LoginRequest（请求参数）、LoginResponse（响应数据）两个 TypeScript 接口
- adminLogin() 使用 URLSearchParams 以 application/x-www-form-urlencoded 格式提交，与后端 Gin form 标签绑定方式匹配
- 请求路径：POST /admin/login

2. 更新 web/src/views/admin/login.vue — 接入真实登录

onSubmit 函数变更：

// 之前（占位代码）
await new Promise((resolve) => setTimeout(resolve, 1500))

// 之后（真实 API 调用）
const res = await adminLogin(loginForm)
useAdminInfo().dataFill(res.data.data, false)
router.push('/admin')

完整流程：
1. 表单校验通过后，调用 adminLogin(loginForm) 发送登录请求
2. 请求成功 → 通过 dataFill(res.data.data, false) 将后端返回的管理员信息 + token 全部填充到 adminInfo Pinia store（false 表示不排除任何字段，token 也会被持久化）
3. 填充完成后 → router.push('/admin') 跳转到后台首页
4. 请求失败（code ≠ 0 或网络错误）→ 由 request.ts 拦截器自动弹出 ElMessage.error，无需额外处理
```

首先还是检查文件加减和目录结构调整是否正确，后经过人工 `review` 后，整理出第二轮需要优化的地方如下：
1. `web/src/api/admin/login.ts` 改为 `index.ts`，后续将后台登录、首页接口等较为公共的请求函数都放入其中，不应该使用特定的 `login` 命名
2. 删除 `LoginRequest`（请求参数）、`LoginResponse`（响应数据）两个 `TypeScript` 接口（PS：前者由 login.vue 提供，本身就带有类型定义，后者如果请求函数需要复用，保留的价值才能对等）
3. 调用 API 请求函数时，使用 then 的语法，而不是 await
4. `useAdminInfo` 在顶层合适的位置提前定义变量，而不是直接调用它的 `dataFill`
5. 登录失败由 `request` 拦截器自动提示，就直接不要 `catch` 了

```bash
全部改完，总结变更：

web/src/api/admin/login.ts → web/src/api/admin/index.ts
- 删除 LoginRequest、LoginResponse 两个接口，参数类型内联为 { username: string; password: string; remember: boolean }
- 文件名改为 index.ts，后续后台公共请求（首页等）都放入此文件

login.vue 变更
- import 路径改为 /@/api/admin
- router 和 adminInfo 在顶层就近定义，不再在 onSubmit 内部调用 useAdminInfo()
- 使用 .then().finally() 链式语法替代 await
- 移除 catch，失败由 request 拦截器统一提示
```

目检代码OK，测试发现了一个计划之外的情况，浏览器提示跨域，直接问不收费的 AI，golang 里边解决跨域问题什么库最好用，star 最多。

### 解决跨域

> PS：此时工作区已经有 登录页接口整合 尚未提交的代码，作者一般喜欢保持工作区干净，这里使用 git stash 保存工作现场（先 add 再 stash 可以将新增文件也储藏起来，否则只是修改部分）

最终决定整合 [cors](https://github.com/gin-contrib/cors) 库，提示词如下：

现在解决跨域问题：
1. 整合 https://github.com/gin-contrib/cors 跨域库（仍旧是直接提供 github 仓库地址，这是必须的，具体原因可以参见第八期的热重载配置一章）
2. 新建一个配置文件，用于配置允许跨域的域名列表，默认允许所有请求头、请求方法等只对域名做限制，预检请求最大缓存时间 24 小时
3. 在合适位置单独建立跨域中间件文件，其中使用以上跨域库结合跨域配置文件实现 CORS 处理，中间件直接于 api/main.go 注册

`review` 时，发现设定允许的请求方法时，AI 将 `GET, POST, PUT, DELETE......` 逐一列出，我们的要求是默认允许所有请求方式，先问 AI：允许所有请求方法能否使用 * 代替？

其次，目录结构有改动，增加了 `config/cors.yaml` 和 `internal\middleware\cors.go` 都是合理的。

接下来整理出了第二轮需要优化的地方：
1. 将 `AllowMethod、AllowHeaders、MaxAge` 等配置项，都放入配置文件内（原需求较为模糊，它只放了域名列表）
2. 默认只允许 `localhost` 和 `127.0.0.1` 而不是 `*`
3. 允许域名列表只需要填写域名即可，无需携带协议，中间件中实现：同时允许该域名的 `http` 和 `https`

还有一点比较重要的：默认值设置到 `config/cors.yaml`，而不是如果 `config/cors.yaml` 为空则硬编码指定的那些默认值

这一点算是 AI 造的坑，它对默认值的理解是硬编码到代码中，而我是希望默认值配置到 `config/cors.yaml` 中即可，如果我修改了 `config/cors.yaml` **默认值**也会随之修改才行，它是这样写的：

```go
// 应用默认值
methods := cfg.Methods
if len(methods) == 0 {
    methods = []string{"GET", "POST", "PUT", "PATCH", "DELETE", "HEAD", "OPTIONS"}
}
headers := cfg.Headers
if len(headers) == 0 {
    headers = []string{"*"}
}

// ...
```

这意味着我清空了配置文件中的配置，它反而又会应用上一些默认值，而我希望的是清空了配置就不再允许该类请求 Methods 和 Headers。

这次意外刚好在一定程度上证明了 `review` 的重要性，本次虽然是跨域配置，但一不小心就可能是重要数据泄露或权限意外提升；跨域中间件和实现完整代码放在文末。

### 继续测试登录页接口整合情况

> PS：在解决跨域问题之前，我们使用 `git stash` 保存了工作现场，先恢复一下，执行 `git stash pop`

> PS：这里测试还是跨域，原来 gin-contrib/cors 库配置的域名，还得加上端口号，这里又退回去让 AI 改为允许一个域名跨域，就允许它的所有端口。

经人工测试，发现了两个问题：

1. 登录请求参数，使用的是 form 绑定：

```go
// LoginRequest 登录请求参数
type LoginRequest struct {
    // 使用了 form
	Username string `form:"username" binding:"required"`
	Password string `form:"password" binding:"required"`
	Remember bool   `form:"remember"`
}
```

请求的 `Content-Type` 为 `application/x-www-form-urlencoded` 或 `multipart/form-data` 使用才使用表单绑定。而我们是 `application/json`，所以需要改为 `json` 绑定才更正规（使用 `ShouldBind` 时会自动选择绑定引擎，所以错误的 tag 目前没有影响值绑定），修改如下：

```go
// LoginRequest 登录请求参数
type LoginRequest struct {
    // 改为 json
	Username string `json:"username" binding:"required"`
	Password string `json:"password" binding:"required"`
	Remember bool   `json:"remember"`
}
```

2. 服务端输出的字段是 `PascalCase` 的命名方式， 如 `Username、CreatedAt`，这些字段名和我们前端状态商店的字段名无法一一对应，且：几乎所有主流 API 都不会使用 `PascalCase`（大写开头）作为 JSON 字段命名，行业标准是 `snake_case`，这里暂时还是不建立 `DTO`，直接让 `AI` 修改模型的 `tag`。

```go
// internal\model\admin.go 文件，所有字段都加上了 json tag，且敏感字段忽略
type Admin struct {
	ID           uint           `gorm:"comment:ID;primarykey;autoIncrement" json:"id"`
	Username     string         `gorm:"comment:用户名;type:varchar(64)" json:"username"`
	Nickname     string         `gorm:"comment:昵称;type:varchar(64)" json:"nickname"`
	Avatar       string         `gorm:"comment:头像;type:varchar(255)" json:"avatar"`
	Email        string         `gorm:"comment:邮箱;type:varchar(128)" json:"email"`
	Mobile       string         `gorm:"comment:手机号;type:varchar(16)" json:"mobile"`
	LoginFailure uint           `gorm:"comment:连续登录失败次数;not null;default:0" json:"-"`
	LastLoginAt  time.Time      `gorm:"comment:上次登录时间" json:"last_login_at"`
	LastLoginIP  string         `gorm:"comment:上次登录IP;type:varchar(64)" json:"last_login_ip"`
	Password     string         `gorm:"comment:密码;type:varchar(255)" json:"-"`
	Bio          string         `gorm:"comment:个人简介;type:varchar(255)" json:"bio"`
	Status       string         `gorm:"comment:状态:enable=启用,disable=禁用;type:varchar(64)" json:"status"`
	UpdatedAt    time.Time      `gorm:"comment:更新时间" json:"updated_at"`
	CreatedAt    time.Time      `gorm:"comment:创建时间" json:"created_at"`
	DeletedAt    gorm.DeletedAt `gorm:"comment:删除时间;index" json:"-"`
}
```

3. 状态商店 `@src\stores\adminInfo.ts` 的 `dataFill` 方法批量填充状态时，会填充 `state` 中没有的字段，需要让 AI 进行过滤：

```bash
你的 AdminInfo 接口定义了 8 个字段：id、username、nickname、avatar、last_login_at、last_login_ip、token、super。

this.$patch(state) 的问题在于：Pinia 的 $patch 会将传入对象的所有（可枚举自有）属性合并到 store 状态中。如果后端返回了 AdminInfo 中不存在的字段（比如 email、phone
等），它们会被悄无声息地写入 store。

解决方案：在 $patch 之前，用 Object.keys(this.$state) 做白名单过滤，只保留 store 中已定义的字段：

const validKeys = Object.keys(this.$state)
const filtered: Record<string, any> = {}
Object.keys(state).forEach((key) => {
	if (validKeys.includes(key) && !exclude.includes(key)) {
		filtered[key] = state[key]
	}
})
```

以上方案看起来不错，不过我们项目已经引入了 `lodash-es`，所以让 AI 不要手搓，查找 `lodash-es` 中的可用函数：

```bash
两种写法

写法一：_.pick + _.omit 链式

import { pick, omit } from 'lodash-es'

const filtered = omit(pick(state, validKeys), exclude)

写法二：_.difference + _.pick（一次 pick，更清晰）

import { pick, difference } from 'lodash-es'

const allowedKeys = difference(validKeys, exclude)
const filtered = pick(state, allowedKeys)

推荐写法二，一次 pick 同时完成白名单过滤和排除，比手写 forEach + includes 更简洁，也比我之前的写法更易读。需要我更新代码吗？
```

最终改动大概是这样的：

```ts
// 白名单过滤：只保留 $state 中已定义的字段，并排除指定字段
// exclude 是开发者调用函数时传入的排除字段列表
const allowedKeys = difference(Object.keys(this.$state), exclude)
this.$patch(pick(state, allowedKeys))
```