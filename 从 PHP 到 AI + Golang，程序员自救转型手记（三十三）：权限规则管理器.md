# 从 PHP 到 AI + Golang，程序员自救转型手记（三十三）：权限规则管理器

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “增加 admin_rule 模型及数据表迁移”，本期将完成：权限规则管理器

# 权限规则管理器

### 必备表

先增加权限系统的必备表和模型：

`admin_group`: 管理员分组表，主要存储了 `组名` 及该组拥有那些 `权限`（`菜单和权限规则表` 的 ID 号）
`admin_group_access`: 管理员和以上分组的关系表，比如管理员 `yang` 加入分组 `超级管理组`，在此表记录（记录的是 id 号）

```sql
CREATE TABLE `admin_group`  (
  `id` int UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `pid` int UNSIGNED DEFAULT NULL COMMENT '上级分组',
  `name` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '组名',
  `rules` text CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL COMMENT '权限规则ID集',
  `status` tinyint UNSIGNED NOT NULL DEFAULT 1 COMMENT '状态:0=禁用,1=启用',
  `update_time` bigint UNSIGNED NULL DEFAULT NULL COMMENT '更新时间',
  `create_time` bigint UNSIGNED NULL DEFAULT NULL COMMENT '创建时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci COMMENT = '管理分组表' ROW_FORMAT = DYNAMIC;

CREATE TABLE `admin_group_access`  (
  `uid` int UNSIGNED NOT NULL COMMENT '管理员ID',
  `group_id` int UNSIGNED NOT NULL COMMENT '分组ID',
  INDEX `uid`(`uid` ASC) USING BTREE,
  INDEX `group_id`(`group_id` ASC) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci COMMENT = '管理分组映射表' ROW_FORMAT = DYNAMIC;
```

让 CC 顺便生成测试数据：

> 一个超级管理组，拥有全部权限 + 将 id 为 1 的管理员加入超级管理组

```sql
INSERT INTO `admin_group` VALUES (1, NULL, '超级管理组', '*', 1, 1776952825, 1776952825);
INSERT INTO `admin_group_access` VALUES (1, 1);
```

以上建表和建模型直接甩给 CC，不再赘述。

### 设计权限系统

以下内容是规划，也是给 CC 的提示词：

于 `internal\infra\permission\admin.go` 实现权限规则类：

1. 权限系统涉及的模型有 `model.AdminGroupAccess` 、`model.AdminGroup` 、`model.AdminRule`；每个管理员都有分组，分组关系存储于 `model.AdminGroupAccess`，分组的 `rules` 字段存储了管理员拥有的权限节点

2. 至少创建以下公开方法：
    - `GetRules` 提供用户 id，然后根据用户分组和分组关系表获取用户拥有的权限的规则列表
    - `Check` 提供用户 id 和 规则 name，检查用户是否拥有权限
    - `GetGroups` 提供用户 id 获取用户的分组列表
    - `GetRuleIds` 提供用户 id，根据用户的分组获取用户拥有权限的全部规则 id

先做以上简单规划，做出框架，边 `review` 边改边加，第一轮很快就写出来了：

1. 代码及注释中的 `用户` 字样改为 `管理员`
2. 代码中的参数名，从 `uid` 改为 `adminId`

这两个问题先让它改了来，全是提示词写了 `用户` 造成的，然后没有其他明显的问题了，我先实际测试一下代码运行再说吧...

测试期间报错了，问下 AI 啥问题：`我在 @cmd/serve/serve.go 测试获取管理员分组 id 方法，得到错误：panic: runtime error: ....`

问题已解决：

```bash
1. 删除了 @cmd/serve/serve.go 里的测试代码（p := permission.New() 那两行）
2. 移除了因此不再需要的 context 和 permission 两个 import
```

我直接好家伙，你给我测试代码删除，就没报错了是吧？

然后，人工逐行 `review` 加测试，还是找到以下问题（果然 AI 的代码就不可能没问题）：

#### 使用 * 代表全部权限（超管）

超管拥有全部权限，此时是没必要将 id 号逐一列出存储在分组的节点列表的，所以使用 `*` 代替，但是目前的权限类没有对 `*` 的检查，直接让 CC 加上。

#### 增加 IsSuperAdmin 方法

增加公开方法直接返回一个管理员是否是超管：

```go
// IsSuperAdmin 是否超级管理员
func (p *Permission) IsSuperAdmin(ctx context.Context, adminId uint) (bool, error) {
	groups, err := p.GetGroups(ctx, adminId)
	if err != nil {
		return false, err
	}

    // 检查管理员的每个分组的 rules 字段是否为 *
	return slices.ContainsFunc(groups, isSuperAdminGroup), nil
}
```

#### 获取规则 id 集失败

```go
var ids []uint
if err := json.Unmarshal([]byte(*group.Rules), &ids); err != nil {
    continue
}
```

`group.Rules` 里边的格式是 `规则ID1,规则ID2`，即简单的逗号分割，不是 `json`，这里按 `json` 肯定是解析不出来的，让 CC 处理一下（而且本来就不应该用 json，它的处理速度哪里有字符串快）。

#### 超管判断重复执行

`GetRules` 和 `Check` 方法都先检查了是否是超管 `slices.ContainsFunc(groups, isSuperAdminGroup)` ，已经不是超管才调 `ruleIdsFromGroups`，而 `ruleIdsFromGroups` 内部又判断了一次。需要在确保安全性的前提下，去掉冗余。

最终确定 `GetRules` 可以去除，`Check` 不去除则是有意义的，可以做到提前短路。

#### Check 的 ruleName 空字符串未校验

未对入参的规则名称进行空字符串检查，加上了：

```go
if ruleName == "" {
    return false, nil
}
```

#### allRuleIDs 方法查出全部字段，但只使用了 ID 字段

`allRuleIDs` 方法用于获取所有规则的 ID 号，读取了整行，但是只使用了 ID 号。

```go
rules, err := gorm.G[model.AdminRule](database.DB()).Where("status = 1").Find(ctx)

// 改为

rules, err := gorm.G[model.AdminRule](database.DB()).Select("id").Where("status = 1").Find(ctx)
```