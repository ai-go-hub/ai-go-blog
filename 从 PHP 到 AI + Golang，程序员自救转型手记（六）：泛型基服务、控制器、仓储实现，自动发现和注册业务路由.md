# 从 PHP 到 AI + Golang，程序员自救转型手记（六）：泛型基服务、控制器、仓储实现，自动发现和注册业务路由

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “注册数据库中间件、初始化模型结构和自动迁移”，本期将完成：泛型基服务、控制器、仓储实现，自动发现和注册业务路由

# 泛型基服务、控制器、仓储实现

由于对大型 golang 项目还不熟悉，直接同时让各大 AI：给我一套 golang WEB 项目的 Handler → Service → Repository → Model 的完整示例代码。

拿到示例代码后，作者逐行做了仔细阅读，并又问了一些问题，**加粗字体为结论**（以下问题只是作者在阅读代码时，对不理解的地方提出的疑问，读者可根据自己的实际情况提问和研究）：
1. 为什么 admin_repo.go，admin_service.go 都有后缀，而 model 不需要叫做 admin_model.go 呢？**因为不加后缀结构体重名、无法使用**
2. NewAdminHandler、NewAdminService、NewAdminRepository 是 golang 标准写法吗？**是的，golang 官方也这么写**
3. 包名已经含有 admin 了，`NewAdminRepository` 中是否可以省略 Admin？**可以省略**
4. **service 和 repository 层额外定义 interface，不直接使用结构体，以解耦、方便测试、方便扩展等**

理解了豆包、文心一言提供的示例代码之后，我开始让 cc 直接在项目中写一套基础的代码出来：
1. 写一个通用的 CRUD 实现，通用文件于 Handler → Service → Repository 对应目录建立，均以 base.go 命名
2. service 和 repository 层额外定义 interface，不直接使用结构体
3. 实现 Create、List、Edit、Delete 四个方法，其中 Edit 分为 GET 和 POST 两种请求方式，GET 为获取被修改的行数据，POST 为修改数据
4. 只使用 GET 和 POST 两种请求方式，不使用 PUT 和 DELETE 等请求方式（除 GET/POST 外的请求方式，在国内的 CDN/全站加速 兼容极差）

```bash
# 大模型是概率生成，不是固定查表，再加上采样策略、上下文、版本等影响，同问题输出都会有差异，以下 AI 输出、包括生成的代码仅供参考，可直接忽略，继续向下查看如何寻找问题和完成整改

全部完成！三个文件已创建并通过编译验证。以下是实现总结：

  --- 新建文件

  1. internal/repository/base.go — 数据访问层

  ┌──────┬────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │ 项目 │                                                内容                                                 │
  ├──────┼────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ 接口 │ CrudRepository[T any]                                                                              │
  ├──────┼────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ 实现 │ genericCrudRepository[T any]                                                                       │
  ├──────┼────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ 方法 │ Create、GetByID、List、Update、Delete                                                               │
  ├──────┼────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ 特点 │ List 使用 GORM Scopes 灵活组合筛选条件；Update 支持 map[string]any 部分更新；Delete 自动适配软删除      │
  └──────┴────────────────────────────────────────────────────────────────────────────────────────────────────┘

  2. internal/service/base.go — 业务逻辑层

  ┌──────┬─────────────────────────────────────────────────────────────────────┐
  │ 项目 │                                内容                                 │
  ├──────┼─────────────────────────────────────────────────────────────────────┤
  │ 接口 │ CrudService[T any]                                                  │
  ├──────┼─────────────────────────────────────────────────────────────────────┤
  │ 实现 │ genericCrudService[T any]                                           │
  ├──────┼─────────────────────────────────────────────────────────────────────┤
  │ 方法 │ Create、GetByID、List、Update、Delete                               │
  ├──────┼─────────────────────────────────────────────────────────────────────┤
  │ 特点 │ 基础实现直接委托 Repository；子类可覆写添加业务校验（如权限、去重） │
  └──────┴─────────────────────────────────────────────────────────────────────┘

  3. internal/handler/base.go — 控制器层

  ┌───────────────────────────┬──────────┬──────────────────────────────────┐
  │           路由            │   方法   │               说明               │
  ├───────────────────────────┼──────────┼──────────────────────────────────┤
  │ POST /{prefix}/create     │ Create   │ 新增记录                         │
  ├───────────────────────────┼──────────┼──────────────────────────────────┤
  │ GET /{prefix}/list        │ List     │ 分页列表（?page=1&page_size=10） │
  ├───────────────────────────┼──────────┼──────────────────────────────────┤
  │ GET /{prefix}/:id/edit    │ EditGet  │ 获取被编辑的行数据               │
  ├───────────────────────────┼──────────┼──────────────────────────────────┤
  │ POST /{prefix}/:id/edit   │ EditPost │ 执行修改                         │
  ├───────────────────────────┼──────────┼──────────────────────────────────┤
  │ POST /{prefix}/:id/delete │ Delete   │ 删除记录                         │
  └───────────────────────────┴──────────┴──────────────────────────────────┘

  统一响应格式：{"code": 0/-1, "message": "...", "data": {...}}

  后续使用方式示例

  // 在 cmd/api/main.go 中初始化
  userRepo := repository.NewCrudRepository[model.User](database.GetDB())
  userSvc  := service.NewCrudService[model.User](userRepo)
  userHandler := handler.NewCrudHandler[model.User](userSvc)
  userHandler.RegisterRoutes(r.Group("/user"))

  即可一键获得 /user 下的完整 CRUD 路由。

✻ Crunched for 5m 24s
```

