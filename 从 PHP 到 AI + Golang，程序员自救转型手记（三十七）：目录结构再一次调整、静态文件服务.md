# 从 PHP 到 AI + Golang，程序员自救转型手记（三十七）：目录结构再一次调整、静态文件服务

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “多驱动上传接口”，本期将完成：目录结构再一次调整、静态文件服务

# 目录结构再一次调整

> 文件上传之后，就是获取文件完整路径 `FullUrl` 函数，在此之间，需要先获取 `当前域名、端口、协议` 等数据，因此方向又短暂的切换到 `http` 相关。

搜索了大量资料，现在建立 `internal/kit`（工具集、成套工具箱）目录，然后先将 `internal/response` 包改名为更大的 `httpx` 移入其中。

这样做的原因是：`internal/kit` 的存储范围更广，且可以包含 `internal/response`，达到：`internal` 目录下文件夹数量不增加，扩展性加强的目的。

之所以不是增加 `internal/common`、`internal/helper` 包，是因为这种名称范围非常广的文件夹，很容易成为未来的超级垃圾堆，go 社区不推荐（推荐的就是你在建包时，研究半天放哪儿😂，好在现在有 AI，且我们的目录结构已经非常完善了）。

现在我们有 `internal/kit`（成套工具箱）、`internal/infra`（基础设施），外层还有不依赖任何基础设施的 `pkg`、`pkg/util`，应该是够了。

### internal/kit/httpx 包

接下来创建 `internal/kit/httpx` 包，将 `internal/response/response.go` 移入其中（作为 `internal/kit/httpx/response.go`），从现在起，创建响应：

```go
// 原来的
response.Fail()
response.Success()
response.New(c).Code(...).Message(...).Send()

// 改为
httpx.Fail()
httpx.Success()
httpx.New(c).Code(...).Message(...).Send()
```

### 获取当前域名、端口、协议等

还是在 `kit` 目录的 `httpx` 包以内，建立 `context.go` 文件，里边建立 `BaseURL、Scheme、Port、ClientIP、ContentType、Referer、UserAgent` 函数，用于从 `c *gin.Context` 获取相应数据。

额外完善一下 `server.trusted_proxies`（可信代理）配置项，以便 `httpx.ClientIP()` 能够完美的工作（不理解可以在 `gin` 文档搜索可信代理）。

# 提供静态文件

直接使用 `Gin` 的 `router.Static(relativePath, root)`，即可创建好静态文件服务，我们将 `根目录/static` 整个目录开放出去，表现大概是：

- GET `/upload/2026/xxx.jpg` → `./static/upload/2026/xxx.jpg`
- GET `/img/logo.png` → `./static/img/logo.png`
- GET `/ping`、GET `/common/upload` 等业务路由优先命中，不受影响
- 找不到文件时 `http.FileServer` 会返回 `404`，与 `Gin` 默认 `404` 行为一致

按照我们项目的规划，将静态文件服务的创建逻辑放至 `internal\router\static.go` 文件中即可。
