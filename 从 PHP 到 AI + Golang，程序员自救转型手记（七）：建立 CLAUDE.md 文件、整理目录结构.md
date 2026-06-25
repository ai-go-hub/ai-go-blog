# 从 PHP 到 AI + Golang，程序员自救转型手记（七）：建立 CLAUDE.md 文件、整理目录结构

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “泛型基服务、控制器、仓储实现，自动发现和注册业务路由”，本期将完成：建立 CLAUDE.md 文件、整理目录结构

# 建立 CLAUDE.md 文件

这个文件根据 CLAUDE 官方的定义，应当在 Claude 第二次犯同样的错误，或者当某个约定给它讲过两次以上时，可以将这些内容写入 CLAUDE.md，当然也可以提前讲清项目定位、目录结构，以便 CLAUDE 更好的工作。

ai-go-mall 项目至今，已经使用 cc 实现了 `配置系统（带自动发现、环境配置文件、环境变量覆盖）、数据库连接（带读写分离，支持链式+函数式选项自定义 db）、模型自动发现和迁移、四层业务架构的基类实现、自动发现和注册路由、统一响应结构和函数（也支持链式调用和函数式选项）` 等一大堆的功能，却连个 CLAUDE.md 都没有，足以见得 cc 的强大，也可能说明了当开发者本身技术够硬时，这些都是花架子，无需过于纠结（因为总有小伙伴纠结这东西咋写，实际上就是随便写，没有固定格式，不要被网上卖课的哄骗（免费的可以看），大不了你多调几次，调好了能少几轮对话，或者多节约一些 token）。

我们直接使用 `/init` 命令，让 cc 先自己做一份 `CLAUDE.md` 出来，然后第一件事就是删除多余内容，因为此文件将作为系统级上下文融入每一次对话中，而 AI 很多时候并不需要特别细致的解释即可很好的完成工作，最终整理出来的 `CLAUDE.md` 如下：

> PS:
> 其实里边还有很多都是特意改了给人看的，让初次上手的开发者也能快速了解项目总体情况；我个人认为目前最值得写入的内容：
> 1. 尽量使用 GORM 的 Generics API 而不是 Traditional API
> 2. 只使用 GET 和 POST 请求方式：大多数 CDN/全站加速 服务对 PUT、DELETE 兼容性差
> 3. 当我询问某方案是否合理时，请先根据社区惯例判断是否合理，合理直接实现，不合理取消实现并解释原因

