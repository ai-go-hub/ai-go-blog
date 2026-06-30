# 从 PHP 到 AI + Golang，程序员自救转型手记（十六）：目录结构更新、完善 token 系统（AI 表示 token 入库无需加密？）

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “优化细节、网络请求封装”，本期将完成：目录结构更新、完善 token 系统

# 对 AI 的小考验

今天在完善 token 系统之前，特意考验了一下 AI ：token 作为管理员身份令牌，其值生成打算使用 `UUID v7`，入库是否还有必要加密

```bash
不需要。

UUID v7 的安全性已足够
UUID v7 结构为 48 位时间戳 + 74 位随机数，每秒可生成 2^74 ≈ 1.8×10²² 个不重复值。暴力枚举一个 128 位的 UUID 在计算上不可行。

哈希的代价，巴拉巴拉...
真正该加哈希的场景，巴拉巴拉...
```

**我去，这不对吧，加密最大的意义是防止被拖库，token 泄露，黑客直接就登录到后台了，而有加密的话黑客必须同时获得程序源码和 token 密文才能入侵到后台，怎么能不加密呢？**

# 目录结构更新

### 新增 infra 目录

目前我们的数据库初始化函数放在 `internal/database/database.go` 文件中，考虑到接下来要增加的 token（多驱动设计）、captcha、upload（多驱动设计）等，按原来的计划，我们的目录结构未来会变成：

```bash
├── internal/
│   ├── handler/
│   ├── model/
│   ├── repository/
│   ├── router/
│   ├── database/
│   ├── captcha/
│   ├── upload/
│   ├── response/
│   ├── middleware/
│   └── service/
```

其中的 database、token、captcha、upload 非常的偏底层，和 handler、model 等业务层的有点不搭，所以我决定再加一层 `infra` 目录，存放基础设施，新的目录结构如下：

```bash
├── internal/
│   ├── infra/
│   │   ├── token/
│   │   │   ├── driver
│   │   │   │   ├── database.go
│   │   │   │   └── redis.go
│   │   │   └─── token.go
│   │   ├── captcha/
│   │   └── upload/
│   ├── handler/
│   ├── model/
│   ├── repository/
│   ├── router/
│   ├── response/
│   ├── middleware/
│   └── service/
```

即新增 `infra` 目录，将 `database`、`token`、`captcha`、`upload` 这类偏底层的放于其中，`router` 和 `response` 我没有选择移进去，其中 `router` 和业务层的放在一起是合理的，它是业务入口，`response` 算是业务出口，属于可移可不移选择不移。

### 配置解析（config.go 文件）移入 infra 目录

项目是一个应用（非库），而 config 模块是项目内部实现细节，且属于带状态的基础设施，现在我们已经建立了 `infra` 目录，那么**配置解析逻辑**（`config/config.go` 文件，非 `yaml` 配置文件）最理想的存放位置当然是 `internal\infra\config\config.go`，AI时代，我们直接让 cc 将其移入其中即可，这类需求只需要一句话，无需人工找文件替换，基本不会出问题，最多全项目搜索 /config 确定一下。

