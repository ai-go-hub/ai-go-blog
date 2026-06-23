# Golang GORM 库学习笔记

本文是系列文章《从 PHP 到 AI + Golang，程序员自救转型手记》作者的学习笔记，完整版开源于：[github](https://github.com/ai-go-hub/ai-go-blog) | [gitee](https://gitee.com/ai-go-hub/ai-go-blog)，实操项目 ai-go-mall 开源于：[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)

GORM 官方中文文档：[https://gorm.io/zh_CN/](https://gorm.io/zh_CN/)

## 模型定义

模型是使用普通结构体定义的。这些结构体可以包含具有基本Go类型、指针或这些类型的别名，甚至是自定义类型。

声明一个结构体，gorm 可将结构体映射到数据库表：

```go
type User struct {
	ID           uint // 名为 id 的字段自动作为主键
	Name         string // 普通字符串字段
	Mobile       *string // 指针类型的字符串，将允许值为 null
	MemberNumber sql.NullString // 显式指定字符串可为 null
	CreatedAt    time.Time // 创建时间字段
}
```

### 约定
1. 主键：GORM 使用一个名为ID 的字段作为每个模型的默认主键。
2. 表名：默认情况下，GORM 将结构体名称转换为 snake_case 并为表名加上复数形式。`User > users`，`GormUserName > gorm_user_names`。
3. 列名：GORM 自动将结构体字段名称转换为 snake_case 作为数据库中的列名。
4. 时间戳字段：GORM使用字段 CreatedAt 和 UpdatedAt 来自动跟踪记录的创建和更新时间。

### gorm.Model

GORM 提供了一个预定义的结构体，名为 `gorm.Model`，其中包含常用字段，可用将其嵌入在自定义的结构体中：

```go
// gorm.Model 的定义
type Model struct {
	ID        uint `gorm:"primaryKey"`
	CreatedAt time.Time
	UpdatedAt time.Time
	DeletedAt gorm.DeletedAt `gorm:"index"`
}

// 嵌入到自己的结构体
type Author struct {
    gorm.Model
    Name  string
    Email string
}
```

### 字段级权限控制

可导出的字段（大写开头的字段）在使用 GORM 进行 CRUD 时拥有全部的权限，此外，GORM 允许您用标签控制字段级别的权限。这样您就可以让一个字段的权限是只读、只写、只创建、只更新或者被忽略。

```go
type User struct {
	Name string `gorm:"<-:create"`          // 允许读和创建
	Name string `gorm:"<-:update"`          // 允许读和更新
	Name string `gorm:"<-"`                 // 允许读和写（创建和更新）
	Name string `gorm:"<-:false"`           // 允许读，禁止写
	Name string `gorm:"->"`                 // 只读（除非有自定义配置，否则禁止写）
	Name string `gorm:"->;<-:create"`       // 允许读和写
	Name string `gorm:"->:false;<-:create"` // 仅创建（禁止从 db 读）
	Name string `gorm:"-"`                  // 通过 struct 读写会忽略该字段
	Name string `gorm:"-:all"`              // 通过 struct 读写、迁移会忽略该字段
	Name string `gorm:"-:migration"`        // 通过 struct 迁移会忽略该字段
}
```

> 使用 GORM Migrator 创建表时，不会创建被忽略的字段。

### 字段标签

多个标签使用 `;` 分隔。

md
| 标签名 | 说明 |
|--------|------|
| column | 指定 db 列名 |
| type | 列数据类型，推荐使用兼容性好的通用类型，例如：所有数据库都支持 bool、int、uint、float、string、time、bytes 并且可以和其他标签一起使用，例如：not null、size, autoIncrement… 像 varbinary(8) 这样指定数据库数据类型也是支持的。在使用指定数据库数据类型时，它需要是完整的数据库数据类型，如：MEDIUMINT UNSIGNED not NULL AUTO_INCREMENT |
| serializer | 指定将数据序列化或反序列化到数据库中的序列化器, 例如: serializer:json/gob/unixtime |
| size | 定义列数据类型的大小或长度，例如 size: 256 |
| primaryKey | 将列定义为主键 |
| unique | 将列定义为唯一键 |
| default | 定义列的默认值 |
| precision | 指定列的精度 |
| scale | 指定列大小 |
| not null | 指定列为 NOT NULL |
| autoIncrement | 指定列为自动增长 |
| autoIncrementIncrement | 自动步长，控制连续记录之间的间隔 |
| embedded | 嵌套字段 |
| embeddedPrefix | 嵌入字段的列名前缀 |
| autoCreateTime | 创建时追踪当前时间，对于 int 字段，它会追踪时间戳秒数，您可以使用 nano/milli 来追踪纳秒、毫秒时间戳，例如：autoCreateTime:nano |
| autoUpdateTime | 创建/更新时追踪当前时间，对于 int 字段，它会追踪时间戳秒数，您可以使用 nano/milli 来追踪纳秒、毫秒时间戳，例如：AutoUpdateTime:milli |
| index | 根据参数创建索引，多个字段使用相同的名称则创建复合索引，查看 索引 获取详情 |
| uniqueIndex | 与 index 相同，但创建的是唯一索引 |
| check | 创建检查约束，例如 check:age > 13，查看 约束 获取详情 |
| <- | 设置字段写入的权限， <-:create 只创建、<-:update 只更新、<-:false 无写入权限、<- 创建和更新权限 |
| -> | 设置字段读的权限，->:false 无读权限 |
| - | 忽略该字段，- 表示无读写，-:migration 表示无迁移权限，-:all 表示无读写迁移权限 |
| comment | 迁移时为字段添加注释 |

## 迁移 AutoMigrate

`AutoMigrate` 可以根据模型定义（结构体）创建数据表。

当结构体改变时，比如新增了外键、约束、列和索引等，`AutoMigrate` 同样能完成表结构更新。但出于保护您数据的目的，它 **不会** 删除未使用的列。

```go
type User struct {
	ID           uint // 名为 id 的字段自动作为主键
	Name         string // 普通字符串字段
	Mobile       *string // 指针类型的字符串，将允许值为 null
	MemberNumber sql.NullString // 显式指定字符串可为 null
	CreatedAt    time.Time // 创建时间字段
}

db.AutoMigrate(&User{})

// 创建多个表
db.AutoMigrate(&User{}, &Product{}, &Order{})

// 创建表时添加后缀
db.Set("gorm:table_options", "ENGINE=InnoDB").AutoMigrate(&User{})
```

### Migrator 接口

Migrator 接口提供了一系列操作数据表的方法，创建表/删除表，列管理，索引管理等。

```go
// 检查表是否存在
db.Migrator().HasTable(&User{})

// 删除表
db.Migrator().DropTable(&User{})

// 创建表
db.Migrator().CreateTable(&User{})

// ...
```

## 增删改查

建议使用 Generics API，编译期类型安全，Traditional API 是运行时类型检查，出错直接 panic

```go
type User struct {
	ID        uint
	Name      string `gorm:"comment:用户名"`
	Email     sql.NullString
	Mobile    *string
	CreatedAt time.Time
}

db := database.GetDB()
ctx := context.Background()
```

### 创建

通过 `user.ID` 的语法，可访问已插入的数据

```go
mobile := "18983076450"
user := User{Name: "Jinzhu", Mobile: &mobile}

err := gorm.G[User](db).Create(ctx, &user)
if err != nil {
	log.Fatalf("create user: %v", err)
}

// 批量创建
users := []User{
	{Name: "jinzhu_1"},
	{Name: "jinzhu_2"},
}
db.Create(&users)
```

### 查询

#### 基本查询

```go
// SELECT * FROM users ORDER BY id LIMIT 1
user, err := gorm.G[User](db).First(ctx)

// SELECT * FROM users LIMIT 1
user, err := gorm.G[User](db).Take(ctx)

// SELECT * FROM users ORDER BY id DESC LIMIT 1
user, err := gorm.G[User](db).Last(ctx)

// SELECT * FROM users WHERE id = 10
user, err := gorm.G[User](db).Where("id = ?", "10").First(ctx)

// 多行
// SELECT * FROM users WHERE id IN (1,2,3)
users, err := gorm.G[User](db).Where("id IN ?", []int{1, 2, 3}).Find(ctx)

// 查询全部
users, err := gorm.G[User](db).Find(ctx)

// 查询特定字段
Select("name", "age")
Select([]string{"name", "age"})

// 排查字段
Omit("name", "age")
```

#### 查询条件

```go
// AND
users, err := gorm.G[User](db).Where("name = ? AND id >= ?", "jinzhu_1", "1").Find(ctx)

// BETWEEN
Where("created_at BETWEEN ? AND ?", lastWeek, today)

// Struct 条件，不支持 0 值（zero values）
Where(&User{Name: "jinzhu", Id: 20})
Where(&User{Name: "jinzhu"}, "name", "Id")

// Map 条件，支持 0 值（zero values）
Where(map[string]any{"name": "jinzhu", "age": 20})

// 主键切片
Where([]int64{20, 21, 22})

// 内联条件
Generics API 不支持

// NOT 条件
users, err := gorm.G[User](db).Not("id = ?", "37").Where("name = ? AND id >= ?", "jinzhu_1", "1").Find(ctx)

// NOT IN
Not(map[string]any{"id": []int{37, 38}})

// Struct NOT
Not(User{Name: "jinzhu", Age: 18})

// 主键切片 NOT
Not([]int64{1,2,3})

// OR
Or("role = ?", "super_admin")

// Struct OR
Or(User{Name: "jinzhu 2", Age: 18})

// Map OR
Or(map[string]any{"name": "jinzhu 2", "age": 18})
```

#### 其他条件

**排序**

```go
// SELECT * FROM users ORDER BY id desc, name desc
Order("id desc, name desc")

// SELECT * FROM users ORDER BY age desc, name
Order("age desc").Order("name")
```

**分页**

```go
// SELECT * FROM users LIMIT 3
Limit(3)

// SELECT * FROM users OFFSET 3
Offset(3)

// SELECT * FROM users OFFSET 5 LIMIT 10
Limit(10).Offset(5)
```

**Group By & Having**

```go
// SELECT name, sum(age) as total FROM users WHERE GROUP BY name
Select("name, sum(age) as total").Group("name")

// SELECT name, sum(age) as total FROM users GROUP BY name HAVING sum(age) > 10
Select("name, sum(age) as total").Group("name").Having("total > ?", 10)
```

**去重**

```go
Distinct("name", "age")
```

**Joins**

```go
type Result struct {
	Name   string
	UserID uint
}

var results []Result

// SELECT users.name, user_logs.user_id FROM `users` left join user_logs on user_logs.user_id = users.id
err := db.Table("users").
	Select("users.name, user_logs.user_id").
	Joins("LEFT JOIN user_logs ON users.id = user_logs.user_id").
	Find(&results).Error

// 多个 join
db.
Joins("JOIN emails ON emails.user_id = users.id AND emails.email = ?", "jinzhu@example.org").
Joins("JOIN credit_cards ON credit_cards.user_id = users.id").Where("credit_cards.number = ?", "411111111111").
Find(&user)
```

#### 高级查询

```go
// 子查询
Where("amount > (?)", db.Table("orders").Select("AVG(amount)"))

// 内嵌子查询
subQuery := db.Select("AVG(age)").Where("name LIKE ?", "name%").Table("users")
db.Select("AVG(age) as avgage").Group("name").Having("AVG(age) > (?)", subQuery).Find(&results)

// From 子查询
db.Table("(?) as u", db.Model(&User{}).Select("name", "age")).Where("age = ?", 18).Find(&User{})

// 分组查询条件
db.Where(
	db.Where("pizza = ?", "pepperoni").Where(db.Where("size = ?", "small").Or("size = ?", "medium")),
).Or(
	db.Where("pizza = ?", "hawaiian").Where("size = ?", "xlarge"),
).Find(&Pizza{})

// 多列 IN
// SQL: SELECT * FROM users WHERE (name, age, role) IN (("jinzhu", 18, "admin"), ("jinzhu 2", 19, "user"))
db.Where("(name, age, role) IN ?", [][]interface{}{{"jinzhu", 18, "admin"}, {"jinzhu2", 19, "user"}}).Find(&users)

// 命名参数
db.Where("name1 = @name OR name2 = @name", sql.Named("name", "jinzhu")).Find(&user)
db.Where("name1 = @name OR name2 = @name", map[string]any{"name": "jinzhu"}).First(&user)

// 扫描结果至 Map
result := map[string]any{}
db.Model(&User{}).First(&result, "id = ?", 1)

var results []map[string]any
db.Table("users").Find(&results)

// Assign 结构上设置属性，不管是否找到记录
// user -> User{Name: "non_existing", Mobile: 18523774412} if not found
mobile := "18523774412"
var user User
db.Assign(User{Mobile: &mobile}).FirstOrInit(&user, User{Name: "non_existing"})

// FirstOrCreate 获取与特定条件匹配的第一条记录，或者如果没有找到匹配的记录，创建一个新的记录
// 还可以和 Assign 配合使用，这些属性会被新增或更新至数据库
result := db.FirstOrCreate(&user, User{Name: "non_existing"})

// FindInBatches 分批次的查询和处理记录

// Pluck 获取某列数据
// 检索所有用户的 age
var ages []int64
db.Model(&User{}).Pluck("age", &ages)

// 检索所有用户的 name
var names []string
db.Model(&User{}).Pluck("name", &names)

// Scopes
// 用于筛选 amount 大于 1000 的记录范围
func AmountGreaterThan1000(db *gorm.DB) *gorm.DB {
	return db.Where("amount > ?", 1000)
}

// 用于 订单使用信用卡支付 的 Scope
func PaidWithCreditCard(db *gorm.DB) *gorm.DB {
	return db.Where("pay_mode_sign = ?", "C")
}

// 使用 Scopes
db.Scopes(AmountGreaterThan1000, PaidWithCreditCard).Find(&orders)

// Count
var count int64

// 计数 有着特定名字的 users
// SELECT count(1) FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2'
db.Model(&User{}).Where("name = ?", "jinzhu").Or("name = ?", "jinzhu 2").Count(&count)

// 计数 有着单一名字条件（single name condition）的 users
// SELECT count(1) FROM users WHERE name = 'jinzhu'
db.Model(&User{}).Where("name = ?", "jinzhu").Count(&count)

// 在不同的表中对记录计数
// SELECT count(1) FROM deleted_users
db.Table("deleted_users").Count(&count)
```

### 更新

#### 保存所有字段

根据 id 确定行需要更新还是插入，`Generics API` 中的 `Save` 方法被特意移除，避免歧义和并发问题。

```go
db.Save(&User{Name: "jinzhu", Age: 100})
// INSERT INTO `users` (`name`,`age`,`birthday`,`update_at`) VALUES ("jinzhu",100,"0000-00-00 00:00:00","0000-00-00 00:00:00")

db.Save(&User{ID: 1, Name: "jinzhu", Age: 100})
// UPDATE `users` SET `name`="jinzhu",`age`=100,`birthday`="0000-00-00 00:00:00",`update_at`="0000-00-00 00:00:00" WHERE `id` = 1
```

#### 更新列

需要条件才能完成更新， 否则会触发 `ErrMissingWhereClause` 错误。

**单列**

```go
ctx := context.Background()

// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE active=true
err := gorm.G[User](db).Where("active = ?", true).Update(ctx, "name", "hello")

// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111
err := gorm.G[User](db).Where("id = ?", 111).Update(ctx, "name", "hello")

// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111 AND active=true;
err := gorm.G[User](db).Where("id = ? AND active = ?", 111, true).Update(ctx, "name", "hello")
```

**多列**
```go
ctx := context.Background()

// 使用 struct 更新字段，只会更新非 0 值
// UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE id = 111
rows, err := gorm.G[User](db).Where("id = ?", 111).Updates(ctx, User{Name: "hello", Age: 18, Active: false})

// rows 为影响的行数
// UPDATE users SET name='hello', age=18, active=false, updated_at='2013-11-17 21:34:10' WHERE id=111;
rows, err := gorm.G[map[string]any](db).Table("users").Where("id = ?", 111).Updates(ctx, map[string]any{"name": "hello", "age": 18, "active": false})
```

**更新选定的字段**
```go
// 更新指定的字段
Select("name", "age")

// 忽略指定的字段
Omit("name", "age")

// 更新所有字段，包括值为 0 的字段
rows, err := gorm.G[User](db).Where("id = ?", 111).Select("*").Updates(ctx, User{Name: "jinzhu", Role: "admin", Age: 0})
```

**防止 ErrMissingWhereClause 错误**

```go
// 添加一个恒为真的条件
Where("1 = 1")

// 执行原生 SQL
Exec("UPDATE users SET name = ?", "jinzhu")

// 会话模式
db.Session(&gorm.Session{AllowGlobalUpdate: true}).Model(&User{}).Update("name", "jinzhu")
```

**使用 SQL 表达式更新**

```go
// UPDATE "products" SET "price" = price * 2 + 100, "updated_at" = '2013-11-17 21:34:10' WHERE "id" = 3
db.Model(&product).Update("price", gorm.Expr("price * ? + ?", 2, 100))

// UPDATE "products" SET "price" = price * 2 + 100, "updated_at" = '2013-11-17 21:34:10' WHERE "id" = 3
db.Model(&product).Updates(map[string]interface{}{"price": gorm.Expr("price * ? + ?", 2, 100)})

// UPDATE "products" SET "quantity" = quantity - 1 WHERE "id" = 3
db.Model(&product).UpdateColumn("quantity", gorm.Expr("quantity - ?", 1))

// UPDATE "products" SET "quantity" = quantity - 1 WHERE "id" = 3 AND quantity > 1
db.Model(&product).Where("quantity > 1").UpdateColumn("quantity", gorm.Expr("quantity - ?", 1))
```

**不使用 Hook 和时间追踪**

```go
// UPDATE users SET name='hello' WHERE id = 111
db.Model(&user).UpdateColumn("name", "hello")

// UPDATE users SET name='hello', age=18 WHERE id = 111
db.Model(&user).UpdateColumns(User{Name: "hello", Age: 18})

// UPDATE users SET name='hello', age=0 WHERE id = 111
db.Model(&user).Select("name", "age").UpdateColumns(User{Name: "hello", Age: 0})
```

**返回修改行的数据**

需要数据库支持，经测试 pgsql 是支持的。

```go
// return all columns
var users []User
db.Model(&users).Clauses(clause.Returning{}).Where("role = ?", "admin").Update("salary", gorm.Expr("salary * ?", 2))
// UPDATE `users` SET `salary`=salary * 2,`updated_at`="2021-10-28 17:37:23.19" WHERE role = "admin" RETURNING *
// users => []User{{ID: 1, Name: "jinzhu", Role: "admin", Salary: 100}, {ID: 2, Name: "jinzhu.2", Role: "admin", Salary: 1000}}

// return specified columns
db.Model(&users).Clauses(clause.Returning{Columns: []clause.Column{{Name: "name"}, {Name: "salary"}}}).Where("role = ?", "admin").Update("salary", gorm.Expr("salary * ?", 2))
// UPDATE `users` SET `salary`=salary * 2,`updated_at`="2021-10-28 17:37:23.19" WHERE role = "admin" RETURNING `name`, `salary`
// users => []User{{ID: 0, Name: "jinzhu", Role: "", Salary: 100}, {ID: 0, Name: "jinzhu.2", Role: "", Salary: 1000}}
```

### 删除

当你试图执行不带任何条件的批量删除时，GORM将不会运行并返回 `ErrMissingWhereClause` 错误。

```go
// 仅主键
// DELETE from users where id = 10
rows, err := gorm.G[User](db).Where("id = ?", 40).Delete(ctx)

// 主键和其他条件
// DELETE from users where id = 10 AND name = "hello"
rows, err := gorm.G[User](db).Where("id = ? AND name = ?", 39, "hello").Delete(ctx)

// 模糊匹配
// rows, err := gorm.G[User](db).Where("email LIKE ?", "%jinzhu%").Delete(ctx)

// 使用恒为真条件避免 ErrMissingWhereClause 错误
// err := gorm.G[User](db).Where("1 = 1").Delete(ctx)
```

#### 软删除

当模型包含 `DeletedAt` 字段时，调用 `Delete` 将字段使用软删除，即不删除数据，仅将 `DeletedAt` 字段值设置为当前时间（而后的一般查询方法将无法查找到此条记录）。

```go
// 使用 Unscoped 查找被软删除的记录
db.Unscoped().Where("age = 20").Find(&users)

// 使用 Unscoped 永久删除匹配的记录
db.Unscoped().Delete(&order)
```

### 执行原生 SQL / 原生 SQL 生成器

**执行原生 SQL**

```go
result, err := gorm.G[User](db).Raw("SELECT id, name FROM users WHERE id = ?", 3).Find(ctx)

age, err := gorm.G[int](db).Raw("SELECT SUM(age) FROM users WHERE role = ?", "admin").Find(ctx)

users, err := gorm.G[User](db).Raw("UPDATE users SET name = ? WHERE age = ? RETURNING id, name", "jinzhu", 20).Find(ctx)

result := gorm.WithResult()
err := gorm.G[any](db, result).Exec(ctx, "DROP TABLE users")

// 批量更新
err = gorm.G[any](db).Exec(ctx, "UPDATE orders SET shipped_at = ? WHERE id IN ?", time.Now(), []int64{1, 2, 3})
err = gorm.G[any](db).Exec(ctx, "UPDATE users SET money = ? WHERE name = ?", gorm.Expr("money * ? + ?", 10000, 1), "jinzhu")

// 使用命名参数
users, err := gorm.G[User](db).Raw("SELECT * FROM users WHERE name1 = @name OR name2 = @name2 OR name3 = @name", sql.Named("name", "jinzhu1"), sql.Named("name2", "jinzhu2")).Find(ctx)

// 命名参数进阶
type NamedArgument struct {
    Name string
    Name2 string
}
users, err := gorm.G[User](db).Raw("SELECT * FROM users WHERE (name1 = @Name AND name3 = @Name) AND name2 = @Name2", NamedArgument{Name: "jinzhu", Name2: "jinzhu2"}).Find(ctx)
```

**原生 SQL 生成器**

```go
stmt := db.Session(&gorm.Session{DryRun: true}).First(&user, 1).Statement
stmt.SQL.String() //=> SELECT * FROM `users` WHERE `id` = $1 ORDER BY `id`
stmt.Vars         //=> []interface{}{1}

sql := db.ToSQL(func(tx *gorm.DB) *gorm.DB {
	return tx.Model(&User{}).Where("id = ?", 100).Limit(10).Order("age desc").Find(&[]User{})
})
sql //=> SELECT * FROM "users" WHERE id = 100 AND "users"."deleted_at" IS NULL ORDER BY age desc LIMIT 10
```

**Row & Rows**

```go
// Generics API
name, err := gorm.G[string](db).
	Raw("SELECT name FROM users WHERE id = ?", 1).
	Find(ctx)

count, err := gorm.G[int64](db).
    Raw("SELECT COUNT(*) FROM users WHERE age > ?", 18).
    Find(ctx)

var name string
err := gorm.G[User](db).
    Where("id = ?", 1).
    Select("name").
    Scan(ctx, &name)

// 传统 API
var name string
err := db.Raw("SELECT name FROM users WHERE id = ?", 1).Scan(&name)
// 没数据 → err == sql.ErrNoRows

var count int64
err := db.Raw("SELECT COUNT(*) FROM users WHERE age > ?", 18).Scan(&count)

var age int
err := db.Model(&User{}).Where("name = ?", "hello").Select("age").Scan(&age)


// 使用 GORM API 构建 SQL
rows, err := db.Model(&User{}).Where("name = ?", "jinzhu").Select("name, age, email").Rows()
defer rows.Close()
for rows.Next() {
  rows.Scan(&name, &age, &email)

  // 业务逻辑...
}

// 原生 SQL
rows, err := db.Raw("select name, age, email from users where name = ?", "jinzhu").Rows()
defer rows.Close()
for rows.Next() {
  rows.Scan(&name, &age, &email)

  // 业务逻辑...
}

// 将 sql.Rows 扫描至 model
rows, err := db.Model(&User{}).Where("name = ?", "jinzhu").Select("name, age, email").Rows() // (*sql.Rows, error)
defer rows.Close()

var user User
for rows.Next() {
  // ScanRows 将一行扫描至 user
  db.ScanRows(rows, &user)

  // 业务逻辑...
}
```

## 关联

学生 belongs to 班级，一个学生只能在一个班级，而班级里能装几十个学生。

人 has one 身份证，一个身份证只能属于一个人，一张身份证也只属于一个人。

班级 has many 学生，一个班级可以有多个学生，反过来每个学生都 belongs to 这个班级，班级表不用存学生 id，学生那边存班级 id 就够关联了。

学生 many to many 课程，多对多，需要中间表记录绑定关系。

### 多对一 belongs to

```go
// User 属于 Company，CompanyID 是外键
type User struct {
	gorm.Model
	Name      string
	CompanyID int
	Company   Company
}

type Company struct {
	ID   int
	Name string
}
```

### 一对一 has one

```go
// User 有一张 CreditCard，UserID 是外键
type User struct {
	gorm.Model
	CreditCard CreditCard
}

type CreditCard struct {
	gorm.Model
	Number string
	UserID uint
}
```

### 一对多 Has Many

```go
// User 有多张 CreditCard，UserID 是外键
// 主表 User 表，不存储 CreditCard 数据，CreditCards 字段，是 GORM 查询时，用来接收关联查询得到的卡片数据的
type User struct {
	gorm.Model
	CreditCards []CreditCard
}

type CreditCard struct {
	gorm.Model
	Number string
	UserID uint
}
```

### 多对多 many to many

```go
// User 拥有并属于多种 language，`user_languages` 是连接表
type User struct {
	gorm.Model
	Languages []Language `gorm:"many2many:user_languages;"`
}

type Language struct {
	gorm.Model
	Name string
}
```

当使用 GORM 的 AutoMigrate 为 User 创建表时，GORM 会自动创建连接表。

### 在创建时自动保存/删除关联

```go

// 两个表将分别插入一行数据，其中 Card.AdminID 将被设置为 Admin.ID
// 通过 Select 和 Omit 方法，可以指定更新数据时，那些字段被包括或排除，包括关联表字段

type Card struct {
	gorm.Model
	CardNumber string
	AdminID    uint
}

type Admin struct {
	gorm.Model
	Card Card
}

db := database.GetDB()
db.AutoMigrate(&Admin{}, &Card{})

admin := Admin{
	Card: Card{
		CardNumber: "123456",
	},
}

// 创建管理员时跳过全部关联关系 Omit(clause.Associations)
// err := gorm.G[Admin](db).Omit(clause.Associations).Create(ctx, &admin)

err := gorm.G[Admin](db).Create(context.Background(), &admin)
if err != nil {
	log.Fatalf("create user: %v", err)
}
```

```go
// 级联删除
admin := Admin{
	Model: gorm.Model{
		ID: 8,
	},
}
db.Select("Card").Delete(&admin)

// 单行简写
db.Select("Card").Delete(&Admin{Model: gorm.Model{ID: 7}})
```

### 关联模式

用 Association("字段名") 来专门「增删改查」某个模型的关联数据（HasOne/HasMany/BelongsTo/Many2Many）。

```go
var admin Admin
db.First(&admin, 9)

var card Card
db.Model(&admin).Association("Card").Find(&card) // 查 admin 对应的那张 card

// 追加关联
newCard := Card{CardNumber: "123456"}
// 关联上（会自动设置 AdminID=9）
// 对 HasOne 来说，Append 等同于「替换」
db.Model(&admin).Association("Card").Append(Card{CardNumber: "123456"})

// 替换关联，Replace 有第二个参数，表示被替换的关联数据
db.Model(&admin).Association("Card").Replace(Card{CardNumber: "999999"})

// 删除关联（无物理删除）
db.Model(&admin).Association("Card").Delete(card)
// Delete(languageZH, languageEN)
// Delete([]Language{languageZH, languageEN})

// 通过Unscoped来变更默认的删除行为
// 软删除
db.Model(&user).Association("Languages").Unscoped().Clear()
// 物理删除 db.Unscoped().Model(&user)
db.Unscoped().Model(&user).Association("Languages").Unscoped().Clear()

// 清空关联
db.Model(&user).Association("Languages").Clear()

// 关联计数
db.Model(&user).Association("Languages").Count()
```

### 预加载

#### Preload 在单独的查询中加载关联数据

```go
type Card struct {
	gorm.Model
	CardNumber string
	AdminID    uint
}

type Admin struct {
	gorm.Model
	Name string
	Card Card
}

// 预加载 Card 表的关联数据
// SELECT * FROM admins
// SELECT * FROM cards WHERE user_id IN (1,2,3,4)
admin, err := gorm.G[Admin](db).Preload("Card", nil).Where("id = ?", 10).Find(ctx)

// SELECT * FROM admins
// SELECT * FROM cards WHERE user_id IN (1,2,3,4) order by cards.card_number DESC
admin, err := gorm.G[Admin](db).Preload("Card", func(db gorm.PreloadBuilder) error {
	db.Order("cards.card_number DESC")
	return nil
}).Find(ctx)
```

#### Joins Preload 单条 SQL 中，以左连接的方式加载关联数据

```go
admin, err := gorm.G[Admin](db).Joins(clause.JoinTarget{Association: "Card"}, nil).Find(ctx)

admin, err := gorm.G[Admin](db).Joins(clause.JoinTarget{Association: "Card"},
	func(db gorm.JoinBuilder, joinTable clause.Table, curTable clause.Table) error {
		db.Where(Card{AdminID: 11})
		return nil
	}).Find(ctx)

// Traditional API
db.Joins("Company").Joins("Manager").Joins("Account").First(&user, 1)
db.Joins("Company").Joins("Manager").Joins("Account").First(&user, "users.name = ?", "jinzhu")
db.Joins("Company").Joins("Manager").Joins("Account").Find(&users, "users.id IN ?", []int{1,2,3,4,5})

db.Joins("Company", DB.Where(&Company{Alive: true})).Find(&users)

// 嵌套模型
db.Joins("Manager").Joins("Manager.Company").Find(&users)
```

#### 预加载全部

```go
// 预加载全部关联模型
type Card struct {
	gorm.Model
	CardNumber string
	AdminID    uint
}

type Admin struct {
	gorm.Model
	Name string
	Card Card
}

var admin Admin
admin.ID = 10
db.Preload(clause.Associations).Find(&admin)

// 预加载全部不含嵌套模型，可以使用以下语法解决
db.Preload("Orders.OrderItems.Product").Preload(clause.Associations).Find(&admin)
```

#### 条件预加载

```go
// SELECT * FROM users
// SELECT * FROM orders WHERE user_id IN (1,2,3,4) AND state NOT IN ('cancelled')
db.Preload("Orders", "state NOT IN (?)", "cancelled").Find(&users)

// SELECT * FROM users WHERE state = 'active'
// SELECT * FROM orders WHERE user_id IN (1,2) AND state NOT IN ('cancelled')
db.Where("state = ?", "active").Preload("Orders", "state NOT IN (?)", "cancelled").Find(&users)
```

#### 自定义预加载 SQL

```go
// SELECT * FROM users
// SELECT * FROM orders WHERE user_id IN (1,2,3,4) order by orders.amount DESC
db.Preload("Orders", func(db *gorm.DB) *gorm.DB {
  return db.Order("orders.amount DESC")
}).Find(&users)
```

## Context

```go
ctx := context.Background()

// Generics API
users, err := gorm.G[User](db).Find(ctx)

// Traditional API
db.WithContext(ctx).Find(&users)

// 持续会话模式
tx := db.WithContext(ctx)
tx.First(&user, 1)
tx.Model(&user).Update("role", "admin")

// context 超时
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
users, err := gorm.G[User](db).Find(ctx)

// Hooks/Callbacks 中的 Context
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
	ctx := tx.Statement.Context
	return
}
```

## 错误处理

```go
// Generics API（错误会直接从操作的方法中返回）
user, err := gorm.G[User](db).Where("name = ?", "jinzhu").First(ctx)
if err != nil {
	// Handle error...
}

err := gorm.G[User](db).Where("id = ?", 1).Delete(ctx)
if err != nil {
	// Handle error...
}

// Traditional API（链式语法集成）
if err := db.Where("name = ?", "jinzhu").First(&user).Error; err != nil {
	// 处理错误...
}

if result := db.Where("name = ?", "jinzhu").First(&user); result.Error != nil {
	// 处理错误...
}


// ErrRecordNotFound
// 当使用 First、Last、Take 等方法未找到记录时，GORM 会返回 ErrRecordNotFound
user, err := gorm.G[User](db).First(ctx)
if errors.Is(err, gorm.ErrRecordNotFound) {
	// Handle record not found error...
}

// 方言转换错误 TranslateError，GORM 会将数据库特有的错误转换为其自己的通用错误
// 通用错误参考：https://github.com/go-gorm/gorm/blob/master/errors.go
db, err := gorm.Open(postgres.Open(postgresDSN), &gorm.Config{TranslateError: true})
```

## 链式方法

GORM 中的方法被分为三大类：Chain（链式）、Finisher（终结）、New Session（新会话），在调用 Chain 和 Finisher 方法后，GORM 会返回一个已经初始化的 `*gorm.DB` 实例（不干净的实例），它可能将之前的状态带回，无法安全的重新使用：

在 Generics API 中一般不用担心这个问题，因为总是在操作方法第一个参数传递 ctx，Traditional API 中预防的方法则是使用 `Session(&gorm.Session{})` 实现安全复用，如：

```go
queryDB := DB.Where("name = ?", "jinzhu").Session(&gorm.Session{})

queryDB.Where("age > ?", 10).First(&user)

queryDB.Where("age > ?", 20).First(&user2)

// age 不会累计，不使用 Session(&gorm.Session{}) 时，则会累加，造成条件重复
```

## 钩子 Hooks

1. Hook 是在创建、查询、更新、删除等操作之前、之后调用的函数。
2. 如果任何回调返回错误，GORM 将停止后续的操作并回滚事务。
3. 钩子方法的函数签名应该是 `func(*gorm.DB) error`

#### 创建时可用的 hook

// 开始事务
`BeforeSave`
`BeforeCreate`
关联前的 save
插入记录至 db
关联后的 save
`AfterCreate`
`AfterSave`
提交或回滚事务


```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
	u.UUID = uuid.New()

	if !u.IsValid() {
		err = errors.New("can't save invalid data")
	}
	return

	tx.Model(u).Update("role", "admin")
}
```

#### 更新时可用的 hook

开始事务
`BeforeSave`
`BeforeUpdate`
关联前的 save
更新 db
关联后的 save
`AfterUpdate`
`AfterSave`
提交或回滚事务

```go
// 在同一个事务中更新数据
func (u *User) AfterUpdate(tx *gorm.DB) (err error) {
	if u.Confirmed {
		tx.Model(&Address{}).Where("user_id = ?", u.ID).Update("verfied", true)
	}
	return
}
```

#### 删除时可用的 hook

开始事务
`BeforeDelete`
删除 db 中的数据
`AfterDelete`
提交或回滚事务

#### 查询时可用的 hook

从 db 中加载数据
Preloading (eager loading)
`AfterFind`

```go
func (u *User) AfterFind(tx *gorm.DB) (err error) {
	if u.MemberShip == "" {
		u.MemberShip = "user"
	}
	return
}
```

#### 修改当前操作

```go
func (u *User) BeforeCreate(tx *gorm.DB) error {
	// 通过 tx.Statement 修改当前操作，例如：
	tx.Statement.Select("Name", "Age")
	tx.Statement.AddClause(clause.OnConflict{DoNothing: true})

	// tx 是带有 `NewDB` 选项的新会话模式 
	// 基于 tx 的操作会在同一个事务中，但不会带上任何当前的条件
	err := tx.First(&role, "name = ?", user.Role).Error
	// SELECT * FROM roles WHERE name = "admin"
	// ...
	return err
}
```

## 事务

为了确保数据一致性，GORM 会在事务里执行写入操作（创建、更新、删除）。

```go
// 全局禁用
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
	SkipDefaultTransaction: true,
})

// 持续会话模式
tx := db.Session(&Session{SkipDefaultTransaction: true})
tx.First(&user, 1)
tx.Find(&users)
tx.Model(&user).Update("Age", 18)
```

#### Generics API

```go
ctx := context.Background()
err := db.Transaction(func(tx *gorm.DB) error {
	if err := gorm.G[Animal](tx).Create(ctx, &Animal{Name: "Giraffe"}); err != nil {
		// 返回任何错误都会回滚事务
		return err
	}

	if err := gorm.G[Animal](tx).Create(ctx, &Animal{Name: "Lion"}); err != nil {
		return err
	}

	// 返回 nil 提交事务
	return nil
})
```

#### Traditional API

```go
db.Transaction(func(tx *gorm.DB) error {
	// 在事务中执行一些 db 操作（从这里开始，应该使用 'tx' 而不是 'db'）
	if err := tx.Create(&Animal{Name: "Giraffe"}).Error; err != nil {
		// 返回任何错误都会回滚事务
		return err
	}

	if err := tx.Create(&Animal{Name: "Lion"}).Error; err != nil {
		return err
	}

	// 返回 nil 提交事务
	return nil
})
```

#### 嵌套事务

```go
db.Transaction(func(tx *gorm.DB) error {
	tx.Create(&user1)

	tx.Transaction(func(tx2 *gorm.DB) error {
		tx2.Create(&user2)
		return errors.New("rollback user2") // Rollback user2
	})

	tx.Transaction(func(tx3 *gorm.DB) error {
		tx3.Create(&user3)
		return nil
	})

	return nil
})

// Commit user1, user3
```

#### 手动事务

```go
// 开始事务
tx := db.Begin()

// 在事务中执行一些 db 操作（从这里开始，应该使用 'tx' 而不是 'db'）
tx.Create(...)

// 遇到错误时回滚事务
tx.Rollback()

// 否则，提交事务
tx.Commit()
```

#### SavePoint、RollbackTo

GORM 提供了保存点以及回滚至保存点功能。

```go
tx := db.Begin()
tx.Create(&user1)

tx.SavePoint("sp1")
tx.Create(&user2)
tx.RollbackTo("sp1") // Rollback user2

tx.Commit() // Commit user1
```

## 日志

```go
newLogger := logger.New(
	log.New(os.Stdout, "\r\n", log.LstdFlags), // io writer
	logger.Config{
		SlowThreshold:              time.Second,   // 慢日志阈值
		LogLevel:                   logger.Silent, // Log 等级：Silent、Error、Warn、Info
		IgnoreRecordNotFoundError: true,           // 忽略记录未找到的错误
		ParameterizedQueries:      true,           // 不要将参数记入日志
		Colorful:                  false,          // 禁用颜色
	},
)

// Globally mode
db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{
	Logger: newLogger,
})

// Continuous session mode
tx := db.Session(&Session{Logger: newLogger})
tx.First(&user)
tx.Model(&user).Update("Age", 18)

// Debug 单个查询
db.Debug().Where("name = ?", "jinzhu").First(&User{})
```

## 设置

GORM 提供了 Set, Get, InstanceSet, InstanceGet 方法来允许用户传值给 勾子 或其他方法。

```go
type User struct {
	gorm.Model
	CreditCard CreditCard
	// ...
}

func (u *User) BeforeCreate(tx *gorm.DB) error {
	myValue, ok := tx.Get("my_value")
	// ok => true
	// myValue => 123
}

type CreditCard struct {
	gorm.Model
	// ...
}

func (card *CreditCard) BeforeCreate(tx *gorm.DB) error {
  myValue, ok := tx.Get("my_value")
	// ok => true
	// myValue => 123
}

myValue := 123
db.Set("my_value", myValue).Create(&User{})
```

```go
type User struct {
	gorm.Model
	CreditCard CreditCard
	// ...
}

func (u *User) BeforeCreate(tx *gorm.DB) error {
	myValue, ok := tx.InstanceGet("my_value")
	// ok => true
	// myValue => 123
}

type CreditCard struct {
	gorm.Model
	// ...
}

// 在创建关联时，GORM 创建了一个新 `*Statement`，所以它不能读取到其它实例的设置
func (card *CreditCard) BeforeCreate(tx *gorm.DB) error {
	myValue, ok := tx.InstanceGet("my_value")
	// ok => false
	// myValue => nil
}

myValue := 123
db.InstanceSet("my_value", myValue).Create(&User{})
```