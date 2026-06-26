# 从 PHP 到 AI + Golang，程序员自救转型手记（九）：管理员登录接口实现

> 人在造出汽车之时就明白，汽车一定是比人跑得更快的，其他很多机器也一样，造出他们的动机就是为了比人干的更好；AI亦是如此，AI替代部分人类的工作，这没什么奇怪的。

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “设计管理员模型、热重载配置”，本期将完成：管理员登录接口实现

# 管理员登录接口实现

### 草稿版实现 + 完善 CLAUDE.md

先让 cc 把文件建上，并实现初版，之前我们的项目已经建立了 `CLAUDE.md`，并在其中描述了四层应用架构模式（Handler → Service → Repository → Model），本次提示词直接描述需求即可：

> PS：
> 1. 之前已经创建了 Admin 模型，这里告诉 AI 基于那个模型实现另外的三层
> 2. 直接指定出新控制器的位置，handler 内按规划是以用户身份建目录的，admin 为所有后台用户的目录，后续前台会员则会建立 user 目录，供应商建立 business 目录
> 3. 新的控制器需要直接嵌入基控制器并扩展管理员登录方法，所以额外提醒它嵌入，避免它建立全新的控制器
> 4. 管理员登录方法只是让它先写个大概，真正的登录实现需要一点一点重新整理，包括密码如何加密，如何验证、如何封装、入库等绝对必需人工一行一行 review
> 5. 先写登录接口，后续再写人机验证码和整合到登录接口，因为人机验证实现需要另外建表

给 cc 的提示词：基于 @internal/model/admin.go 实现 @internal/handler/admin/index.go 控制器、服务、仓储；控制器嵌入基控制器，并扩展一个管理员登录方法

```bash
现在我来创建完整的 admin 模块三层架构。首先创建仓储层...
```

好家伙，直接指定出了控制器的位置果然是正确的，因为没有指定具体位置的服务和仓储层，一个在 `@internal\repository\admin_repository.go`，一个在 `@internal\service\admin_service.go`，果然不会自动创建一层 admin 目录，这里要求它建立子目录同时完善 CLAUDE.md：

将 admin 的仓储、服务层也建立 admin 目录存放，并于 CLAUDE.md 合适位置整理并记录关于四层架构的新规划/说明：
未来项目功能扩张，早晚都是需要建子目录的，所以：
1. handler、repository、service 层直接以用户身份设立子目录，其中 admin 为所有后台用户的目录，后续前台会员则会建立 user 目录，供应商建立 business 目录等，脱离用户身份的仓储或服务，可以不放入子目录或放入 common 子目录
2. model 层无需建立子目录，因为：一个 model 文件里边可以放置多个模型，比如 `admin.go` 模型文件内可以同时放置 Admin、AdminGroup、AdminRule 等，且不同身份的用户，可能使用同一个模型，比如验证码、配置、省份地区数据模型

先来看看 cc 在 `CLAUDE.md` 中新增的内容：

```md
## 四层应用架构和泛型基类

......原有内容省略......

**目录结构规划**：

随着项目功能扩张，各层按以下规则组织子目录：

| 层级       | 子目录规则                                                                                             |
| ---------- | ------------------------------------------------------------------------------------------------------ |
| Handler    | 按用户身份设立子目录：`admin/`（后台用户/后台端）、`user/`（前台会员/会员端）、`business/`（供应商）等 |
| Service    | 按用户身份设立子目录，与 Handler 对应                                                                  |
| Repository | 按用户身份设立子目录，与 Handler 对应                                                                  |
| Model      | **不设子目录**，按业务模块组织文件（如 `admin.go`、`user.go`），一个文件可包含多个相关模型             |

**设计理由**：

1. **Handler/Service/Repository 按用户身份分目录**：不同身份的用户（后台管理员、前台会员、供应商）有各自独立的业务逻辑和 API 接口，分目录隔离更清晰
2. **脱离用户身份的通用服务**：可放入 `common/` 子目录或不设子目录（如验证码、配置、地区数据等跨身份服务）
3. **Model 不按身份分目录**：
   - 一个模型文件可包含多个相关模型（如 `admin.go` 内同时定义 `Admin`、`AdminGroup`、`AdminRule`）
   - 不同身份用户可能共用同一模型（如验证码、配置、省份数据等）

**当前结构示例**：

internal/
├── handler/
│   └── admin/           # 后台管理员相关接口
├── service/
│   └── admin/           # 后台管理员相关业务逻辑
├── repository/
│   └── admin/           # 后台管理员相关数据访问
└── model/
    ├── admin.go         # Admin, AdminGroup, AdminRule 等模型
    ├── user.go          # User, UserProfile 等模型
    └── common.go        # 验证码、配置等通用模型

```

### review 草稿并修改

#### internal\repository\admin\admin_repository.go

按规划嵌入了通用仓库实现，额外扩展了 `FindByUsername`、`FindByEmail`、`FindByMobile`、`UpdateLoginInfo`、`IncrementLoginFailure` 五个方法，发现以下问题：

1. 所有的查询语法使用了 GORM 的 Traditional API 语法，这与 `CLAUDE.md` 中的要求相悖，应该使用 Generics API
2. 后台属于内部系统，管理员没有邮箱、手机号绑定的功能，都是直接在后台设置的，所以目前无需实现通过邮箱、手机号登录，可直接去掉多余的 `FindByXXX` 方法（因为手机号可能被设置为 18888888888 等易被猜测到的手机号）

这一轮 AI 将 Traditional API 语法，改为了执行原生 SQL 语句，AI 响应如下：