首先检查目录结构的变更是否合理，有问题就先要求修改，目录结构没问题之后，才开始对代码 `review`：
1. AI 按要求创建了三个 base.go 文件，并没有创建多余的文件。
2. AI 于 `repository/base.go` 建立了多个数据操作方法，分别是 `Delete、List、Create、GetByID`，其中的 GetByID 方法用于根据主键 ID 查询单条记录，其命名规范算是 Go 社区惯例，但其余方法都是单个单词，所以这里特意将 `GetByID` 改名为了 `Row`，干净利落，但此时单行是 `Row`，多行的 `List` 当然应该改为 `Rows`，相当于和 database/sql 风格保持一致。
3. 此时 `Rows` 方法是支持分页的，传递了 page 和 page_size 等参数，而实际的项目中这里还得组装 where，order 等参数，AI 写的并不齐全，暂时干脆将 page 、page_size 也取消掉，保持基本实现即可，后续结合实际项目需求，再统一整理列表数据的查询方法。
4. AI 使用了 GORM `Traditional API` 的语法，这里要求它改写为更新的 `Generics API` 语法。
5. interface 和 struct 命名问题，AI 使用了 `genericCrudRepository`、`CrudRepository` 等名称，基础的 CRUD 实现是很方便扩展的，假设扩展了还叫 xxCrud 就不合适，而且作为基础接口和结构体，当然是越简短越好，而且这些定义本身就在 `base.go` 文件内，直接改为 `Repository(struct)` 和 `IRepository(interface)` 等，干净利落。
6. 控制器中 List 和 Edit 都被分为了两个处理方法，特别是 Edit，以 GET 和 POST 区分获取和编辑，一般需要又单独注册路由，所以干脆同 repo 层一样，以 Row 获取，以 Update 更新即可。

统一响应结构和函数单独要求 cc 修改，提出更多具体的要求：
1. AI 将 Response（统一响应）的结构体和方法放在了 handler/base.go 中，这是跨 handler 共享的 API 契约（比如错误处理中间件），需要拆分出来，并且加上 Time 字段，让前端方便的做缓存、去重、排序等。
2. AI 定义了多个响应函数，`Success(200)、BadRequest（400）、NotFound（404）、InternalError（500）` 等，它们使用不同的 HTTP 状态码，这种方案又与 CDN/全站加速 不兼容，它们对非 200 的状态码处理完全是一团乱麻，所以这里改为统一响应 200，然后以 code 字段体现业务状态，直接删除了 `BadRequest（400）、NotFound（404）` 函数，`InternalError` 暂留，增加了与 `Success` 对应的 `Fail` 快捷响应函数，统一使用 200 ok 状态码，code 则默认为 1
3. 统一响应方法支持同时支持链式调用和函数式选项（Functional Options）两种调用方法：
  - 链式调用：`response.New(c).Code(1).Message("测试").Send()`
  - 函数式选项：`response.Fail/Success(c, response.WithMessage("参数错误"), response.WithCode(1), response.WithData("test"))`（函数式选项时，后三个参数都可以省略）

将以上整理出来的发给 AI，让其重新生成，根据使用模型的不同，可能还需要对话几轮，最终生成的代码已开源：

# 自动发现和注册业务路由

路由系统设计：
1. 路由定义全部放入 @internal/router 目录
2. 实现子级目录路由定义的自动发现（`init + 空白导入` 的方案），建立专门的 @internal\router\registry\registry.go 文件来放置通用的 registry 函数，以及用来 `空白导入` 子目录路由文件触发 init
3. 预设 /admin、/user、/ 三组示例路由，里边注册一个测试用的 /ping、/admin/ping、/user/ping 路由即可，其中 /ping 的路由注册放在 @internal/router/index.go 中，无需建立子目录
4. @internal/router/router.go 实现 Setup 函数，后续于 main.go 内调用它即可完成全部业务路由的注册，Setup 中不要创建新的 r.Group，路由应该注册到那个分组，由各个子路由文件中的 init 决定

这里 AI 告诉我，我的提示词出了一点问题，`@internal\router\registry\registry.go` 里边放置 `空白导入` 的代码，会造成循环导入，因为空白导入的包内，使用了 `registry` 函数，遂将方案改为于最外层的 `router.go` 中完成空白导入。

```bash
全部完成，编译通过 ✅。最终结构：

  internal/router/
  ├── router.go              # Setup() + 空白导入自动发现
  ├── registry/
  │   └── registry.go        # Register(group, fn) + Setup(engine)
  ├── admin/
  │   └── admin.go           # GET /admin/ping → "admin pong"                                                                                    
  ├── user/
  │   └── user.go            # GET /user/ping → "user pong"
  └── index.go               # GET /ping → "index pong"

以后新增路由模块只需两步：新建子目录并 init() 调 registry.Register()，然后在 router.go 加一行 _ 导入即可。

✻ Cooked for 3m 46s
```

# 说几句题外话
1. 发给 AI 的提示中：`AI 于 xxx 建立了错误的文件，这里要求它改为 xxx` 这种第三人称的视角也是能直接用的，它能看懂，包括错别字，少量的模糊不清都没关系。
2. 语言初学者写提示词时，往往会不确定自己对不对，可以直接让 AI 确定是否合理，如果合理则立即实现，否则取消并给出解释。
2. 至此项目架构与地基已经完成，后续主要是业务面的东西，设计数据表，登录注册，CRUD 啥的，那些作者就很熟悉了，历时不到两个礼拜，终于回到了以往的 “舒适圈”。

今日产生的完整代码已经开源于：[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)