> PS：config/*.yaml 无需移动，该文件属于运行时配置，放在 /config 是符合社区习惯的做法。

# 完善 token 系统

token 系统规划如下：

1. 于 internal/model/common.go 建立 token 模型，有 token、type（字符串）、user_id、创建时间、过期时间字段，字段全部带有中文注释
2. 于 internal/infra/token/token.go 建立 token 管理接口和结构体，使用多驱动模式，所有驱动存放于 `internal/infra/token/driver` 目录，一个驱动一个文件，目前实现 database 一种驱动
3. 增加 `token.driver` 配置项，默认值为 `database`，`internal/infra/token/token.go` 内读取驱动配置，返回对应驱动的 token 管理实例。
4. token 驱动实现 `Create、Get、Delete、Clear（删除指定会员指定类型的所有token）` 方法，token 管理器的结构体除驱动的所有方法外，额外实现 `Check` 方法（使用 Get 方法读取 token 信息后检查是否过期）
5. token 入库使用 SHA256 加密

目前 token 系统还未考虑额外的全局秘钥（后续应该会考虑），也未考虑【SHA256 索引 + bcrypt 校验】双字段方案，但配合验权接口节流，就算泄露了 Token SHA256，抗爆破能力还是够的，将之前的规划发给 cc，最终核心代码如下：

```yaml
# config/config.yaml 增加 token 驱动配置，目前只实现了 database 驱动
# 未来可以增加 redis 等驱动，得益于 AI 的帮助，加驱动基本上只需要一句话

token:
  driver: database
```

```go
// internal\model\common.go 文件，用于存放 captcha、area 等公共模型

// Token 令牌模型，用于存储各类用户令牌
type Token struct {
	Token     string    `gorm:"comment:令牌;type:varchar(64);primaryKey" json:"-"`
	Type      string    `gorm:"comment:令牌类型;type:varchar(32);not null" json:"type"`
	UserID    uint      `gorm:"comment:用户ID;not null;index" json:"user_id"`
	CreatedAt time.Time `gorm:"comment:创建时间" json:"created_at"`
	ExpiredAt time.Time `gorm:"comment:过期时间;not null;index" json:"expired_at"`
}
```

```go
// internal\infra\token\token.go 文件，用于存放 token 管理接口和结构体

package token

import (
	"context"
	"crypto/sha256"
	"fmt"
	"sync"
	"time"

	"ai-go-mall/internal/infra/config"
	"ai-go-mall/internal/infra/token/driver"
	"ai-go-mall/internal/model"
)

// Driver 令牌存储驱动接口
type Driver interface {
	Create(ctx context.Context, token *model.Token) error
	Get(ctx context.Context, token string) (*model.Token, error)
	Delete(ctx context.Context, token string) error
	Clear(ctx context.Context, userID uint, tokenType string) error
}

// Manager 令牌管理器
type Manager struct {
	driver Driver
}

// NewManager 创建令牌管理器
func NewManager(driver Driver) *Manager {
	return &Manager{driver: driver}
}

// Create 创建令牌，入库前自动对 Token 做 SHA256
func (m *Manager) Create(ctx context.Context, token *model.Token) error {
	token.Token = sha256Hex(token.Token)
	return m.driver.Create(ctx, token)
}

// Get 获取令牌信息
func (m *Manager) Get(ctx context.Context, token string) (*model.Token, error) {
	return m.driver.Get(ctx, sha256Hex(token))
}

// Check 检查令牌是否存在且未过期
func (m *Manager) Check(ctx context.Context, token string) bool {
	t, err := m.Get(ctx, token)
	if err != nil || t == nil {
		return false
	}
	return time.Now().Before(t.ExpiredAt)
}

// Delete 删除令牌
func (m *Manager) Delete(ctx context.Context, token string) error {
	return m.driver.Delete(ctx, sha256Hex(token))
}

// Clear 清除指定用户指定类型的所有令牌
func (m *Manager) Clear(ctx context.Context, userID uint, tokenType string) error {
	return m.driver.Clear(ctx, userID, tokenType)
}

// sha256Hex 返回 raw 的 SHA256 十六进制字符串
func sha256Hex(raw string) string {
	sum := sha256.Sum256([]byte(raw))
	return fmt.Sprintf("%x", sum)
}

// ==================== 全局单例 ====================

var (
	instance *Manager
	once     sync.Once
)

// Instance 返回全局令牌管理器实例，首次调用时根据配置自动初始化
func Instance() *Manager {
	once.Do(func() {
		instance = NewManager(newDriver(config.Get().Token.Driver))
	})
	return instance
}

// newDriver 根据配置创建存储驱动
func newDriver(name string) Driver {
	switch name {
	default:
		return driver.NewDatabase()
	}
}
```

```go
// internal\infra\token\driver\database.go 文件，token 数据库驱动

package driver

import (
	"context"
	"errors"

	"ai-go-mall/internal/infra/database"
	"ai-go-mall/internal/model"

	"gorm.io/gorm"
)

// Database 基于关系型数据库的令牌驱动
type Database struct{}

// NewDatabase 创建数据库令牌驱动
func NewDatabase() *Database {
	return &Database{}
}

// Create 创建令牌
func (d *Database) Create(ctx context.Context, t *model.Token) error {
	return gorm.G[model.Token](database.DB()).Create(ctx, t)
}

// Get 获取令牌信息
func (d *Database) Get(ctx context.Context, token string) (*model.Token, error) {
	t, err := gorm.G[model.Token](database.DB()).
		Where("token = ?", token).
		First(ctx)
	if err != nil {
		if errors.Is(err, gorm.ErrRecordNotFound) {
			return nil, nil
		}
		return nil, err
	}
	return &t, nil
}

// Delete 删除令牌
func (d *Database) Delete(ctx context.Context, token string) error {
	_, err := gorm.G[model.Token](database.DB()).Where("token = ?", token).Delete(ctx)
	return err
}

// Clear 清除指定用户指定类型的所有令牌
func (d *Database) Clear(ctx context.Context, userID uint, tokenType string) error {
	_, err := gorm.G[model.Token](database.DB()).
		Where("user_id = ? AND type = ?", userID, tokenType).
		Delete(ctx)
	return err
}
```