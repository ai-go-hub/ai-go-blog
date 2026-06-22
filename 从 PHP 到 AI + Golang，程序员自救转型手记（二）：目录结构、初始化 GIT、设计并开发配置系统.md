# 从 PHP 到 AI + Golang，程序员自救转型手记（二）：目录结构、初始化 GIT、设计并开发配置系统

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

> 此文发布时已经写到【从 PHP 到 AI + Golang，程序员自救转型手记（二十一）：登录接口整合点选验证码，AI 大翻车】已经完成了项目基本架构和前几个接口，不用担心作者弃坑，请放心阅读。

在上一期，我们已经讲到必须学什么，如何快速学习，以及完成立项前准备，本期将完成 ai-go-mall 的目录结构创建、初始化 GIT、设计并开发配置系统。

# 目录结构

1. 创建 `ai-go-mall` 目录，并执行 `go mod init ai-go-mall` 命令完成初始化
2. 下载第一期准备好的 `项目目录结构.md` 和 `编码风格最佳实践.md` 放入 `/doc` 目录
3. 切换到 `ai-go-mall` 目录，执行 `claude` 命令，直接让它：根据 @doc\项目目录结构.md 创建需要的文件夹
4. AI 需要授权时，一般选择同意，需要防止越过项目执行命令，防止安装盗版依赖，不建议使用中转站以防止中间人攻击，最后我们会对每一行代码进行 `review`。

```bash
.
├── go.mod
├── .claude/
│   └── settings.local.json
├── cmd/
│   └── api/
├── config/
├── doc/
│   ├── 项目目录结构.md
│   └── 编码风格最佳实践.md
├── internal/
│   ├── handler/
│   ├── model/
│   ├── repository/
│   └── service/
├── pkg/
│   ├── logger/
│   └── validator/
└── script/
```

4. 直接将短时间应该用不上的目录删除，作者这里删除了 `pkg` 和 `script`，同时创建缺少的 `cmd/api/main.go` 文件

# 初始化 GIT

git 是全球领先的分布式版本控制系统，程序员必备，不会的快去学，很简单。

```bash
git init
git config user.name "yang"
git config user.email "1094963513@qq.com"
```

如果不需要上传至远程仓库备份的话，name 和 email 可以随便写，接下来要求 `claude`：帮我建立 .gitignore 文件，忽略 .DS_Store、*.css.map、*.local.* 等通常需要忽略的文件。

作者这边生成的 .gitignore 文件内容如下：

```bash
# OS
.DS_Store
Thumbs.db

# IDE
.idea/
.vscode/
*.swp
*.swo

# Local config
*.local.*
.env.*

# Go
/bin/
/vendor/
*.exe
*.test
*.out

# Frontend
*.css.map
*.js.map
node_modules/
/dist/

# Build output
/tmp/
/log/
```

作者手动将 `*.swo、*.swp` 改为了 `*.sw?`，去掉了 `.env`

# 设计并开发配置系统

