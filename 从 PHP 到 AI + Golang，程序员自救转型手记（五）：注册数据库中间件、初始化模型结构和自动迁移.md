# 从 PHP 到 AI + Golang，程序员自救转型手记（五）：注册数据库中间件、初始化模型结构和自动迁移

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “整合 Gin 框架、必学项 + 2”，本期将完成：注册数据库中间件、初始化模型结构和自动迁移

我下载了大量的 golang 项目源码来阅读，发现了一个问题，为啥它们的目录结构很多都不对，比如：

在 golang 中，标准的结构是 internal/model 目录存放数据库表结构体，和数据表一一对应，不放任何逻辑，不放 SQL，最多放一个表名函数，为啥现实是很多 golang 项目在 model 下放些乱七八糟的东西，比如初始化数据库连接？分页函数，创建/获取数据，更狠的是一个项目把全局通用的响应函数 `Success` 和 `Fail` 放在了 model 目录里边，这合理吗？

# 注册数据库中间件

在第三期我们已经完成了数据库的连接，第四期完成了 Gin 依赖的安装，所谓的数据库中间件，就是将连接数据库的 `gorm.Open` 方法返回的 `*gorm.DB` 注入到 `gin.Context` 中，以便 handler、repository 层可以随时通过 Context 获取到，经过第四期我们对 Gin 和 GORM 的学习，此时提示词可以写的更加得心应手，懵统而不懵逼，写明了概念而又随意，直接要求 cc：

在 database.go 中写一个 Gin 自定义中间件，用于将 *gorm.DB 注入到 gin.Context 中（`c.Set`），绑定当前请求的 context，独立于 `GetDB` 的裸连接，在 mian.go 注册该中间件，另外写一个 handler、repository 层快捷获取该 *gorm.DB 的方法（c.Get），新建的函数具有合理的命名和类型定义。

```bash
# 大模型是概率生成，不是固定查表，再加上采样策略、上下文、版本等影响，同问题输出都会有差异，以下 AI 输出、包括生成的代码仅供参考，可直接忽略，继续向下查看如何寻找问题和完成整改

编译通过。以下是改动说明：

  internal/database/database.go — 新增 3 处

  ┌──────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │                 内容                 │                                                       说明                                                        │
  ├──────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ ctxDBKey = "db"                      │ 上下文键常量，避免字符串硬编码                                                                                    │
  ├──────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ DBMiddleware() gin.HandlerFunc       │ Gin 中间件，通过 GetDB().WithContext(c.Request.Context()) 创建绑定请求 context 的 *gorm.DB 实例，注入 gin.Context │
  ├──────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ FromContext(c *gin.Context) *gorm.DB │ 使用 c.MustGet 从 gin.Context 取出 *gorm.DB，handler 和 repository 层均可调用                                     │
  └──────────────────────────────────────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  关键设计：中间件使用 db.WithContext(c.Request.Context()) 而非裸 GetDB()，确保每次请求的数据库操作携带请求级 context（超时、取消、trace 信息等），与全局 GetDB 返回的连接相互独立。

  cmd/api/main.go — 第 26 行

  r.Use(database.DBMiddleware())

  注册到 Gin 引擎，所有路由自动生效。

  调用示例

  // handler 层
  func GetUser(c *gin.Context) {
      db := database.FromContext(c)
      var user model.User
      db.First(&user, c.Param("id"))
      // ...
  }

  // repository 层
  func FindUserByID(c *gin.Context, id uint) (*model.User, error) {
      db := database.FromContext(c)
      var user model.User
      err := db.First(&user, id).Error
      return &user, err
  }

✻ Cogitated for 1m 8s
```

如上，一次性完成需求，而且代码质量很高，只需要微调一下细节，比如：函数命名为 `DBMiddleware`，我给手动调成了 `Middleware`，这个语境下，Db是多余的，ctxDBKey 常量避免硬编码此处感觉意义也不是很大，主要是这变量名太难看了，使用仅两处，直接去掉。

# 初始化模型结构和自动迁移

首先是应用启动总是执行自动迁移的必要性：

经过研究，应用启动时总是执行自动迁移是很有必要的，因为在 GORM 中，表结构变更也能自动完成同步，自动可以避免表结构不至于，以下是一些不适合使用自动迁移的场景：

1. 生产环境安全合规：一般会有独立手动迁移工具，避免应用启动时意外修改生产 schema，其实中小项目生产使用自动迁移影响不大，且非常方便，是否使用取决于项目需要。
2. 超大表：大表做表结构变更可能会很慢，拖慢启动速度。
3. 变更审计严格：比如金融、医疗行业，迁移一般是独立的工具，和第一点类似。

接下来是写给 cc 的提示词，也是功能规划（目前为止，作者的 cc 任然是最初始的状态，除了多了一些项目级授权）：
1. internal/model 目录存放数据库表结构体，和数据表一一对应，不放任何逻辑，不放 SQL，最多放一个表名函数，先默认建立一个测试用的 User 模型。
2. 利用 GORM 自带的 AutoMigrate 实现自动迁移，自动迁移逻辑代码放在 database.go 的 Init 函数中。
3. 新增模型时，不需要手动修改自动迁移代码，应用启动时可自动获取到新增的模型，并完成迁移（ps：就像配置系统一样，不要新增了配置文件就需要修改配置读取代码）。

```bash
当前代码已经实现了这三个需求的核心机制，我来逐一确认并完善。

  Read 3 files (ctrl+o to expand)

● 核心设计：init() + Register() 自注册模式，新增模型只需新增一个 .go 文件，不改任何已有代码。
● 略...
```

最后生成的代码如下：

```go
// internal\model\user.go

package model

import "time"

func init() {
	Register(&User{})
}

type User struct {
	ID        uint
	Name      string `gorm:"type:varchar(64);comment:用户名"`
	Email     string `gorm:"type:varchar(128);comment:邮箱"`
	Mobile    string `gorm:"type:varchar(16);comment:手机号"`
	CreatedAt time.Time
	UpdatedAt time.Time
}
```

```go
// internal\model\model.go

package model

var models []any

// 注册需要自动迁移的模型，各模型文件内可通过 init() 机制实现自动调用
func Register(m any) {
	models = append(models, m)
}

// All 返回所有已注册的模型，供 database.Init 的 AutoMigrate 使用
func All() []any {
	return models
}
```

```go
// internal\database\database.go 的 Init() 函数尾部增加

// 自动迁移所有已注册的模型
if models := model.All(); len(models) > 0 {
    if err := db.AutoMigrate(models...); err != nil {
        return fmt.Errorf("auto migrate: %w", err)
    }
}
```


`model` 支持子级文件夹？（考虑到一个数据表对应一个模型，未来模型可能会很多）经过研究：

1. 常规 Go 项目中，不强求一个模型文件只存放一个结构体，而是按内聚性组织，关联紧密的可以放一起：比如 User 和 UserProfile、UserSetting 可以放在同一个 user.go，因为它们围绕同一业务实体。
2. 子级文件夹的支持实现比较麻烦，因为注册新增模型的原理是利用 golang 的 init() 机制，至少需要导入包才能触发。（当然也可以更换实现方案，比如 go generate 扫描文件系统自动生成 imports.go，每次新增子级模型文件时，手动运行 go generate 命令）