# 从 PHP 到 AI + Golang，程序员自救转型手记（四）：整合 Gin 框架、必学项 + 2

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

> 此文发布时已经写到【从 PHP 到 AI + Golang，程序员自救转型手记（二十一）：登录接口整合点选验证码，AI 大翻车】已经完成了项目基本架构和前几个接口，不用担心作者弃坑，请放心阅读。

在上一期，我们已经完成 “数据库连接，读写分离配置”，本期将完成：整合 Gin 框架、必学项 + 2

# 整合 Gin 框架

作为 WEB 项目，使用一个成熟的 WEB 框架会为我们省很多事，经过精心对比选型，我决定使用 [Gin](https://github.com/gin-gonic/gin)，它在 github 上有多达 88.5k 的 star，是目前 golang 语言 WEB 最受欢迎的框架，如果你需要高性能和高效开发，你一定会爱上 Gin。

先让 cc 完成 Gin 的整合：
1. WEB 框架使用 Gin（https://github.com/gin-gonic/gin）
2. 帮我于 main.go 内写好第一个测试路由并启动开发服务器，服务器端口使用 server.port 配置项

> PS：作者安装依赖是一般都会带上 github 或其他官方链接，这是很有必要的，主要避免 AI 安装了盗版依赖包，比如某个黑客写了个 https://github.com/hacker/air-true，并全网发布各类教程/宣发文章，此时若我们的提示词不带官方仓库链接，AI 可能会联网搜索 golang air 依赖，随即被网文所骗，且它还能正常运行，只是额外携带了病毒代码，半夜用你的电脑/服务器挖坑或当做肉鸡使用。

自动安装完依赖之后，只修改了 `main.go` 文件，的内容如下：

```go
package main

import (
	"fmt"
	"log"
	"net/http"

	"ai-go-mall/config"
	"ai-go-mall/internal/database"

	"github.com/gin-gonic/gin"
)

func main() {
	if err := config.Init(); err != nil {
		log.Fatalf("config init: %v", err)
	}

	if err := database.Init(); err != nil {
		log.Fatalf("database init: %v", err)
	}

	cfg := config.Get()

	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"message": "pong",
		})
	})

	addr := fmt.Sprintf(":%d", cfg.Server.Port)
	if err := r.Run(addr); err != nil {
		log.Fatalf("server start: %v", err)
	}
}
```

可以看到注册了一个 ping 路由，执行 `go run cmd/api/main.go` 启动服务后，浏览器就可以访问 `http://localhost:8080/ping` 了。

# 必学项 + 2

接下来就是继续深造了，磨刀不误砍柴工，同必需学习 go 语法的理由一样，我们此时需要先完成通读 Gin 框架文档的工作，还有 GORM 的文档，配置管理用的 Viper 到是几乎没什么需要深入了解的地方（想了解用了 cc 为什么还要学基础的朋友请转到第一期，需要快速学习教程的朋友也可以转到第一期）。

- Gin 的文档目录很多，但每一篇的内容其实是很短的，过一遍很快，基于已经搭好的架子，快速写一些 demo 很快就过完了，作者这里还是整理了笔记，共 553 行，开源于：[github](https://github.com/ai-go-hub/ai-go-blog) | [gitee](https://gitee.com/ai-go-hub/ai-go-blog)
- GORM 也是必须深入学习的，官方文档中的查询和高级查询两章非常长，这两篇就花了作者一天时间，但是非常值得，对 golang 的语法也更加熟悉了，GORM 还支持 Auto Migration，写好结构体，就能自动建立数据表，作者学习笔记同样已开源，共 1290 行，开源地址同上。

从我以往的学习经验来看，这两关过了后面就简单了，大多是 Viper 这种级别的依赖，不需要全面学习，使用时再查文档即可。