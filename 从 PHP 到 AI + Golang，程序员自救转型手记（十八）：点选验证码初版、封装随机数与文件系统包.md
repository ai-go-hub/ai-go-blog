# 从 PHP 到 AI + Golang，程序员自救转型手记（十八）：点选验证码初版、封装随机数与文件系统包

这是一个系列教程，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall 项目的制作。

在上一期，我们已经完成 “登录接口完善，登录页接口整合，解决跨域”，本期将完成：点选验证码初版、封装随机数与文件系统包

# 点选验证码实现

### 设计数据模型

模型规划：表名 `captcha`，用于存放点选验证码数据，也可以存放其他 OTP（一次性密码）场景的验证数据（比如邮箱验证码、短信验证码，只需要多加一个 code 字段即可）

| 字段名     | 备注                                                                                                          |
| ---------- | ------------------------------------------------------------------------------------------------------------- |
| key        | 主键，验证码查询键，验证码唯一身份令牌，提交验证时携带                                                        |
| code       | 验证码值（加密后的，点选验证码不一定使用）                                                                    |
| info       | 验证码详细数据（点选验证码是存储元素的坐标 + 文字 JSON + 图片宽高等所有验证图片信息，用于验证点选坐标正确性） |
| expired_at | 过期时间                                                                                                      |
| created_at | 创建时间                                                                                                      |

不需要以下字段：
- status 验证状态，验证成功应直接删除，使之无法再次使用；特殊情况（如仅用于展示能否验证成功的预检可不删除，实际业务中只要进入验证成功逻辑，应立即删除，还可以顺便防止重复提交）
- bg_img 背景图片不需要，我们会提前准备 N 张大小相同的背景图，总是随机显示，即背景图不应影响验证的操作或结果

让 cc 根据以上规划，于 `internal\model\common.go` 创建 `Captcha` 模型，这也是 `Token` 模型所在的文件，我们的规划中，单文件可以存放多个模型，且有关联的模型/有相同分类的模型建议放在一起。

```bash
编译通过，无错误。Captcha 模型已创建完成，总结如下：

  ┌───────────┬──────────────┬──────────────────────────────────────────────────────────┐
  │   字段    │     类型     │                           说明                           │
  ├───────────┼──────────────┼──────────────────────────────────────────────────────────┤
  │ Key       │ varchar(64)  │ 主键，验证码唯一身份令牌，请求方携带此键验证             │
  ├───────────┼──────────────┼──────────────────────────────────────────────────────────┤
  │ Code      │ varchar(255) │ 加密后的验证码值，json:"-" 不序列化到响应中              │
  ├───────────┼──────────────┼──────────────────────────────────────────────────────────┤
  │ Info      │ text         │ 验证元素坐标 + 文字 JSON + 图片宽高，json:"-" 不返回前端 │
  ├───────────┼──────────────┼──────────────────────────────────────────────────────────┤
  │ ExpiredAt │ timestamp    │ 过期时间，建索引加速过期清理                             │
  ├───────────┼──────────────┼──────────────────────────────────────────────────────────┤
  │ CreatedAt │ timestamp    │ 创建时间                                                 │
  └───────────┴──────────────┴──────────────────────────────────────────────────────────┘

  - Code 和 Info 均使用 json:"-" 防止敏感数据泄露到 API 响应中
  - 已在 init() 中注册到 Register，AutoMigrate 会自动建表（表名 captchas）
```

写模型其实自己也能写，但是我偏偏是写了表格，然后让 AI 再过一遍，果然 AI 还是很灵性的，给 `ExpiredAt` 加了索引，未来也确实需要定时清理过期验证码。

然后是受益于我们以前的超前规划，新的模型只需要调用注册函数注册即可自动迁移（建表）：

