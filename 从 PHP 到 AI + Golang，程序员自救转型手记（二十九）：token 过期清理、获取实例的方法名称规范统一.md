# 从 PHP 到 AI + Golang，程序员自救转型手记（二十九）：token 过期清理、获取实例的方法名称规范统一

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “管理员登录态保持优化、验权中间件”，本期将完成：token 过期清理、获取实例的方法名称规范统一

# token 过期清理

突然发现我们的 token 管理器没有过期清理机制，这会导致 tokens 表无限膨胀，现在对 token 清理功能进行设计：

1. 采用 `写时附带清理` 模式，即：每次写入 `token` 时，都检查一下是否有过期 `token` 可以清理
2. 清理时机仅为 `Create` 时，`Get` 和 `Check` 这类高频调用的方法，不触发清理
3. 请求方法使用独立 `context` 不受请求生命周期影响

AI 在驱动层创建了 `ClearExpired` 方法，然后在 token 管理器的 `Create` 方法中调用，核心代码如下：

**internal\infra\token\driver\database.go 文件：**

```go
func (d *Database) ClearExpired(ctx context.Context) error {
	_, err := gorm.G[model.Token](database.DB()).
		Where("expired_at < ?", time.Now()).
		Delete(ctx)
	return err
}
```

**internal\infra\token\token.go 文件：**

```go
func (m *Manager) Create(ctx context.Context, token *model.Token) error {
	// 清理过期令牌（Create 为低频操作，使用独立 context 不受请求生命周期影响）
	m.cleanExpired()

	token.Token = sha256Hex(token.Token)
	return m.driver.Create(ctx, token)
}

func (m *Manager) cleanExpired() {
	_ = m.driver.ClearExpired(context.Background())
}
```

可以看到 `cleanExpired` 的封装很多余，直接在 `Create` 调用驱动的 `ClearExpired` 不就可以了，这里还是尊重一下 AI，问它是不是要改：

```bash
合理。理由：

1. cleanExpired() 只被 Create 调用一次，是单点使用的薄包装（仅做"用 Background 调一次并忽略错误"），抽象价值低
2. Go 社区倾向避免不必要的间接层（YAGNI），单次调用的简单委托直接内联更清晰
3. captcha 的 cleanExpired 有存在意义——它是包级函数且 bootstrap() 里还耦合了 bootstrapOnce 资源加载；token 这个 cleanExpired 只是一行委托，没有额外逻辑，去掉合理
```

### 可能的疑问

**Q**：`token` 不同于只需要 5 分钟有效期的验证码，有的读者会认为我们需要做一定的延后清理，比如未来有可能需要在短时间内对已过期 token 给与友好提示（比如提示登录态已过期，而不是直接提示请登录）

**A**：本项目的 `AdminAuth` 中间件内，只要在请求时发送了 `token` 值，那么无论 token 是没查到还是过期，都统一提示 `身份认证令牌失效，请重新登录`，所以可以在 token 过期时就立即删除，没差别。

# 获取实例的方法名称统一

1. `internal\infra\config\config.go` 文件配置管理，使用 `Viper` 方法获取全局实例，使用 `Get` 方法获取配置
2. `internal\infra\database\database.go` 数据库初始化，使用 `DB` 方法获取全局数据库实例
3. `internal\infra\token\token.go` token管理器，使用 `Instance` 方法获取全局实例

我直接问了 AI：`我考虑统一一下获取实例的方法名称，或总结一个命名风格，你觉得如何做比较好？`

1. 按返回物命名：`DB()、Client()、Config()` 等，在 `Go` 高星仓库中很常见，例如很多项目用 `db.GetDB()` 或 `database.DB()`。
2. 考虑到你们项目里 `config` 比较特殊——它同时暴露了"解析后的配置结构体"（`Get()`）和"原始 `viper` 引擎"（`Viper()`）——如果强行把三个都统一成同一个名字，反而会让 `config` 的语义变模糊。
3. 建议只把 `token.Instance()` 改成 `token.Manager()`，避免 `token.Instance()` 与其他包风格割裂。

分析和总结的不错，接下来就按第 3 点修改，直接让 CC 改好就行了。