经过多方对比，决定选择 [Viper](https://github.com/spf13/viper) 管理配置，它是适用于 Go 应用程序的完整配置解决方案，github 上有 30k star，非常完善和通用。

规划配置系统（直接复制以下规划，让 cc 帮助实现）：
- 使用 github.com/spf13/viper 管理配置
- 初版有 `应用名称、开发服务端口号、数据库相关` 配置项，放在一个配置文件内即可
- 所有配置文件，使用 YAML 格式保存于 /config 目录下，配置系统能自动加载所有 YAML 配置文件，增加配置文件时，不需要手动改代码
- 额外一个 .env.yaml 环境配置文件，优先级最高，用于开发环境自定义数据库配置，GIT 需要忽略此文件
- 额外一个 .env.yaml.example 文件，GIT 不忽略，用于方便开发者将项目拉到本地之后，快速创建 .env.yaml 文件
- 数据库配置细节：带有 type 字段表示数据库类型，默认使用 PostgreSQL 数据库；带有 max_open_conns 等连接池配置；

> PS：作者安装依赖是一般都会带上 github 或其他官方链接，这是很有必要的，主要避免 AI 安装了盗版依赖包，比如某个黑客写了个 https://github.com/hacker/air-true，并全网发布各类教程/宣发文章，此时若我们的提示词不带官方仓库链接，AI 可能会联网搜索 golang air 依赖，随即被网文所骗，且它还能正常运行，只是额外携带了病毒代码，半夜用你的电脑/服务器挖坑或当做肉鸡使用。

AI 工作完成后，我们得到以下输出：

```bash
# 大模型是概率生成，不是固定查表，再加上采样策略、上下文、版本等影响，同问题输出都会有差异，以下 AI 输出、包括生成的代码仅供参考，可直接忽略，继续向下查看如何寻找问题和完成整改

配置系统已实现完毕。以下是最终文件结构和验证结果：

  config/
  ├── config.yaml           # 主配置文件（Git 跟踪）
  ├── .env.yaml             # 本地环境配置（Git 忽略）
  └── .env.yaml.example     # 环境配置示例（Git 跟踪）
  pkg/config/
  └── config.go             # 配置加载逻辑
  cmd/api/
  └── main.go               # 入口，调用配置系统

  核心设计：

  - config.Load("config") 自动扫描 /config 目录下所有 .yaml 文件，按文件名排序后逐个 Merge，.env.yaml 排最后加载，值覆盖前面的文件
  - 新增配置文件无需改代码，放入 /config 目录即可
  - 数据库默认类型为 PostgreSQL，连接池默认值 max_open=25 / max_idle=10 / max_lifetime=5m
  - 提供 ConnMaxLifetimeDuration() 方法将字符串转为 time.Duration

  验证结果：

  ┌─────────────────┬─────────────┬─────────────────────┐
  │     配置项      │ config.yaml │ .env.yaml（覆盖后） │
  ├─────────────────┼─────────────┼─────────────────────┤
  │ app.name        │ ai-go-mall  │ ai-go-mall          │
  ├─────────────────┼─────────────┼─────────────────────┤
  │ server.port     │ 8080        │ 8080                │
  ├─────────────────┼─────────────┼─────────────────────┤
  │ database.host   │ localhost   │ 192.168.1.100       │
  ├─────────────────┼─────────────┼─────────────────────┤
  │ database.port   │ 5432        │ 5433                │
  ├─────────────────┼─────────────┼─────────────────────┤
  │ database.dbname │ ai-go-mall  │ ai-go-mall-dev      │
  └─────────────────┴─────────────┴─────────────────────┘

✻ Churned for 2m 50s
```

检查结果，从目录结构的修改看起，发现一些问题：
1. AI 将 `.env.yaml` 和 `.env.yaml.example` 放入了 `/config` 目录，这与预期不符，其实我的本意是希望环境配置直接在根目录
2. 然后是配置文件的加载逻辑放在了 `pkg/config.go`，这虽然很合理，但是我更希望配置加载直接在 `/config` 目录以内，即最终得到 `/config/config.go + 多个 .yaml 文件` 的目录结构（未来会迁移到 `internal` 目录，最终 `/config` 只存储 `yaml` 文件）。
3. .env.yaml 里边只有数据库配置，没有 server.port 和 app.name 配置项

将以上发现直接发给 cc，让它继续修改，由于目录结构错误，暂时未对配置加载的代码进行 `review`：

```bash
# 大模型是概率生成，不是固定查表，再加上采样策略、上下文、版本等影响，同问题输出都会有差异，以下 AI 输出、包括生成的代码仅供参考，可直接忽略，继续向下查看如何寻找问题和完成整改

三个问题已全部修正，最终结构如下：

  .
  ├── .env.yaml              # 根目录，Git 忽略 ✓
  ├── .env.yaml.example      # 根目录，Git 跟踪，含全部配置项 ✓
  ├── .gitignore
  ├── config/
  │   ├── config.go          # 配置加载逻辑 ✓
  │   └── config.yaml        # 主配置文件
  └── cmd/api/main.go

  改动要点：

  ┌──────────────────────────────────┬───────────────────────────────────────────────────────────┐
  │               问题               │                           修正                            │
  ├──────────────────────────────────┼───────────────────────────────────────────────────────────┤
  │ .env.yaml 放入 /config           │ 移至根目录                                                │
  ├──────────────────────────────────┼───────────────────────────────────────────────────────────┤
  │ config.go 放入 /pkg/config       │ 移至 /config/config.go，import 路径变为 ai-go-mall/config │
  ├──────────────────────────────────┼───────────────────────────────────────────────────────────┤
  │ .env.yaml.example 仅含数据库配置 │ 补充了 app.name、server.port 及数据库全套配置             │
  └──────────────────────────────────┴───────────────────────────────────────────────────────────┘

  加载顺序： config/*.yaml → 根目录 .env.yaml（最后加载，覆盖前面的值）。.env.yaml 不存在时优雅降级，不报错。

✻ Worked for 1m 24s
```

接下来的工作就非常重要了，为了学习和商业级项目的目标，我们需要**逐行理解代码**，并找出代码中的不足，或者就特意的做一些自己更加喜欢的修改，作者找到的问题如下：
1. config.Load 函数需要传递 config 目录路径，这在当前项目是不需要的，固定 /config 目录即可
2. config.Load 有加载的意思，在 main.go 中调用，但我认为改为 config.Init 函数更合理
3. 未加载到任何配置文件时，不需要报错
4. 数据库配置缺失 `timezone` 和 `prefix`（历往项目习惯有个 `prefix` 配置项，可以不用，但可以有）
5. 不需要对数据库配置进行任何预处理，后续于连接数据库函数内处理
6. 经过多方研究，再要求 AI 提供一个 `config.Get` 函数，用于获取配置项（现在是 `cfg.App.Name`，改为 `config.Get().App.Name`），前者配置可以被外部修改，且包装一个方法扩展性总是更高一些的
7. 对环境变量进行加载，并覆盖值到已有配置项，即调用 `viper.AutomaticEnv()`（非加载项目的 .env.yaml 环境配置文件），这种环境变量可以通过以下方式动态设定：

```bash
# Linux / Mac
SERVER_PORT=8080 APP_NAME=myapp go run main.go

# Windows CMD
set SERVER_PORT=8080&& set APP_NAME=myapp&& go run main.go

# Windows PowerShell
$env:SERVER_PORT="8080"; $env:APP_NAME="myapp"; go run main.go
```

动态设定的 APP_NAME 会覆盖到 app.name 配置项，cc 还修改了 `go.mod` 和 `.gitignore` 等文件，其中对 `.gitignore` 的修改如下：

```bash
# .gitignore
- .env.*

+ .env.yaml
+ !.env.yaml.example
```

修改很合理，但是 `+ !.env.yaml.example` 是多余的，直接删除即可。

继续要求 AI 按需求进行修改，再经过几轮对话之后，配置系统开发完成，完整代码开源于：[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)，核心代码如下：

1. 又让 AI 增加了监听配置改变，以及配置数据的读写锁
2. 监听配置改变时，当新配置为空可能不会覆盖，经研究后添加了 `c.ZeroFields = true` 选项
3. 同时询问了 AI，确定：结构体定义代码在最上面，是 golang 项目通用做法

```go
// main.go

package main

import (
	"fmt"
	"log"

	"ai-go-mall/config"
)

func main() {
	// 初始化配置
	if err := config.Init(); err != nil {
		log.Fatalf("config init: %v", err)
	}

	cfg := config.Get()

	fmt.Printf("应用名称: %s\n", cfg.App.Name)
	fmt.Printf("服务端口: %d\n", cfg.Server.Port)
	fmt.Printf("数据库类型: %s\n", cfg.Database.Type)
	fmt.Printf("数据库地址: %s:%d\n", cfg.Database.Host, cfg.Database.Port)
	fmt.Printf("数据库名称: %s\n", cfg.Database.DBName)
	fmt.Printf("连接池配置: max_open=%d max_idle=%d max_lifetime=%d\n",
		cfg.Database.MaxOpenConns,
		cfg.Database.MaxIdleConns,
		cfg.Database.ConnMaxLifetime,
	)
}
```

```yaml
app:
  name: ai-go-mall

server:
  port: 8080

database:
  type: postgres
  host: 127.0.0.1
  port: 5432
  user: postgres
  password: "123456"
  dbname: aigo
  prefix: 
  sslmode: disable
  timezone: Asia/Shanghai
  max_open_conns: 25
  max_idle_conns: 10
  conn_max_lifetime: 300 # 秒
```

```go
// config.go

package config

import (
	"fmt"
	"log"
	"os"
	"path/filepath"
	"sync"

	"github.com/fsnotify/fsnotify"
	"github.com/go-viper/mapstructure/v2"
	"github.com/spf13/viper"
)

// App 应用配置
type App struct {
	Name string
}

// Server 服务配置
type Server struct {
	Port int
}

// Database 数据库配置
type Database struct {
	Type            string
	Host            string
	Port            int
	User            string
	Password        string
	Prefix          string
	DBName          string `mapstructure:"dbname"`
	SSLMode         string `mapstructure:"sslmode"`
	Timezone        string `mapstructure:"timezone"`
	MaxOpenConns    int    `mapstructure:"max_open_conns"`
	MaxIdleConns    int    `mapstructure:"max_idle_conns"`
	ConnMaxLifetime int    `mapstructure:"conn_max_lifetime"`
}

// Config 聚合配置
type Config struct {
	App      App
	Server   Server
	Database Database
}

var (
	vip    *viper.Viper
	config = new(Config)
	mu     sync.RWMutex
)

// 初始化配置
func Init() error {
	if vip == nil {
		vip = viper.New()

		// 读取 config 目录下所有 yaml 文件
		files, err := filepath.Glob(filepath.Join("./config", "*.yaml"))
		if err != nil {
			return fmt.Errorf("get config files: %w", err)
		}

		for _, file := range files {
			vip.SetConfigFile(file)
			if err := vip.MergeInConfig(); err != nil {
				return fmt.Errorf("merge config file %s: %w", file, err)
			}
		}

		// 加载根目录 .env.yaml 文件（覆盖从 config 文件夹读取的配置）
		if _, err := os.Stat(".env.yaml"); err == nil {
			vip.SetConfigFile(".env.yaml")
			if err := vip.MergeInConfig(); err != nil {
				return fmt.Errorf("merge config file .env.yaml: %w", err)
			}
		}

		// 支持环境变量覆盖
		vip.AutomaticEnv()

		// 解析到结构体
		if err := vip.Unmarshal(config); err != nil {
			return fmt.Errorf("unmarshal config: %w", err)
		}

		// 监听配置改变
		vip.OnConfigChange(func(e fsnotify.Event) {
			mu.Lock()
			defer mu.Unlock()
			if err := vip.Unmarshal(config, func(c *mapstructure.DecoderConfig) {
				c.ZeroFields = true
			}); err != nil {
				log.Printf("config reload failed: %v", err)
			}
		})
		vip.WatchConfig()
	}
	return nil
}

// 返回全局配置副本。
func Get() Config {
	mu.RLock()
	defer mu.RUnlock()
	return *config
}

// 获取 Viper 全局单例
func GetViper() *viper.Viper {
	return vip
}

// 是否已经初始化
func IsInit() bool {
	return vip != nil
}
```