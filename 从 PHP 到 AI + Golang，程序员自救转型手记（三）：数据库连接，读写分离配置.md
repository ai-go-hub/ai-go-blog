# 从 PHP 到 AI + Golang，程序员自救转型手记（三）：数据库连接，读写分离配置

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

> 此文发布时已经写到【从 PHP 到 AI + Golang，程序员自救转型手记（二十一）：登录接口整合点选验证码，AI 大翻车】已经完成了项目基本架构和前几个接口，不用担心作者弃坑，请放心阅读。

在上一期，我们已经完成 “目录结构创建、初始化 GIT、设计并开发配置系统”，本期将完成：数据库连接，读写分离配置

# 数据库连接

经过多方对比，决定选择 [GORM](https://gorm.io/zh_CN/) 作为数据库框架，它是一个致力于对开发者友好的优秀 Golang ORM 库，在 github 上有将近 40k star。

给 cc 的提示词如下：
1. 使用 GORM（https://gorm.io）作为数据库框架
2. 实现读写分离，且读副本，也需要有 MaxOpenConns 等连接池配置
3. 在 main.go 中完成数据库连接，初始化函数名为 Init（同配置系统的初始化函数），该函数无需额外传递参数，所需的数据库配置项在函数内获取
4. 数据库初始化的代码文件放于 `/internal/database/database.go` 内（这个目录结构也是经过研究的，是一个很合理的方案）
5. 修改配置文件中的数据库配置：写库和读库可单独配置，但共用 type 和 prefix 字段，读库的配置项额外增加 enabled 字段，表示是否启用读库，其余配置于写库一致

> PS：作者安装依赖是一般都会带上 github 或其他官方链接，这是很有必要的，主要避免 AI 安装了盗版依赖包，比如某个黑客写了个 https://github.com/hacker/air-true，并全网发布各类教程/宣发文章，此时若我们的提示词不带官方仓库链接，AI 可能会联网搜索 golang air 依赖，随即被网文所骗，且它还能正常运行，只是额外携带了病毒代码，半夜用你的电脑/服务器挖坑或当做肉鸡使用。

经过多次手工确认，AI 工作完成后，我们得到以下输出：

```bash
# 大模型是概率生成，不是固定查表，再加上采样策略、上下文、版本等影响，同问题输出都会有差异，以下 AI 输出、包括生成的代码仅供参考，可直接忽略，继续向下查看如何寻找问题和完成整改

所有任务已完成。以下是变更总结：

  修改的文件

  config/config.go — 重构数据库配置结构体：
  - 新增 DBInstance 结构体，包含连接参数和连接池配置（写库/读库共用）
  - 新增 ReadDBInstance 结构体，嵌入 DBInstance 并额外包含 Enabled bool 字段
  - Database 结构体将 Type、Prefix 提升为共享字段，Write 和 Read 各自独立配置

  config/config.yaml / .env.yaml.example / .env.yaml — 更新为新的读写分离 YAML 结构：
  database:
    type: postgres
    prefix:
    write:
      host: ...
      max_open_conns: 25
      ...
    read:
      enabled: false   # 是否启用读库
      host: ...
      max_open_conns: 25
      ...

  新增的文件

  internal/database/database.go — 数据库初始化：
  - Init() 函数无参数，内部通过 config.Get() 获取配置
  - 使用 GORM + PostgreSQL 驱动连接写库，配置了 TablePrefix
  - 写库连接池：SetMaxOpenConns / SetMaxIdleConns / SetConnMaxLifetime
  - 当 read.enabled = true 时，通过 dbresolver 插件注册读库，同样配置完整的连接池参数
  - 暴露 DB() 函数获取 *gorm.DB 实例

  cmd/api/main.go — 在 config.Init() 之后调用 database.Init()，并更新打印信息以反映新的配置结构。

✻ Sautéed for 4m 14s
```

首先检查目录结构的变更是否合理，有问题就先要求修改，目录结构没问题之后，才开始对代码 `review`：

#### 结构体命名修改不合理

```go
// config.go 文件

- type Database struct {
    // ...
}

+ type DBInstance struct {
    // ...
}
```

如上，AI 将配置的结构体名称给改了，从 `Database` 改为了 `DBInstance`，这是数据库连接配置的结构体，含有数据库的 `type、host、port` 等字段，使用 `Instance` 命名非常不合理，因为 `Instance` 一般表示数据库实例。还有就是配置 database.prefix 之前，没有检查 prefix 的值是否为空，接下来要求 AI 改正：

检查代码发现以下问题：
1. 原 `Database` 结构体被改名为了 `DBInstance`，用词不合理，可改为 `DatabaseConfig` 存储总的数据库配置，单个数据库连接配置的结构体可以命名为 `DatabaseSource`，读库配置的结构体名称也不要使用 `Instance` 单词，可以为 `ReadConfig`
2. `database.prefix` 有值才使用，否则忽略

```bash
# 大模型是概率生成，不是固定查表，再加上采样策略、上下文、版本等影响，同问题输出都会有差异，以下 AI 输出、包括生成的代码仅供参考，可直接忽略，继续向下查看如何寻找问题和完成整改

编译和静态检查均通过。两个问题已修复：

  1. 结构体重命名：
    - DBInstance → DatabaseSource（单个数据库连接配置）
    - ReadDBInstance → ReadConfig（读库配置，Enabled + 嵌入 DatabaseSource）
    - Database → DatabaseConfig（聚合配置：Type、Prefix、Write、Read）
  2. database.prefix 有值才使用：Init() 中仅在 cfg.Database.Prefix != "" 时才设置 GORM 的 NamingStrategy.TablePrefix。
```

由于提示词写的非常细，甚至直接提供了新的变量名等，这回一次性修改完成，代码开源于：[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)；核心代码如下：

```yaml
# config/config.yaml 文件
app:
  name: ai-go-mall

server:
  port: 8080

database:
  type: postgres
  prefix:
  write:
    host: 127.0.0.1
    port: 5432
    user: postgres
    password: "123456"
    dbname: aigo
    sslmode: disable
    timezone: Asia/Shanghai
    max_open_conns: 25
    max_idle_conns: 10
    conn_max_lifetime: 300 # 秒
  read:
    enabled: false # 是否启用读写分离
    host: 
    port: 5433 # 读副本默认配置了不同的端口号
    user: 
    password: 
    dbname: 
    sslmode: disable
    timezone: Asia/Shanghai
    max_open_conns: 50
    max_idle_conns: 20
    conn_max_lifetime: 300 # 秒
```

```go
// internal\database\database.go 文件

package database

import (
	"fmt"
	"time"

	"ai-go-mall/config"

	"gorm.io/driver/postgres"
	"gorm.io/gorm"
	"gorm.io/gorm/schema"
	"gorm.io/plugin/dbresolver"
)

var db *gorm.DB

// 初始化数据库连接
func Init() error {
	cfg := config.Get().Database

	gormCfg := &gorm.Config{}
	if cfg.Prefix != "" {
		gormCfg.NamingStrategy = schema.NamingStrategy{
			TablePrefix: cfg.Prefix,
		}
	}

	var err error
	db, err = gorm.Open(postgres.Open(buildDSN(cfg.Write)), gormCfg)
	if err != nil {
		return fmt.Errorf("connect write database: %w", err)
	}

	sqlDB, err := db.DB()
	if err != nil {
		return fmt.Errorf("get sql.DB: %w", err)
	}

	// 连接池配置
	sqlDB.SetMaxOpenConns(cfg.Write.MaxOpenConns)
	sqlDB.SetMaxIdleConns(cfg.Write.MaxIdleConns)
	sqlDB.SetConnMaxLifetime(time.Duration(cfg.Write.ConnMaxLifetime) * time.Second)

	// 配置读写分离
	if cfg.Read.Enabled {
		resolver := dbresolver.Register(dbresolver.Config{
			Replicas: []gorm.Dialector{postgres.Open(buildDSN(cfg.Read.DatabaseSource))},
		})
		resolver.SetMaxOpenConns(cfg.Read.MaxOpenConns)
		resolver.SetMaxIdleConns(cfg.Read.MaxIdleConns)
		resolver.SetConnMaxLifetime(time.Duration(cfg.Read.ConnMaxLifetime) * time.Second)

		if err := db.Use(resolver); err != nil {
			return fmt.Errorf("configure read database: %w", err)
		}
	}

	return nil
}

// 返回全局数据库实例
func GetDB() *gorm.DB {
	return db
}

// 根据配置构建 PostgreSQL DSN
func buildDSN(src config.DatabaseSource) string {
	return fmt.Sprintf("host=%s user=%s password=%s dbname=%s port=%d sslmode=%s TimeZone=%s",
		src.Host, src.User, src.Password, src.DBName, src.Port, src.SSLMode, src.Timezone,
	)
}
```