# 从 PHP 到 AI + Golang，程序员自救转型手记（四十七）：列表通用排序接口实现（增量重排法）

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “再次优化基类”，本期将完成：列表通用排序接口实现（增量重排法）

# 列表通用排序接口实现（增量重排法）

## 实现初版的提示词

继续啃 `BuildAdmin` 的老本：

参考 `@../badmin-v2.3.7-full/app/admin/library/traits/Backend.php` 中的 `sortable` 方法，在当前项目的基类（`@internal/handler/base.go` 和 `@internal/repository/base.go` 和 `@internal/service/base.go`）中，实现 `sort` 方法，并注册路由，前端传递的参数有：

```bash
move: moveRow[table.pk!],                               // 移动行
target: targetRow[table.pk!],                           // 目标行
sort: table.filter?.sort,                               // 排序字段（权重字段）
order: table.filter?.order,                             // 排序方式
direction: evt.newIndex > evt.oldIndex ? 'down' : 'up', // 拖拽方向
```

- 忽略参考代码中的 `$dataLimitAdminIds`
- 忽略参考代码中的 “当前是否以权重字段排序” 检查（即：只检查当前排序和默认排序字段，不检查有序保证字段）

## 功能需求分析

前端拖动排序，如：将权重为 `1` 的行（`排序行`），拖动至权重为 `5` 的行（`目标权重行` 或 `目标行`）

所谓 `增量重排法`，也可以称做 `区间位移法`：实现的是将涉及到的行，权重值全部按算法重设，以达到视觉上的排序调整

比起全量重排，增量重排法影响范围更小，比起交换重排，重排后的顺序更加合理。

难点/注意点如下：

1. 复用 `List` 列表方法的排序和过滤规则，确保执行重新排序时，查得的数据列表和前端的一样。
2. 只支持以 `weigh`（权重）字段排序时，进行重排操作（字段名是可以自定义的；只使用权重字段，是因为重排是修改权重字段值实现的，假设以 `创建时间` 排序，重排功能不可能去修改创建时间）。
3. 用户当前可能以 `weigh asc` 和 `weigh desc` 两种方式进行排序，并且可能将排序行 `向上拖` 或者 `向下拖`，需要全部考虑到位。
4. 目标行的权重值，不只一行：比如目标行权重为 `5`，但数据表中权重值为 `5` 的行有很多。
5. 不要一个 SQL 修改一行，而是尽量一个 SQL 完成一类权重值的全部修改：比如 `权重值 > 目标权重` 的行全部 `+1`，不能一条一条的遍历去改。

## AI 实现后

控制器层的代码很简单，主要是新增了 `Sort` 方法及其请求体结构体定义，并注册了 `sort` 路由。

主要功能实现是在服务层，所以服务层问题也最多：

### weigh 字段名不固定

实现排序功能期间，需要读取目标行的权重值，但字段名是不固定的，go 里边没有 `$row[$weigh]` 这种写法，所以 AI 使用了反射的方式去读取权重值，反射首先是实现复杂，二是性能一般（拖拽排序其实无需考虑性能），但我们还是直接改为从前端传递 `目标行权重值` 即可，服务端直接读取并使用，去掉反射相关代码。

AI 根据需求实现后，接受的前端变量名为 `target_weigh`，由于我们服务端不需要使用 `排序行` 的权重值，只需要 `目标行` 的权重值，这里将 `target_weigh` 改名为更简洁的 `weigh` 即可。

并且，AI 将 `weigh` 的类型定义为 `any`，这里直接固定为 `int64`。

### 改为 Generics API

在这种需要比较复杂的 update 语句时，AI 又开始忘记 `AGENTS.md` 中的规则了，这里强行要求它改为 `Generics API`：

```go
// 原来的
if err := tx.Model(new(T)).
    Where(weighField+" "+bulkOp+" ?", weigh).
    Where(pkField+" <> ?", move).
    UpdateColumn(weighField, gorm.Expr(weighField+" "+updateOp+" ?", weighRowsCount)).Error; err != nil {
    return err
}

// 改为
gtx := gorm.G[T](tx)

if err := gtx.Where(weighField+" "+bulkOp+" ?", weigh).
    Where(pkField+" <> ?", move).
    UpdateColumn(weighField, gorm.Expr(weighField+" "+updateOp+" ?", weighRowsCount)); err != nil {
    return err
}

// 且 gtx 后续复用
```