```md
# CLAUDE.md

本文件为 Claude Code 在当前代码库中工作时提供指导。

## 项目概述

爱购商城（ai-go-mall），技术栈：Go 1.25 + Gin + GORM + PostgreSQL。

## 回答偏好
- 回复使用中文
- 遇到有多种实现方案时，列出选项让我选择，而不是直接选一种
- 当我询问某方案是否合理时，请先根据社区惯例判断是否合理，合理直接实现，不合理取消实现并解释原因（社区惯例指对应技术栈的社区，如 golang 开源社区，Gin 开源社区，开源高星仓库，官方文档，权威 blog 等）

## 泛型基类体系

核心应用架构为：泛型驱动的四层架构模式，请求 → Handler（控制器）→ Service（业务逻辑）→ Repository（数据访问）→ Model（数据模型）

前三层都有**泛型基础实现**，具体模块通过**组合（嵌入）**复用：

| 层级       | 泛型基类        | 接口             | 文件                          |
| ---------- | --------------- | ---------------- | ----------------------------- |
| Repository | `Repository[T]` | `IRepository[T]` | `internal/repository/base.go` |
| Service    | `Service[T]`    | `IService[T]`    | `internal/service/base.go`    |
| Handler    | `Handler[T]`    | 无               | `internal/handler/base.go`    |

**扩展模式**（以 User 为例）：

- `UserRepository` 嵌入 `*Repository[model.User]`，可访问 `GetDB()` 编写自定义查询
- `UserService` 嵌入 `IService[model.User]` 并持有 `*UserRepository`，可覆盖业务逻辑
- `UserHandler` 嵌入 `*Handler[model.User]` 并持有 `*UserService`，注册路由可调用 `RegisterBaseRoutes` 后再追加自定义路由

## 路由自动发现

`internal/router/registry/` 提供 `Routes` 切片 + `Register(fn)` 函数。

子模块在 `init()` 中调用 `registry.Register(func(r *gin.Engine) { ... })` 自注册路由分组，`internal/router/router.go` 通过空白导入触发 `init()`：

router.go ──_ import──→ admin/ ──init()──→ registry.Register(...)
           ├─_ import──→ user/  ──init()──→ registry.Register(...)
           └─Setup() ──→ 遍历 registry.Routes

新增路由模块：
1. 新建子目录并于 `init()` 调 `registry.Register`
2. 在 `router.go` 加一行空白导入以触发 `init()`

## 启动流程

1. `config.Init()` — 合并 `config/*.yaml` + `.env.yaml`，环境变量覆盖，热加载
2. `database.Init()` — 连接 PostgreSQL，配置读写分离，`AutoMigrate` 所有 `model.Register()` 的模型
3. `engine.Use(database.Middleware())` — 注入 `*gorm.DB` 到 `gin.Context`
4. `router.Setup(engine)` — 遍历 `registry.Routes` 注册所有路由
5. `engine.Run()`

## 关键约定

- **只使用 GET 和 POST 请求方式**：大多数 CDN/全站加速 服务对 PUT、DELETE 兼容性差
- **包名**：全小写、单数、无下划线（`handler`、`service`）
- **文件名**：全小写、下划线分隔（`user_service.go`）
- **导出符号**：大驼峰，私有符号小驼峰
- **数据库字段**：蛇形命名（`user_name`、`created_at`）
- **统一响应**：`response.Success(c, opts...)` / `response.Fail(c, opts...)`，支持 functional options 和链式调用两种风格，优先使用 functional options
- **GORM**：尽量使用 `GORM` 的 `Generics API（gorm.G[Model](db)....）`，而不是 `Traditional API`；在使用 `Generics API` 时，一般应直接使用全局 db 实例（`internal\database\database.go` 中的 `GetDB` 可获取），调用操作方法时再传递合适的 ctx 即可。
```

# 整理目录结构

### 新增目录

比起最初的目录结构，我在 `internal` 下增加了 `database、response、router` 三个目录，它们三都是值得于 internal 单独一个目录存放的内容。

1. `database`：数据库的重要性不必多少，单独一个包首先就是调用方法简洁方便，虽目前只有数据库连接实现、未来可能会有数据库中间件、一些数据库相关的全局类等等
2. `router`：路由系统作为 WEB 项目核心，且未来路由定义文件可能很多，单目录（又支持子级目录）的路由自动发现和注册非常合理
3. `response`：存放统一响应的实现，响应比数据库的调用会更频繁，单独包只为调用方便也是值得的

### 四层架构功能明确

即 Handler → Service → Repository → Model 四层应用架构，说实话，我还是第一次接触这种架构，只是基本理解，这里结合各大 AI 的解释，对架构各部分功能进行明确如下：

#### Handler
1. 处理器层
2. 解析请求（JSON → struct）> 调用 Service > 序列化响应（struct → JSON）
3. 可以做请求合法性校验，如手机号格式、参数非空（一般是参数不对就可以直接返回错误信息的检查）

#### Service
1. 服务层
2. 处理业务逻辑，业务规则校验（如余额是否足够），逻辑编排，事务控制
3. 接收 Handler 层调用，调用 Repository 层，返回结果给 Handler 层

#### Repository
1. 仓储层
2. 只处理数据库操作，把数据从 DB 搬到内存，或反过来
3. 接收 Service 层调用，返回结果给 Service 层

#### Model
1. 模型层
2. 只定义表结构，可有表名函数

#### 常见问题

##### 如何区分某参数应该在 Handler 还是 Service 层检查？

Handler 中的检查，和业务无关，只看 “请求对不对”，非法请求在入口直接拦截，不进业务、不查库，包括：

1. 参数非空（必传字段）
2. 参数格式错误（手机号、邮箱、身份证、UUID）
3. 参数长度超限（用户名太短 / 太长）
4. 参数类型错误（数字传成字符串）
5. 枚举值不在范围内（性别只允许男 / 女 / 未知）
6. 路径参数 / 查询参数格式错误
7. 请求体结构非法

总结：**前端传错了、格式错了、少传了 → Handler 直接拦掉**

Service 中的检查和业务强相关，必须依赖业务逻辑 / 数据库 / 状态判断的，包括：

1. 数据是否已存在（用户名重复、手机号已注册）
2. 数据状态是否允许操作（已取消的订单不能支付）
3. 权限校验（不是自己的地址不能删除）
4. 业务规则限制（余额不足、库存不足、年龄不够）
5. 多参数关联校验（开始时间不能晚于结束时间）
6. 跨表 / 跨服务校验（用户是否存在、商品是否上架）

总结：**需要查库、需要业务状态、需要走流程、前端不可能知道的**

##### Handler 和 Service 能否融合？
不能，Service 层可以脱离 HTTP 单独测试，Service 层还可以供定时任务、消息队列直接使用。
