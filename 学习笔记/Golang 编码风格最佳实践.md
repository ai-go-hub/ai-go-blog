# Golang 编码风格最佳实践

本文是系列文章《从 PHP 到 AI + Golang，程序员自救转型手记》作者整理而来，完整版开源于：[github](https://github.com/ai-go-hub/ai-go-blog) | [gitee](https://gitee.com/ai-go-hub/ai-go-blog)

> 这份规范由多个 AI 生成初版对照，并人工整理和对比高星 top20 项目，结合 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目落地，完全贴合 Go 官方风格 + 企业级落地标准，包含文件夹、文件、变量、PostgreSQL 字段等全场景命名与编码规范。

# 一、核心命名总原则

- 严格区分大小写
- 简洁、语义化、见名知意，不使用拼音、无意义缩写
- 包名 / 文件夹名：全小写，无下划线、无驼峰
- 变量 / 函数：小驼峰（私有）、大驼峰（导出 / 公有）
- 数据库：蛇形命名（下划线）

# 二、文件夹 / 包 命名规范

- 全部小写
- 简短、单数形式优先
- 无下划线 \_、无横杠 -、无驼峰

# 三、文件命名规范

- 全部小写
- 多个单词用下划线 \_ 分隔
- 不使用空格、特殊符号、驼峰

# 四、变量 / 常量 / 函数 / 结构体命名

### 1. 变量

- 私有变量：小驼峰 camelCase
- 导出变量：大驼峰 CamelCase
- 布尔值用 is/has/can 前缀
- 禁止单字母（循环 i/j/k 除外）
- 禁止无意义命名、禁止中文拼音

### 2. 常量

全大写 + 下划线：`STATUS_OK`

### 3. 函数 / 方法

- 私有函数：小驼峰
- 导出函数：大驼峰
- 动词 + 名词 语义清晰 `getUserByID`

### 4. 结构体

- 大驼峰
- 字段：大驼峰，JSON 标签用蛇形

```go
type User struct {
    ID        int64     `json:"id"`
    UserName  string    `json:"user_name"`
    CreatedAt time.Time `json:"created_at"`
}
```

# 五、PostgreSQL 数据库字段命名

### 1. 命名规范

PG 大小写不敏感，统一使用蛇形命名（下划线）：

- 表名：小写 + 下划线
- 字段名：小写 + 下划线
- 禁止驼峰、禁止中文、禁止保留字

### 2. 必备公共字段

```sql
id                  bigint PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
created_at          timestamptz NOT NULL DEFAULT now(),
updated_at          timestamptz NOT NULL DEFAULT now(),
deleted_at          timestamptz NULL
```

# 六、快速对照表（速查表）

|                      |                              |                         |
| :------------------: | :--------------------------: | :---------------------: |
|       **场景**       |         **命名风格**         |        **示例**         |
|    文件夹 / 包名     |       全小写、无下划线       |  `api、service、model`  |
|        文件名        | 全小写、多个单词用下划线分隔 |    `user_service.go`    |
|  私有变量/函数/方法  |            小驼峰            |  `userID、getUserByID`  |
| 导出变量/函数/结构体 |            大驼峰            |  `UserID、GetUserByID`  |
|         常量         |        全大写+下划线         |       `STATUS_OK`       |
|   PostgreSQL 相关    |       小写，下划线分割       | `user_name、created_at` |