```bash
1. `UpdateLoginInfo` 使用 `Exec` 执行原生 SQL 语句：`Exec("UPDATE admins SET last_login_ip = ?, last_login_at = NOW(), login_failure = 0 WHERE id = ?", loginIP, id).Error` 更新，理由是 `Generics API` 的 `Updates` 不支持 SQL 表达式和选择更新字段...

2. `IncrementLoginFailure` 也使用了 `Exec`，理由是 `Generics API` 的 `Updates` 不支持字段自增操作 `field = field + 1`...
```

这执行原生 SQL 也太 Low 了，还不如 Traditional API 呢，重新要求 cc：`UpdateLoginInfo` 改为使用基类的 Update，以 go 获取当前时间即可，无需使用 pgsql 的 NOW 函数：

这一轮好歹是勉强合格了，更新登录信息时是这样写的（登录时间、IP、重置失败次数）：

```go
// internal\repository\admin\admin_repository.go 的 UpdateLoginInfo 方法 struct 更新版

_, err := gorm.G[model.Admin](r.GetDB()).
    Where("id = ?", id).
    Select("last_login_ip", "last_login_at", "login_failure").
    Updates(c.Request.Context(), model.Admin{
        LastLoginIp: loginIP,
        LastLoginAt: time.Now(),
        LoginFailure: 0,
    })
```

如上，终于按要求改为了 Generics API，并使用了 Updates 传递一个 struct 更新（struct 默认跳过零值字段），所以又使用 `Select` 选择了更新字段，这种写法是没有问题的，只是略显冗余，强行使用了 `model.Admin` struct， 这里直接手工改为使用 map（map 不会跳过零值字段），如下，无需再写 `Select`，且还支持了原生 SQL 表达式（**谁说不支持的？**）：

```go
// internal\repository\admin\admin_repository.go 的 UpdateLoginInfo 方法 map 更新版
_, err := gorm.G[map[string]any](r.DB()).Table(model.Admin{}.TableName()).
	Where("id = ?", id).
	Updates(c.Request.Context(), map[string]any{
		"last_login_ip": loginIP,
		"last_login_at": gorm.Expr("NOW()"),
		"login_failure": 0,
	})
return err
```

`IncrementLoginFailure` 方法也是执行原生 SQL：

```go
func (r *AdminRepository) IncrementLoginFailure(c *gin.Context, id uint) error {
	return r.GetDB().WithContext(c.Request.Context()).
		Exec("UPDATE admins SET login_failure = login_failure + 1 WHERE id = ?", id).Error
}
```

手工修改为 `Generics API`：

```go
func (r *AdminRepository) IncrementLoginFailure(c *gin.Context, id uint) error {
	_, err := gorm.G[model.Admin](r.DB()).
		Where("id = ?", id).
		Update(c.Request.Context(), "login_failure", gorm.Expr("login_failure + ?", 1))
	return err
}
```

#### internal\service\admin\admin_service.go

原格式是 token 单独于管理员信息一个字段，而前端一般是 adminInfo 同 token 一起整体保存于一个缓存中，做修改如下：

```go
// LoginResponse 登录响应数据
type LoginResponse struct {
	Admin *model.Admin `json:"admin"`
	Token string       `json:"token,omitempty"`
}
```

修改后：

```go
// LoginResponse 登录响应数据
type LoginResponse struct {
	model.Admin
	Token string `json:"token,omitempty"`
}
```

#### internal\handler\admin\index.go

handler 层单独定义了一遍 `LoginRequest`，此结构体和 `service` 层的 `LoginRequest` 一样，所以可以删除掉，直接导入 `service` 层的 `LoginRequest` 使用，若后续业务复杂，两层使用的结构体不一样了，再修改即可：

```go
// LoginRequest 登录请求参数
type LoginRequest struct {
	Username string `json:"username" binding:"required"`
	Password string `json:"password" binding:"required"`
}

// 直接使用 service 层的 LoginRequest，以下代码可以大幅精简
result, err := h.adminSvc.Login(c, &svcAdmin.LoginRequest{
	Username: req.Username,
	Password: req.Password,
})

// 精简后
result, err := h.adminSvc.Login(c, &req)
```


#### internal\model\admin.go

`Admin.Password` 和 `Admin.LoginFailure` 字段不能被输出，在模型层，增加 `json:"-"`，后续如果有必要可以单独拆 DTO：

```go
type Admin struct {
	ID           uint           `gorm:"comment:ID;primarykey;autoIncrement"`
	Username     string         `gorm:"comment:用户名;type:varchar(64)"`
	Nickname     string         `gorm:"comment:昵称;type:varchar(64)"`
	Avatar       string         `gorm:"comment:头像;type:varchar(255)"`
	Email        string         `gorm:"comment:邮箱;type:varchar(128)"`
	Mobile       string         `gorm:"comment:手机号;type:varchar(16)"`
	LoginFailure uint           `gorm:"comment:连续登录失败次数" json:"-"`
	LastLoginAt  time.Time      `gorm:"comment:上次登录时间"`
	LastLoginIp  string         `gorm:"comment:上次登录IP;type:varchar(64)"`
	Password     string         `gorm:"comment:密码;type:varchar(255)" json:"-"`
	Bio          string         `gorm:"comment:个人简介;type:varchar(255)"`
	Status       string         `gorm:"comment:状态:enable=启用,disable=禁用;type:varchar(64)"`
	UpdatedAt    time.Time      `gorm:"comment:更新时间"`
	CreatedAt    time.Time      `gorm:"comment:创建时间"`
	DeletedAt    gorm.DeletedAt `gorm:"comment:删除时间;index"`
}
```