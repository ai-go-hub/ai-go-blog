# 从 PHP 到 AI + Golang，程序员自救转型手记（十）：交叉验证AI模型的工作成果

> 人在造出汽车之时就明白，汽车一定是比人跑得更快的，其他很多机器也一样，造出他们的动机就是为了比人干的更好；AI亦是如此，AI替代部分人类的工作，这没什么奇怪的。

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “管理员登录接口实现”，本期将完成：交叉验证AI模型的工作成果

终于是第十期了，ai-go-mall 项目至今，已经使用 cc 实现了 `配置系统（带自动发现、环境配置文件、环境变量覆盖）、数据库连接（带读写分离，支持链式+函数式选项自定义 db）、模型自动发现和迁移、四层业务架构的基类实现、自动发现和注册路由、统一响应结构和函数（也支持链式调用和函数式选项）` 等一大堆的功能，满怀期待的让 cc（换了个模型）扫描了一下现在的代码，最后确定有以下几个问题值得修改：

# 一、基仓储方法改名

基仓储的 Row / Rows 方法建议改名，理由如下：

当前 `Row(c, id)` 和 `Rows(c, filters...)` 这组命名虽然短，但不太符合 Go **业务层**常见习惯。
Row / Rows 更像 SQL 行的概念，偏数据库底层。但在 Handler / Service 层调用时：`h.Row(c)`、`s.Row(c, id)`，语义略显别扭。

最终决定换为更能表达业务意图且和 `database/sql` 不一样的：`Get` 和 `List`

类似换名的工作交给 AI 只是一句话的事，很少出问题，不再赘述。

# 二、字段命名规范统一

`LastLoginIp` 建议改为 `LastLoginIP`

1. 当前已经有 `ID`，使用 `LastLoginIP` 会更加统一一些
2. 更符合 Go 社区命名惯例，Go 社区对常见 initialism 有固定习惯，例如：

```bash
ID
URL
HTTP
IP
JSON
SQL
API
ClientIP
```

# 三、Go 社区明确不建议字段 getter 写成 GetXxx

向另外几个 AI 模型确定了一下，确实这样的，好在现在有 cc，批量改也很简单：

1. `internal\repository\base.go` 中的 `GetDB` 改名为 `DB`；而且其中有一个自定义 db 的方法名是 `DB`，这个反而是社区建议的 getter 命名方式，改名为 `WithDB`
2. `internal\database\database.go` 中的 `GetDB` 改名为 `DB`。
3. `config\config.go` 中的 `GetViper` 改名为 `Viper`；注意此文件中的 `Get` 方法并不典型，且官方标准包也有这样的用法，无需为了特意 “getter 不写 GetXxx” 而强行修改。

# 四、文件改名

建议将 `internal/service/admin/admin_service.go` 改名为 `internal/service/admin/service.go`
建议将 `internal/repository/admin/admin_repository.go` 改名为 `internal/repository/admin/repository.go`

即去掉文件名的 `admin` 前缀，因为包名已经含有 `admin` 了

优点

- 可以避免 `stutter` 的问题，减少文件名上的重复
- 和类型名一致
- 包内更简洁

缺点

- 如果以后一个包内有多个服务文件，例如 auth_service.go、group_service.go，service.go 可能不够明确

经过研究，只需要改为：`internal/service/admin/admin.go` 和 `internal/repository/admin/admin.go` 就可以了，后续文件增加目录结构可以自然的扩展为：

```go
internal/service/admin/
  ├── admin.go
  ├── group.go
  └── rule.go

internal/repository/admin/
├── admin.go
├── group.go
└── rule.go

internal/handler/admin/
├── admin.go
├── group.go
└── rule.go
```

# 五、IsInit 改为 Initialized

因为 `IsInit` 显口语化，以及 `config.Initialized` 更加自然。

# 小技巧分享
发现一个小细节，cc 可以读取到 git 的暂存区/工作区，比如你可以直接让它帮你：`检查暂存区的修改` / `看看我暂存区的修改是否合理` 等