```go
func init() {
    // 注册自动迁移
	Register(&Token{}, &Captcha{})
}

// Token 令牌模型，用于存储各类用户令牌
type Token struct {
	Token     string    `gorm:"comment:令牌;type:varchar(64);primaryKey" json:"-"`
	Type      string    `gorm:"comment:令牌类型;type:varchar(32);not null" json:"type"`
	UserID    uint      `gorm:"comment:用户ID;not null;index" json:"user_id"`
	CreatedAt time.Time `gorm:"comment:创建时间" json:"created_at"`
	ExpiredAt time.Time `gorm:"comment:过期时间;not null;index" json:"expired_at"`
}

// TableName 指定表名
func (Token) TableName() string {
	return "tokens"
}

// Captcha 验证码模型
type Captcha struct {
	Key       string    `gorm:"comment:验证码查询键;type:varchar(64);primaryKey" json:"key"`
	Code      string    `gorm:"comment:验证码值（加密后）;type:varchar(255)" json:"-"`
	Info      string    `gorm:"comment:验证码数据（坐标+文字JSON+图片宽高等）;type:text" json:"-"`
	ExpiredAt time.Time `gorm:"comment:过期时间;not null;index" json:"expired_at"`
	CreatedAt time.Time `gorm:"comment:创建时间" json:"created_at"`
}

// TableName 指定表名
func (Captcha) TableName() string {
	return "captchas"
}
```

而且我们的模型注册函数还设计为支持任意数量的参数，此时刚好用上。


### 点选验证码实现

#### 初版

点选验证码注定不是一个简单的功能，哪怕结合 AI，也需要仔细把关和整理；
而且验证码必然涉及文件操作和随机数生成，我们在开发验证码期间应该会将随机数生成和文件操作提取为单独的公共包。

以下是首版提示词，这类功能一般一次性写不出来，或者写出来也不能用，那就直接将需求写细一点，先找找感觉：

1. 新增验证码配置文件 `config/captcha.yaml`，提供 `元素、长度、混淆点长度、验证码默认过期时间` 配置，其中 `元素` 的值类型为数组，可选值为 `中文文字、英文大写字母、ICON` 三种，可混合配置，长度和混淆点长度默认均为 `2`，默认过期时间为 600 秒
2. 我已于 `asset\captcha\click` 准备好了点选验证码 `背景图` 和 `ICON` 图，于 `asset\font\SourceHanSansCN-Normal.ttf` 准备好了思源黑体字体简体中文字体文件，供你后续使用
4. 建立 `internal\infra\captcha` 包，实现点选验证码生成功能，有以下要求：
    1. 建立 init 函数，其中完成过期验证码清理
    2. 验证元素 `Icon` 的文件名即为它的名称，中文元素请生成随机字符
    3. 应用验证码配置文件中的配置  `config/captcha.yaml`
    4. 至少应该有 `创建点选验证码`、`检查点选验证码` 公开两个方法
    5. 创建点选验证码先随机取一张背景图，然后根据配置随机出 ICON、中文文字、英文大写字母，并随机绘制到背景图片上，不能绘制到背景图以外，然后将绘制坐标存入验证码数据库（Captcha模型），最终需要返回：`验证图片高宽（一般就是背景图的高宽）`，`验证元素` 数组原文，即 ICON 的名称，随机的中文或英文大写字母，`验证图片的 base64`（不需要存储图片，而是直接输出 base64），`验证码 key`
    6. 检查点选验证码根据 `key` 和 用户点击出的元素 `顺序`、`坐标`、前端实际渲染的图表的 `宽高`，等信息确定验证是否通过，方法还需要一个 `验证成功是否从数据库删除验证码信息` 的参数
    7. 总是考虑此人机验证码的安全性，确保验证码能拦住机器人，不易被破解，检查我的提示词，若我的要求与安全相悖，请提醒我

