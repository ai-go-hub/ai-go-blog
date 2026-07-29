# 从 PHP 到 AI + Golang，程序员自救转型手记（四十）：基类改造

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “前端上传组件实现”，本期将完成：基类改造

# 基类改造

> 基类严格来讲不是 go 里边的概念，在本 blog 中使用它范指 基控制器、基服务、基仓储

接下来就是做出第一个后台的管理功能了，在此之前，对基类进行完善；

这是一个及其需要细心和周全考虑的工作，各大基类的每一行代码，都需要是最佳实践，扩展性拉满，性能拉满。

基类改造 + 前端表格组件一起，作者整整花了将近两周时间，基类改造情况如下（每一个小节一般就是提交给 AI 的一个需求，一般一个需要需要多轮对话才能完成，然后边修改边人工 `review`，最终再完成测试）：

## 基仓储改造

### 利用 gorm.Statement.Schema 检查字段是否存在

必需特意写明利用 `gorm.Statement.Schema`；

因为直接告诉 `AI` 需要一个 `检查字段是否存在` 的方法的话，它会根据当前模型，写一大堆代码预解析出 `Schema` 并缓存；

我测试了一下这种方案还是各大 AI 模型的首选方案，实际上当然是没有必要的，直接伸手向 `GORM` 拿就行了。

最终 `检查字段是否存在` 方法的签名为：`func (r *Repository[T]) FieldExists(field string) (bool, error)`

### 获取主键字段方法

依旧是利用 `gorm.Statement.Schema` 获取主键字段，调用 `GORM Statement.Parse` 方法解析模型数据，该函数内部会复用 `GORM` 的 `Schema` 缓存。

复合主键模型返回 `GORM` 识别的优先主键字段，最终方法签名为：`func (r *Repository[T]) PrimaryKeyField() (string, error)`

代码如下：

```go
// FieldExists 检查当前模型是否包含指定字段
// 同时支持 Go 字段名和数据库列名；Statement.Parse 会复用 GORM 的 Schema 缓存
func (r *Repository[T]) FieldExists(field string) (bool, error) {
	field = strings.TrimSpace(field)
	if field == "" {
		return false, nil
	}

	stmt := &gorm.Statement{DB: r.DB()}
	if err := stmt.Parse(new(T)); err != nil {
		return false, err
	}
	return stmt.Schema.LookUpField(field) != nil, nil
}
```

### 增加 Count 方法

此方法用于获取数据条数，签名为 `func (r *Repository[T]) Count(c *gin.Context, opts Options) (int64, error)`，使用 `opts.Scopes`，忽略 `opts` 中的 `分页、排序、字段选择` 等。

### 已有方法改造

先定义一个所有方法通用的 `Options` 结构体：

```go
// Options 通用仓库操作选项（各选项可按需使用）
type Options struct {
	Omit       []string                // 排除出入库字段，将在 Create、Update、List 等方法中应用至 GORM 的 Select 方法
	Select     []string                // 选择出入库字段，其余同上
	Scopes     []func(*gorm.Statement) // 将在 List 和 Get 等方法中直接传递给 GORM 的 Scopes 方法，可以自定义 查询、排序、分页 等等
	PrimaryKey string                  // 主键值，提供则会在 Get 方法中作为 Where 的参数
}
```

#### 基仓储 List 改造

丰富入参和功能，如下：

```go
// 原来的
func (r *Repository[T]) List(c *gin.Context, scopes ...func(*gorm.Statement)) ([]T, error)

// 改为
func (r *Repository[T]) List(c *gin.Context, opts Options) ([]T, error)
```

即：将 `scopes` 参数改为 `opts`，`opts` 里边可以传递 `scopes`，并且额外还可以传递 `排序字段、选择排除字段` 等选项，未来要扩展也很方便；

`List` 方法内，根据 `opts` 确定如何读取数据；另外，对于 `Select` 和 `Omit` 字段，我并没有使用 `FieldExists` 方法对字段名逐一确定是否存在（开发者需自行对传入的字段名负责，程序不会对未知字段报警）。

#### 基仓储 Create / Update / Get 改造

仓储层的 `Create / Update / Get` 方法，也需要增加 `opts Options` 参数，并应用上这些配置。

在 `Create / Update` 方法内：`Select / Omit` 配置决定 `入库字段`。

在 `Get` 方法内，不仅可以使用 `Select / Omit` 来决定 `出库字段`，也支持 `scopes` 和 `PrimaryKey`（主键值）选项；若提供了 `PrimaryKey`，`Get` 会拼接类似这样的查询条件：`Where(pk+" = ?", opts.PrimaryKey)`，同时并不会忽略 `scopes`。