另外，重排实现内，必需使用一下传统 API 的 `Pluck` 方法，当然也可以使用 `Generics API` 的 `Select` 方法去选择字段，只是这样又需要 `反射` 读取结构体中数据库列名对应的字段值了，没必要。

### 反转切片代码优化

实现重排时，有一段反转数组的代码：

```go
// 向下拖动时反转，保证等权重区间内相对顺序不变
if direction == "down" {
    for i, j := 0, len(weighIDs)-1; i < j; i, j = i+1, j-1 {
        weighIDs[i], weighIDs[j] = weighIDs[j], weighIDs[i]
    }
}
```

在 `Go 1.25 +` 提供了 `slices.Reverse` 方法，可以直接使用，简化为：

```go
// 向下拖动时反转，保证等权重区间内相对顺序不变
if direction == "down" {
    slices.Reverse(weighIDs)
}
```

### 多余的封装

AI 帮忙封装了一个 `exprOffset` 函数，用于组装一些排序用的查询表达式，但其对整体无益，且不被 `Sort` 方法以外的地方复用，这里让它直接改为：在 `Sort` 方法内封装为闭包函数即可，不要放在 `Sort` 方法外边。

### BUG

我在测试前，首先让 AI 直接再写一点调试代码，打印出 `Sort` 方法内执行的所有 `SQL`，然后前端进行操作，对比我的需求与执行的 `SQL`，发现一处排序错误；

代码我们已经 `review` 过了，没有发现明显问题，应该是某处细节上的逻辑错误，这里直接把日志给 AI，并加以描述：

提示词：好像有 BUG，排序后有权重值重复的行，我有权重值为 `1,2,3` 的行，入参 `move=1 target=3` 时，执行的 `SQL` 有以下：

```bash
2026/07/25 00:19:34 C:/Users/yang/Desktop/admin/internal/service/base.go:148
[0.709ms] [rows:1] SELECT "id" FROM "admin_rules" WHERE weigh = 3 ORDER BY weigh DESC,id DESC

2026/07/25 00:19:34 C:/Users/yang/Desktop/admin/internal/service/base.go:171
[0.525ms] [rows:0] UPDATE "admin_rules" SET "weigh"=weigh - 1,"updated_at"='2026-07-25 00:19:34.578' WHERE weigh > 3 AND id <> '1'

2026/07/25 00:19:34 C:/Users/yang/Desktop/admin/internal/service/base.go:201
[0.628ms] [rows:1] UPDATE "admin_rules" SET "weigh"=3,"updated_at"='2026-07-25 00:19:34.578' WHERE id = '1'

2026/07/25 00:19:34 C:/Users/yang/Desktop/admin/internal/service/base.go:213
[0.541ms] [rows:1] UPDATE "admin_rules" SET "weigh"=2,"updated_at"='2026-07-25 00:19:34.579' WHERE id = '3'
```

最终结果是：有两行的权重值均为 `2`，这与目标不符，应该不出现重复的权重才对，对比参考代码：`@../badmin-v2.3.7-full/app/admin/library/traits/Backend.php` 的 `sortable` 方法，找出原因：

找到原因了。对比 `PHP` 代码的 `bulk update` 逻辑：

PHP（正确）：

```php
// updateMethod == 'dec' → WHERE weigh < target
// updateMethod == 'inc' → WHERE weigh > target
->where(weigh, $updateMethod == 'dec' ? '<' : '>', $weigh)
```

Go（错误）：

```go
// updateOp == "-" → bulkOp = ">"  ← 反了！
// updateOp == "+" → bulkOp = "<"
bulkOp := ">"
if updateOp == "+" {
    bulkOp = "<"
}
```

dec（权重 −1）应该影响权重比目标小的行（<），让它们再减，给拖动行腾位置。我把 > 和 < 写反了。


> 嚯，AI 居然还会犯这种逻辑错误。