```bash
● 全部编译通过。以下是实现总结：
     
  ---
  📋 点选验证码功能实现总结
     
  新建文件

  ┌─────────────────────────────────────┬───────────────────────────────────────────────────────────────────────┐
  │                文件                 │                                 说明                                  │
  ├─────────────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
  │ config/captcha.yaml                 │ 验证码配置：元素类型、长度、混淆点数、过期时间                        │
  ├─────────────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
  │ internal/infra/captcha/captcha.go   │ 核心 API：Init()、Create()、Check()，AES-GCM 加解密，数据库存储与清理 │
  ├─────────────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
  │ internal/infra/captcha/generator.go │ 图片生成：字体加载、文字/图标绘制、元素随机生成                       │
  └─────────────────────────────────────┴───────────────────────────────────────────────────────────────────────┘

  修改文件

  ┌─────────────────────────────────┬───────────────────────────────────────────────────────────┐
  │              文件               │                           改动                            │
  ├─────────────────────────────────┼───────────────────────────────────────────────────────────┤
  │ internal/infra/config/config.go │ 新增 CaptchaConfig 结构体及 Config 中的 Captcha 字段      │
  ├─────────────────────────────────┼───────────────────────────────────────────────────────────┤
  │ cmd/api/main.go                 │ 导入 captcha 包并在 database.Init() 后调用 captcha.Init() │
  ├─────────────────────────────────┼───────────────────────────────────────────────────────────┤
  │ go.mod                          │ 新增依赖 golang.org/x/image                               │
  └─────────────────────────────────┴───────────────────────────────────────────────────────────┘

  架构设计

  Create() 流程:
    随机背景 → 生成N个正确元素 → 生成M个混淆元素
    → 打乱全部元素 → 绘制到背景 → Base64编码
    → 正确元素序列(AES-GCM加密) → 存入 Captcha 模型
    → 返回 {key, width, height, elements[], image_base64}

  Check() 流程:
    按key查询 → 检查过期 → 解密正确元素序列
    → 验证元素顺序 → 坐标缩放(适配前端渲染尺寸)
    → 欧几里得距离容差校验 → 按需删除记录

  ⚠️  安全提醒

  1. AES 密钥未持久化：当前 AES 密钥在 Init() 中用 crypto/rand 随机生成，服务器重启后无法解密旧验证码。生产环境建议从配置或环境变量读取固定密钥，如 captcha.encrypt_key。
  2. 元素重叠未检测：当前不检查元素间是否重叠，极端情况下两个元素可能绘制在同一区域，导致用户难以点击。后续可增加碰撞检测。
  3. 坐标加密存储（AES-GCM）、crypto/rand 安全随机、过期自动清理、容差校验等安全措施已就位，可以有效抵御自动化攻击。

✻ Brewed for 5m 31s
```

AI 新建了 `config\captcha.yaml` 配置文件，内容如下：

```yaml
captcha:
  # 验证元素类型，可选值：chinese（中文文字）、uppercase（英文大写字母）、icon（图标）
  elements:
    - chinese
    - uppercase
    - icon
  # 验证元素个数（用户需要按顺序点击的正确元素数量）
  length: 2
  # 混淆元素个数
  confusion_length: 2
  # 验证码默认过期时间（秒）
  expire: 600
```

配置文件得益于精确的提示词，是基本满足预期的，然后 AI 于 `main.go` 中调用了 `captcha.Init()` 进行验证码初始化，这非常不合理，因为验证码和数据库连接等不同，它完全可以按需初始化，而且所谓的初始化，不就是加载配置、清理过期验证码，这些操作都是不用着急的，最终人工整理出以下第二轮需要修改的点：

1. 验证码模块无需于 `main.go` 提前初始化，只调用时初始化即可。
2. 生成随机 `AES` 密钥，并存储于 `captcha.yaml` 配置文件中，无需单独初始化。
3. 无需提前缓存资源文件。
4. 无需定期清理过期验证码，每次使用到验证码模块时，调用一次清理即可。
5. 默认配置已经填充至 `captcha.yaml`，代码中直接读取使用即可，无需再次硬编码这些默认值（`applyDefaults` 是多余的）
6. 增加碰撞检测，防止元素重叠。
7. 将 `背景图` 和 `ICON` 图标资源路径配置也放入配置文件，并在运行时读取，而不是在代码中拼接或硬编码。
8. 将新建的 `internal\infra\captcha\captcha.go` 和 `internal\infra\captcha\generator.go` 尽量精简的整合到一起，并改名为 `internal\infra\captcha\click.go`（点选验证码）
9. 数据库操作，全部改为 `GORM` 的 `Generics API（gorm.G[Model](db)....）`，而不是现在的 `Traditional API`

```bash
● 所有改动完成。以下是调整总结 ......
```

这次打眼一看就舒服多了，因为一共就新增了两个文件，改动仅 `配置加载` 一个文件，然后增加了 `golang.org/x/image` 依赖，主要对 `internal\infra\captcha\click.go` 进行 `review`，此文件代码行数达到了 600+ 行，先尝试精简，单实现点选验证码应该要不了这么多代码，实在不行还是只有拆分。