## 基控制器、基服务改造

首先是整体规划的改变，比如控制器要支持更多的请求数据，本次改造支持了 `排序、分页、多字段条件查询` 等；

特别是服务层的 `List` 方法，首先看基仓储的 `List` 方法，它太简单了，如下：

```go
func (r *Repository[T]) List(c *gin.Context, opts Options) ([]T, error) {
	q := gorm.G[T](r.DB()).Scopes(opts.Scopes...)

	// 出库字段的选择与忽略
	if len(opts.Select) > 0 {
		q = q.Select(strings.Join(opts.Select, ","))
	}
	if len(opts.Omit) > 0 {
		q = q.Omit(opts.Omit...)
	}

	return q.Find(c.Request.Context())
}
```

可以看到仓储层主要是接受了一个 `Scopes` 函数切片，这意味着我们前端传递的 `排序、分页、多字段条件查询` 等，都需要在服务层构建为 `Scopes`，然后直接传递给仓储层使用（`Select` 和 `Omit` 逻辑为和其他方法保持统一，并未使用 `Scopes` 的方式定义）。

#### 基控制器接受主键的方式优化

旧的主键接受，固定接受为 `int` 类型，如下：

```go
id, err := strconv.ParseUint(c.Param("id"), 10, 64)
```

新的接受方式，首先变量改名为更加通用的 `pk`，然后直接接受为 `string`，并以 `string` 类型直接传递至 服务、仓储 等各层，如下：

```go
pk := c.Param("pk")
```

> 只在需要做比较、计算时，再根据当前模型的主键类型转为 int 即可，且一般无需转换，因为对于 GORM 来说，主键传的是 int 或 string，生成的 SQL 都是一样的。

#### 服务层 List 方法响应字段增加

`list` 接口目前直接响应单个 `data` 字段，内容是对应模型中的数据；

需要改为 `data.list`（数据）、`data.total`（数据条数，使用仓储层的 `Count` 方法）

#### 分页支持

基控制器额外接受 `page` 和 `limit` 参数，传递给服务层，由服务层建立一个 `Paginate Scopes`，最终传递给仓储的 `List` 方法（该方法支持 `Scopes`，能直接使用分页器）。

排序支持基本同上。

#### 多字段条件查询支持

对应基服务（`internal/service/base.go`）的 `BuildWhereScopes` 方法，最终会返回多个 `scopes`。

前端传递的查询条件是这样的：

```json
{
    "wheres": [
        {
            "field": "username",
            "operator": "NOT ILIKE",
            "value": "test",
        }
    ]
}
```

`BuildWhereScopes` 会根据以上前端传值组装 `where`，实现起来非常简单，主要是 `operator` 它代表 SQL 运算符号，需要支持很多，比如 `=,>,<,LIKE,ILIKE,IS NULL` 等，这里有一个小坑，`=,>,<` 这类符号在传输时，很可能被转义掉，这里需要写一个转换函数，如下：

```go
// GetOperatorByAlias 符号类运算符别名 → SQL 运算符
func GetOperatorByAlias(op string) string {
	switch op {
	case "eq", "":
		return "="
	case "ne":
		return "!="
	case "gt":
		return ">"
	case "gte":
		return ">="
	case "lt":
		return "<"
	case "lte":
		return "<="
	default:
		return op // LIKE、IN、NOT IN 等单词运算符直接透传
	}
}
```

然后前端使用 `operator: eq`、`operator: ne` 等就行了，不要使用原始符号。

> 未来还会再次完善，支持字段白名单、操作符号白名单、OR 条件等

## 测试

#### IS NOT NULL 无效

在人工测试中，找到一个 `review` 没发现的问题，`list` 接口筛选数据时，操作符使用 `IS NOT NULL` 无效，查看了代码，发现 AI 居然犯了个很低级的错误：

```go
switch op {
case "IS NULL", "IS NOT NULL":
    scopes = append(scopes, func(stmt *gorm.Statement) {
        stmt.Where(w.Field + " IS NULL")
    })
}
```

操作符号为 `IS NULL` 和 `IS NOT NULL` 时：都是固定的 `stmt.Where(w.Field + " IS NULL")`，改为 `stmt.Where(w.Field + " " + op)` 才对。

#### 不兼容范围查询

```go
switch op {
case "BETWEEN":
    scopes = append(scopes, func(stmt *gorm.Statement) {
        stmt.Where(w.Field+" BETWEEN ? AND ?", w.Value)
    })
}
```

如上，`w.Value` 只有一个，但是 `BETWEEN` 需要一个范围值，解决方案也很简单，前端传递数组，或简单的以 `,` 分割多个值，后端这边再切一下即可。