# 从 PHP 到 AI + Golang，程序员自救转型手记（八）：设计管理员模型、热重载配置

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “建立 CLAUDE.md 文件、整理目录结构”，本期将完成：设计管理员模型、热重载配置

# 设计管理员模型/数据表

开始建表了，用了 `GORM`，按它的规范设计模型，然后使用自动迁移（`AutoMigrate`）就能完成建表，而且修改模型时，表结构也会随之自动更新。

我打算先实现一些后台功能后，再考虑 `反 XSS、跨域、节流` 等功能，我们之前已经实现了自动迁移的支持，新的模型只需要建立于 model 目录，并调用 `Register` 函数注册即可，系统启动时就会自动建表了。

### 自动迁移优化

目前注册自动迁移模型的语法是 `Register(&Admin{})`，一次只能注册单个模型；而多个相关联的模型是可以放入一个模型文件中的，比如 `admin.go` 模型文件中可以放 `Admin、AdminGroup、AdminRule` 至少三个模型定义，所以我们首先将 `Register` 函数改为支持接受任意数量的参数，实现 `Register(&Admin{}, &AdminGroup{}, &AdminRule{})` 这种写法。

直接要求 cc：将 @internal/model/model.go 中的 Register 函数，改为支持任意数量的参数，就像 db.AutoMigrate(&User{}, &Product{}, &Order{}) 一样

```bash
# 大模型是概率生成，不是固定查表，再加上采样策略、上下文、版本等影响，同问题输出都会有差异，以下 AI 输出、包括生成的代码仅供参考，可直接忽略，继续向下查看如何寻找问题和完成整改

现在我了解了当前的实现。修改 Register 函数支持可变参数：

● Update(internal\model\model.go)
  ⎿  Added 3 lines, removed 2 lines
       3  var models []any
       4  
       5  // 注册需要自动迁移的模型，各模型文件内可通过 init() 机制实现自动调用
       6 -func Register(m any) {                                                                                                                                                      
       7 -  models = append(models, m)                                                                                                                                                
       6 +// 支持一次注册多个模型，如 Register(&User{}, &Product{}, &Order{})                                                                                                         
       7 +func Register(models ...any) {                                                                                                                                              
       8 +  registered = append(registered, models...)                                                                                                                                
       9  }
      10  
      11  // 返回所有已注册的模型，供 database.Init 的 AutoMigrate 使用

● 还需要把变量名 models 改为 registered，避免与参数名冲突...
```

很简单的需求，总共没改几行，一眼就能确定新的代码没问题，核心是：

```go
func Register(models ...any) {
	registered = append(registered, models...)
}
```

### 管理员模型设计

当然也可以直接让 cc 参考 GORM 的文档设计模型：
1. 于 internal\model\admin.go 中以 GORM 的语法约定（https://gorm.io/zh_CN/docs/models.html），定义管理员模型，并注册自动迁移
2. 不要嵌入 `gorm.Model`，而是单独定义所有字段，因为它没有字段注释，且字段排序无法控制，请将 `ID` 字段放在第一位，`UpdatedAt、CreatedAt、DeletedAt` 字段按顺序放在最后
3. 每个字段都需要有 `中文` 注释（ID 字段除外，以 `ID` 作为注释即可）
4. 有规范有顺序的填写字段标签，可参考已有的模型定义，其中 `comment` 总是放在第一位，因为所有字段都会有 `comment`
5. 管理员模型的字段有 `ID, Username, Nickname, Avatar, Email, Mobile, LoginFailure, LastLoginAt, LastLoginIp, Password, Bio, Status, UpdatedAt, CreatedAt, DeletedAt`

> PS：
> 1. 作者这里以往设计过很多次管理员表，所以直接复制了之前的建表 SQL，并让其他 AI 将字段名整理出来，贴了进去，一般历往无设计时，可以让 AI 全新设计
> 2. 这份新建模型的提示词以后也可以用，目前是否全面合理也不一定，边用边完善吧

人工微调之后，设计很工整，其中 varchar 的 64 128 255 没啥特别的意义，就是为了好看 + 比较合理的长度，Status 字段没有使用 enum 类型，以便开发者可自行扩展其他状态值。

```go
type Admin struct {
	ID           uint           `gorm:"comment:ID;primarykey;autoIncrement"`
	Username     string         `gorm:"comment:用户名;type:varchar(64)"`
	Nickname     string         `gorm:"comment:昵称;type:varchar(64)"`
	Avatar       string         `gorm:"comment:头像;type:varchar(255)"`
	Email        string         `gorm:"comment:邮箱;type:varchar(128)"`
	Mobile       string         `gorm:"comment:手机号;type:varchar(16)"`
	LoginFailure uint           `gorm:"comment:连续登录失败次数"`
	LastLoginAt  time.Time      `gorm:"comment:上次登录时间"`
	LastLoginIp  string         `gorm:"comment:上次登录IP;type:varchar(64)"`
	Password     string         `gorm:"comment:密码;type:varchar(255)"`
	Bio          string         `gorm:"comment:个人简介;type:varchar(255)"`
	Status       string         `gorm:"comment:状态:enable=启用,disable=禁用;type:varchar(64)"`
	UpdatedAt    time.Time      `gorm:"comment:更新时间"`
	CreatedAt    time.Time      `gorm:"comment:创建时间"`
	DeletedAt    gorm.DeletedAt `gorm:"comment:删除时间;index"`
}
```

# 热重载配置

修改代码后总是需要重新执行 `go run cmd/api/main.go` 命令，配置好开发环境的代码更新自动热重载可以省很多事，golang 语言本身或标准库中没有自带类似功能，Gin 框架推荐使用 [Air](https://github.com/air-verse/air) 实现热重载。

其实安装和配置 Air 很简单，不过这里还是让 cc 来完成，直接告诉它：安装配置 Air（https://github.com/air-verse/air） 实现开发环境热重载功能

> PS：作者安装依赖是一般都会带上 github 或其他官方链接，这是很有必要的，主要避免 AI 安装了盗版依赖包，比如某个黑客写了个 https://github.com/hacker/air-true，并全网发布各类教程/宣发文章，此时若我们的提示词不带官方仓库链接，AI 可能会联网搜索 golang air 依赖，随即被网文所骗，且它还能正常运行，只是额外携带了病毒代码，半夜用你的电脑/服务器挖坑或当做肉鸡使用。

使用 cc 安装还是有好处的，生成的配置文件有根据项目目录结构适应，且写了注释；这里需要注意的是 air 并不是作为当前项目的依赖，而是 go install 到了 $GOPATH/bin，只 `.air.toml` 配置文件是在当前项目根目录，这里作者也做了研究，配置文件能不能换为 `YAML` 格式，避免项目中又多一种 `toml` 文件格式，可惜答案是不行。
