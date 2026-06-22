# Golang WEB 项目目录结构

本文是系列文章《从 PHP 到 AI + Golang，程序员自救转型手记》作者整理而来，完整版开源于：[github](https://github.com/ai-go-hub/ai-go-blog) | [gitee](https://gitee.com/ai-go-hub/ai-go-blog)

> 这是一份使用 豆包、文心一言等大模型，反复确认了 go web 项目的通用目录结构，并要求它们提供参考来源，提供 github 链接，人工确认整理 top20 高星项目，最终得到了一份 完全贴合 Go 官方风格 + 企业级落地标准 的目录结构设计，设计随 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）项目持续完善。

```bash
.
├── .claude                     # Claude Code 相关
│
├── cmd/                        # 主应用程序入口
│   └── api/                    # 每个可执行文件一个子目录
│       └── main.go             # 包含 `main()` 函数
│
├── internal/                   # 私有应用代码（仅限本项目内部使用）
│   ├── infra/                  # 基础设施层
│   │   ├── database/           # 初始化数据库连接等
│   │   ├── captcha/            # 验证码相关
│   │   ├── token/              # 令牌相关
│   │   │   ├── driver          # 令牌存储驱动，文件名代表对应驱动的实现
│   │   │   │   ├── database.go
│   │   │   │   └── redis.go
│   │   │   └─── token.go
│   │   └── upload/             # 文件上传相关，也是多驱动设计
│   │
│   ├── handler/                # 处理器，解析请求（JSON→struct）> 调用 Service > 序列化响应（struct→JSON），可做参数格式合法性检查，如手机号格式、参数非空（只处理参数、返回值）
│   ├── model/                  # 数据模型（只定义表结构，表名函数）
│   ├── repository/             # 数据访问层（只处理数据库操作，把数据从 DB 搬到内存，或反过来）
│   ├── response/               # 统一响应结构定义和函数
│   ├── router/                 # 路由定义及路由自动发现实现
│   └── service/                # 业务服务层，参数业务合法性检查，如用户名不存在、余额是否足够，逻辑编排、事务控制（只处理业务逻辑）
│
├── pkg/                        # 公共库代码（可被其他项目引用）
│   ├── logger/                 # 日志封装
│   ├── validator/              # 参数校验
│   └── ...                     # 其他可复用模块
│
├── config/                     # 配置文件模板（如 YAML/JSON）
├── script/                     # 构建/部署脚本
├── .env.yaml.example           # 环境配置文件示例
├── .gitignore                  # Git 忽略规则
├── CLAUDE.md                   # Claude Code 的工作指导文档
├── go.mod                      # 依赖管理
├── go.sum                      # 依赖校验
├── LICENSE                     # 许可证文件
└── README.md                   # 项目说明
```

## 结构特点

1. 采用多入口设计（即 cmd 下面一个目录一个入口），golang 的源码也是这样设计的
2. 和部分比较知名项目略有不同的是，文件夹名称全部使用单数（包括 config 和 script）
    - 首先是因为这是 Go 社区的命名惯例，比如 `vendor、test`，标准库也是清一色的单数命名
    - 其次就是为了保持风格统一，特别是 `internal` 下面的 `handler、model、service` 都是单数形式；很多项目外层是复数形式的 `config`，内层是 `handler` 的单数形式。
    - 以上命名方式都有自己的解释，但我认为最重要的还是 Go 社区给包命名的定义：**包名描述的是"这个包是什么"，而不是"这个包里装了什么"，所以应该使用单数。**
3. 分层架构的调用关系（固定流向，永远不变）：`Handler > Service > Repository > Model`
4. 结构并不完整，后续根据实际项目需求扩展

## 核心结构选型背景

目录结构，特别是 `internal` 下面的 `handler、model、service、model` 四层直接平铺经过很长时间的研究。

首先是这种事情问 AI 太磨人了，不同的 AI 不同的答案，换个问法它自己还会修改观点，只能自己全面调研和研究...

go 社区有很多种做法：

#### 第一种：一个业务或者能力相关的都放在一个目录内（ddd），比如：

```bash
├── internal/
│   ├── user/
│   │   ├── handler
│   │   ├── service
│   │   ├── repository
│   │   └── model
│   └── order/
│   │   ├── handler
│   │   ├── service
│   │   ├── repository
│   │   └── model
```

这种方案目前不合适，项目前期单体架构即可，微服务架构是未来考虑的事，前期发展试错阶段业务增多，同级的能力也会极速增多，找文件夹都得翻页。

#### 第二种：以 app 作为顶层，然后再以业务进行划分，比如：

```bash
├── internal/
│   ├── app/
│   │   └── admin
│   │   │   ├── user/
│   │   │   │   ├── handler
│   │   │   │   ├── service
│   │   │   │   ├── repository
│   │   │   │   ├── model
│   │   │   └── order/
│   │   │   │   │   ├── handler
│   │   │   │   │   ├── service
│   │   │   │   │   ├── repository
│   │   │   │   │   └── model
```

这种方案，由于 golang 中多了一层 `internal`，显得目录层级很深，而控制器、模型等又是经常改动的文件。

#### 第三种：即现在的方案，internal 下平铺分层架构的目录，放弃 app 目录，比如：

```bash
├── internal/
│   ├── handler/
│   │   ├── user/
│   │   │   ├── log.go
│   │   │   ├── rule.go
│   │   │   ├── ...
│   │   │   └── group.go
│   ├── model/
│   ├── repository/
│   └── service/
```

放弃 app 目录，直接就让层级 - 2 （去掉 app/admin），新的结构到了 handler、model 以内：你想分的粗一点，就是 common、user、admin，而想分的细一点，可以是 group、log、rule、config、article、fields 等。

#### 其他

-   也有看到 golang 的 internal 机制出来之前的项目，这类项目将业务代码放在 internal 之外，可能被外部引用。
-   也有看到明显是从其它语言带过来的方案，`biz 存放业务逻辑层，dal 存放数据访问层，schema 存放数据模型层`
-   自带 API 分版本的方案 `api > v1 > system`，我们的项目之所以不分版本，是因为当你需要文件夹层面分版本时，再去新建文件夹，复制过去修改即可，开源系统本身不带这个。
-   `app > admin > models + apis + service` 的方案
