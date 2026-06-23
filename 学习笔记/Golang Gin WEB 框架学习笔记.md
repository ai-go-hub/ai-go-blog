# Golang Gin WEB 框架学习笔记

本文是系列文章《从 PHP 到 AI + Golang，程序员自救转型手记》作者的学习笔记，完整版开源于：[github](https://github.com/ai-go-hub/ai-go-blog) | [gitee](https://gitee.com/ai-go-hub/ai-go-blog)，实操项目 ai-go-mall 开源于：[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)

## 路由

### HTTP 方法

```go
func main() {
    r := gin.Default()
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "message": "pong",
        })
    })
    r.POST("/post", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "method": "POST",
        })
    })

    // 还支持一些不常用的 HTTP 方法（这些方法大多数国内 CDN 都不支持，一般不用）
    // r.PUT
    // r.DELETE
    // r.PATCH
    // r.HEAD
    // r.OPTIONS

    addr := fmt.Sprintf(":%d", cfg.Server.Port)
    if err := r.Run(addr); err != nil {
        log.Fatalf("server start: %v", err)
    }
}
```

### 路径参数和查询字符串参数

```go
r.GET("/ping/:name", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{
        "name": c.Param("name"),
        "age": c.Query("age"),
        // 可设定默认值
        "sex": c.DefaultQuery("sex", "man"),
    })
})
```

### Multipart/Urlencoded 表单

```go
// curl -X POST http://localhost:8080/post -d "name=hello"
r.POST("/post", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{
        "name": c.PostForm("name"),
        "age":  c.DefaultPostForm("age", "18"),
    })
})
```

### Map 作为查询字符串或表单参数

```go
// curl -X POST "http://localhost:8080/post?ids\[a\]=1234&ids\[b\]=hello" -d "names[first]=thinkerou&names[second]=tianou"
r.POST("/post", func(c *gin.Context) {
    ids := c.QueryMap("ids")
    names := c.PostFormMap("names")
    fmt.Printf("ids: %v; names: %v\n", ids, names)
    c.JSON(http.StatusOK, gin.H{
        "ids":   ids,
        "names": names,
    })
})
```

### 文件上传

```go
r.POST("/upload", func(c *gin.Context) {
    file, err := c.FormFile("file")
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    log.Println(file.Filename)

    // 文件夹可自动创建
    dst := filepath.Join("./files/", filepath.Base(file.Filename))
    c.SaveUploadedFile(file, dst)

    c.String(http.StatusOK, fmt.Sprintf("'%s' uploaded!", file.Filename))
})
```

### 路由分组

```go
{
    // /v1/echo
    v1 := r.Group("/v1")
    v1.GET("/echo", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "echo":    "echo ok",
            "version": "v1",
        })
    })
    // v1.Use(中间件)

    // /v1/input
    v1.GET("/input", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "input":   "input ok",
            "version": "v1",
        })
    })

    // 分组还支持嵌套
    posts := v1.Group("/posts")
    posts.GET("/", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "test": "ok",
        })
    })
    posts.GET("/:id", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "test": "ok",
        })
    })
}

v2 := r.Group("/v2")
v2.GET("/echo", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{
        "echo":    "echo ok",
        "version": "v2",
    })
})
```

### 重定向

```go
router.POST("/submit", func(c *gin.Context) {
    c.Redirect(http.StatusFound, "/result")
})
router.GET("/result", func(c *gin.Context) {
    c.String(http.StatusOK, "Redirected here!")
})
```

## 数据绑定

### 模型绑定和验证

```go
// 绑定结构体到 JSON，其中 user 字段是必填的，test 字段必填且长度需要为 10
type Login struct {
    User string `form:"user" json:"user" binding:"required"`
    Test string `form:"test" json:"test" binding:"required,len=10"`
}

// curl -v -X POST "http://127.0.0.1:8080/ping" -H 'content-type: application/json' -d '{ \"user\": \"manu\", \"test\": \"1234567891\" }'
r.POST("/ping", func(c *gin.Context) {
    var login Login
    if err := c.ShouldBindJSON(&login); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "login": login,
    })
})
```

一些细节：
1. `ShouldBindQuery` 可以只绑定查询字符串，完全忽略请求体。
2. `Bind` 系列出错时会由 Gin 直接返回 400 错误，`ShouldBind` 系列由开发者自行处理错误。
3. Gin 可以从多种来源绑定数据：JSON、XML、YAML、TOML、表单数据（URL 编码和 multipart）、查询字符串、URI 参数和请求头。使用相应的结构体标签（json、xml、yaml、form、uri、header）来映射字段。验证规则放在 binding 标签中，使用 go-playground/validator 语法。
4. ShouldBind 会根据 HTTP 方法和 Content-Type 请求头自动选择绑定引擎：
    - 对于 GET 请求，使用查询字符串绑定（form 标签）。
    - 对于 POST/PUT 请求，它会检查 Content-Type——对 application/json 使用 JSON 绑定，对 application/xml 使用 XML 绑定，对 application/x-www-form-urlencoded 或 multipart/form-data 使用表单绑定。
    - 这意味着单个处理函数可以同时接受来自查询字符串和请求体的数据，一般无需手动选择数据源。
5. 标准的绑定方法如 c.ShouldBind 会消费 c.Request.Body，它是一个 io.ReadCloser——一旦读取后就无法再次读取。

### 自定义验证器

```go
type Login struct {
    User string `form:"user" json:"user" binding:"required"`
    Test string `form:"test" json:"test" binding:"required,len=10,testvvv"`
}

// 自定义验证函数
var testValid validator.Func = func(fl validator.FieldLevel) bool {
    data, ok := fl.Field().Interface().(string)
    fmt.Printf("vvv data %s", data)
    if ok {
        if data == "1234567891" {
            return true
        }
    }
    return false
}

// 注册为 testvvv 验证器
if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
    v.RegisterValidation("testvvv", testValid)
}

r.POST("/ping", func(c *gin.Context) {
    var login Login
    if err := c.ShouldBindJSON(&login); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "login": login,
    })
})
```

### 数组的集合格式

绑定数据时，使用 `collection_format` 结构体标签来控制 Gin 如何拆分数组字段的值，此标签可同时用于默认值设定。

- multi（默认）：重复键或逗号分隔的值
- csv：逗号分隔的值
- ssv：空格分隔的值
- tsv：制表符分隔的值
- pipes：管道符分隔的值

```go
type Filters struct {
    Tags      []string `form:"tags" collection_format:"csv"`     // /search?tags=go,web,api
    Labels    []string `form:"labels" collection_format:"multi"` // /search?labels=bug&labels=helpwanted
    IdsSSV    []int    `form:"ids_ssv" collection_format:"ssv"`  // /search?ids_ssv=1 2 3
    IdsTSV    []int    `form:"ids_tsv" collection_format:"tsv"`  // /search?ids_tsv=1\t2\t3
    Levels    []int    `form:"levels" collection_format:"pipes"` // /search?levels=1|2|3
}
```

### 绑定请求头

```go
type testHeader struct {
    Rate   int    `header:"Rate"`
    Domain string `header:"Domain"`
}

r.GET("/ping", func(c *gin.Context) {
    var h testHeader
    if err := c.ShouldBindHeader(&h); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "h": h,
    })
})
```

## 渲染

### JSON 渲染

REST API 和浏览器客户端最常用的选择。使用 `c.JSON()` 进行标准输出，或使用 `c.IndentedJSON()` 在开发时获得可读性更好的格式，Gin 会自动设置相应的 Content-Type。

```go
// gin.H 是 map[string]any 的简写形式，
router.GET("/someJSON", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"message": "hey", "status": http.StatusOK})

    // c.PureJSON 可以输出未经过任何转义的原始 JSON 数据
})
```

### 提供文件

**提供静态文件**
- router.Static(relativePath, root) — 提供整个目录。对 relativePath 的请求会映射到 root 下的文件。例如，router.Static("/assets", "./assets") 会在 /assets/style.css 处提供 ./assets/style.css。
- router.StaticFS(relativePath, fs) — 类似于 Static，但接受一个 http.FileSystem 接口，让你可以更好地控制文件解析方式。当你需要从嵌入式文件系统提供文件或自定义目录列表行为时使用。
- router.StaticFile(relativePath, filePath) — 提供单个文件。适用于 /favicon.ico 或 /robots.txt 等端点。

**从文件提供数据**
- c.File(path) — 从本地文件系统提供文件。内容类型会自动检测。当你在编译时已知确切的文件路径或已验证过路径时使用。
- c.FileFromFS(path, fs) — 从 http.FileSystem 接口提供文件。适用于从嵌入式文件系统（embed.FS）、自定义存储后端提供文件，或当你想限制对特定目录树的访问时。
- c.FileAttachment(path, filename) — 通过设置 Content-Disposition: attachment 头将文件作为下载提供。浏览器会提示用户使用你提供的文件名保存文件，而不管磁盘上的原始文件名。

**流式传输**
go 原生支持大文件的流式传输，使用 `DataFromReader`

## 中间件

### 使用中间件

Gin 支持三个级别的中间件附加：

- 全局中间件 — 应用于路由器中的每个路由。使用 router.Use() 注册。适用于日志记录和 panic 恢复等普遍适用的关注点。
- 分组中间件 — 应用于路由组中的所有路由。使用 group.Use() 注册。适用于将认证或授权应用到路由子集（例如所有 /admin/* 路由）。
- 路由级中间件 — 仅应用于单个路由。作为额外参数传递给 router.GET()、router.POST() 等。适用于路由特定的逻辑，如自定义限流或输入验证。

```go
package main

import (
    "github.com/gin-gonic/gin"
)

func main() {
    router := gin.New()
    router.Use(gin.Logger())
    router.Use(gin.Recovery())
    router.GET("/benchmark", MyBenchLogger(), benchEndpoint)

    authorized := router.Group("/")
    authorized.Use(AuthRequired())
    {
        authorized.POST("/login", loginEndpoint)
        authorized.POST("/submit", submitEndpoint)
        authorized.POST("/read", readEndpoint)

        // nested group
        testing := authorized.Group("testing")
        testing.GET("/analytics", analyticsEndpoint)
    }

    router.Run(":8080")
}
```

**细节**
1. 在中间件或处理函数中启动新的 Goroutine 时，不应该在其中使用原始上下文，必须使用只读副本。

### 自定义中间件

**中间件执行流程**

中间件函数分为两个阶段，由 c.Next() 调用分隔：

- c.Next() 之前 — 此处的代码在请求到达主处理函数之前运行。用于设置任务，如记录开始时间、验证令牌或使用 c.Set() 设置上下文值。
- c.Next() — 调用链中的下一个处理函数（可能是另一个中间件或最终的路由处理函数）。执行在此暂停，直到所有下游处理函数完成。
- c.Next() 之后 — 此处的代码在主处理函数完成后运行。用于清理、记录响应状态或测量延迟。

如果你想完全停止链（例如认证失败时），调用 c.Abort() 而不是 c.Next()。

```go
func LoggerNew() gin.HandlerFunc {
	return func(c *gin.Context) {
		t := time.Now()

		// Set example variable
		c.Set("example", "12345")

		// before request

		c.Next()

		// after request
		latency := time.Since(t)
		log.Print(latency)

		// access the status we are sending
		status := c.Writer.Status()
		log.Println(status)
	}
}

func main() {
	if err := config.Init(); err != nil {
		log.Fatalf("config init: %v", err)
	}

	if err := database.Init(); err != nil {
		log.Fatalf("database init: %v", err)
	}

	cfg := config.Get()

	r := gin.Default()
	r.Use(LoggerNew())

	r.GET("/ping", func(c *gin.Context) {
		var msg struct {
			Name    string
			Message string
			Number  int
			Example string
		}
		msg.Name = "Lena"
		msg.Message = "hey"
		msg.Number = 123
		msg.Example = c.MustGet("example").(string)

		c.JSON(http.StatusOK, msg)
	})

	addr := fmt.Sprintf(":%d", cfg.Server.Port)
	if err := r.Run(addr); err != nil {
		log.Fatalf("server start: %v", err)
	}
}
```

### 依赖注入

#### 闭包模式

最符合 Go 习惯的方式：通过闭包传递依赖。此模式适用于中小型应用。每个处理函数显式声明其依赖。

```go
func PingHandler(db *sql.DB) gin.HandlerFunc {
	return func(c *gin.Context) {
		if err := db.Ping(); err != nil {
			c.JSON(http.StatusServiceUnavailable, gin.H{"error": "database unreachable"})
			return
		}
		c.JSON(http.StatusOK, gin.H{"status": "ok"})
	}
}

r.GET("/ping", PingHandler(db))
```

#### 基于结构体的处理函数

对于有许多共享依赖的应用，将处理函数分组到结构体中。

```go
package main

import (
	"fmt"
	"log"

	"github.com/gin-gonic/gin"
)

type Test struct {
	a string
	b string
	c int
}

func (t *Test) TestFunc(c *gin.Context) {
	// 使用结构体示例中的数据
	fmt.Println(t.a)
	fmt.Println(t.b)
	fmt.Println(t.c)
}

func main() {
	r := gin.Default()

	// 创建结构体实例，并取它的内存地址
	t := &Test{
		a: "aaa",
		b: "bbb",
		c: 123,
	}
	r.GET("/ping", t.TestFunc)

	addr := fmt.Sprintf(":%d", 8080)
	if err := r.Run(addr); err != nil {
		log.Fatalf("server start: %v", err)
	}
}
```

#### 中间件注入

使用中间件将依赖注入到请求上下文中。当许多处理函数需要相同的依赖时很有用。

```go
package main

import (
	"fmt"
	"log"

	"github.com/gin-gonic/gin"
)

func TestMiddleware(test string) gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Set("test", test)
		c.Next()
	}
}

func main() {
	r := gin.Default()

	r.Use(TestMiddleware("test"))

	r.GET("/ping", func(c *gin.Context) {
		test := c.MustGet("test").(string)
		fmt.Printf("test - 1: %s", test)
	})

	r.GET("/ping2", func(c *gin.Context) {
		test := c.MustGet("test").(string)
		fmt.Printf("test - 2: %s", test)
	})

	addr := fmt.Sprintf(":%d", 8080)
	if err := r.Run(addr); err != nil {
		log.Fatalf("server start: %v", err)
	}
}
```

#### 依赖注入模式比较

| 模式 | 类型安全 | 可测试性 | 适用场景 |
|------|----------|----------|----------|
| 闭包 | 编译时 | 容易 mock | 小型应用，少量依赖 |
| 结构体 | 编译时 | 容易 mock | 中大型应用 |
| 中间件 | 运行时 | 中等 | 横切关注点，共享状态 |

## 日志

默认日志记录器写入 os.Stdout（就是每次请求终端输出的、一行一行的带请求路径的内容）

```go
func main() {
    // Disable Console Color, you don't need console color when writing the logs to file.
    gin.DisableConsoleColor()

    // Logging to a file.
    f, _ := os.Create("gin.log")
    gin.DefaultWriter = io.MultiWriter(f)

    // Use the following code if you need to write the logs to file and console at the same time.
    // gin.DefaultWriter = io.MultiWriter(f, os.Stdout)

    router := gin.Default()
    router.GET("/ping", func(c *gin.Context) {
        c.String(200, "pong")
    })

    router.Run(":8080")
}
```

其他细节：
1. 支持自定义日志格式
2. 支持特定路径跳过日志记录
3. 控制日志输出着色
4. Gin 推荐使用第三方库，结合日志系统使用，比如日志轮转库 `lumberjack`，结构化日志库 `slog`