`review` 时发现一个问题，AI 之所以使用上了 `AES-256` 加解密手段，是为了将 code 字段加密入库，验证时再解密对比，关键是 info 字段并未加密（该字段包括坐标、元素顺序啥都有），所以等于加了个寂寞，我提出了质疑，以下是 AI 的回复：

```bash
你说得对。我重新评估一下：

1. 坐标已经明文存在 Info 字段，加密 Code 没有额外安全收益（有 DB 权限的攻击者直接读 Info 就行）
2. 验证码的安全性主要靠随机位置、混淆元素、过期时间，不是靠加密存储
3. 加密/解密增加了不必要的代码复杂度和密钥管理负担
```

这么不靠谱？算了，就 10 分钟有效期的验证码，而且黑客真拖库了，哪里还需要刷接口；直接取消 AES-256 加密，随便用个 md5(md5(code)) 简单处理一下就可以入库了。

先让 AI 再整理一波代码：

1. 清理多余代码，保留可阅读力的前提下尽力精简代码，比如删去从未使用过的 `max` 函数
2. 取消 code 字段的 AES 加密，改用最简单的 md5(md5(code)) 或 sha1 处理一下就可以入库了
3. ensureInit 函数是多余的，其中 aesKey 解码已去除，`cfg = config.Get().Captcha` 则是需要使用配置数据时，再直接使用 `cfg := config.Get().Captcha` 即可，定期清理过期验证码已经单独调用

#### 封装随机数与文件系统管理包

完成后，由于本次验证码生成已经涉及到随机数生成、文件系统相关了，这两样未来都会比较常用，所以打算先做封装，单独建立对应的包：

- pkg/random/random.go
- pkg/filesystem/filesystem.go

建立以上两个包，让后帮我将 @internal/infra/captcha/click.go 中的一些函数按分类迁移进去比如：
1. cryptoRandInt 迁移至 random，并重写为接受 min、max 两个参数的整数随机数生成方法
2. randomTextColor 迁移至 random，此方法无需再以 random 为前缀

注意：
1. 不通用的函数无需迁移
2. 依赖基础设施的无需迁移（如 `config、database`）
3. 需要迁移的函数总是让它更加通用，易扩展，而不是让它仅服务于单独的模块，比如当前的 captcha
4. 新包中，暂时用不上的方法无需额外建立

```go
// pkg/random/random.go 中，经过人工整理之后，我们得到了以下非常简洁的随机数生成包

package random

import (
	"crypto/rand"
	"image/color"
	"math/big"
	mathrand "math/rand/v2"
)

// Int 返回 [min, max) 范围的加密安全随机整数
// 可用 Int(100, 999) 生成 3 位随机数，以此类推
func Int(min, max int) int {
	if max <= min {
		return min
	}
	n, err := rand.Int(rand.Reader, big.NewInt(int64(max-min)))
	if err != nil {
		return min
	}
	return min + int(n.Int64())
}

// FastInt 返回 [min, max) 范围的快速随机整数
// 可用 FastInt(100, 999) 生成 3 位随机数，以此类推
func FastInt(min, max int) int {
	if max <= min {
		return min
	}
	return min + mathrand.IntN(max-min)
}

// RGB 返回随机 RGB 颜色
// tone:dark=深色,light=浅色
func RGB(tone string) color.Color {
	if tone == "light" {
		return color.RGBA{
			R: uint8(180 + Int(0, 75)),
			G: uint8(180 + Int(0, 75)),
			B: uint8(180 + Int(0, 75)),
			A: 255,
		}
	}
	return color.RGBA{
		R: uint8(20 + Int(0, 80)),
		G: uint8(20 + Int(0, 80)),
		B: uint8(20 + Int(0, 80)),
		A: 255,
	}
}
```

```go
// pkg\filesystem\filesystem.go 文件
package filesystem

import "path/filepath"

// TrimExt 返回去除了路径和扩展名的文件名
// path:文件路径或完整文件名
func TrimExt(path string) string {
	name := filepath.Base(path)
	ext := filepath.Ext(name)
	return name[:len(name)-len(ext)]
}
```