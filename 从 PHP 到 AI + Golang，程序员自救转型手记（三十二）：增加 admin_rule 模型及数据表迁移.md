# 从 PHP 到 AI + Golang，程序员自救转型手记（三十二）：增加 admin_rule 模型及数据表迁移

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “增加 config 模型、数据表自动迁移优化”，本期将完成：增加 admin_rule 模型及数据表迁移

# 增加 admin_rule 模型及数据表迁移

> 这次算是跑通了后续增加迁移和模型的流程，如果需要示例数据也是一句话的事，AI 时代太方便了。

管理员规则表，存储了后台的菜单数据，权限节点数据等，上一期我们已经完成了数据表自动迁移的优化，现在称热打铁，再迁移一个 `admin_rule` 表。

1. 管理员规则表 SQL 如下，帮我实现对应模型及迁移，模型放在 `internal\model\admin.go` 中，迁移则是 `cmd\migrate\migrations\000001_admin.up.sql / 000001_admin.down.sql`：

```sql
CREATE TABLE `ba_admin_rule`  (
  `id` int UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `pid` int UNSIGNED NOT NULL DEFAULT 0 COMMENT '上级规则',
  `type` enum('dir','menu','node') CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT 'menu' COMMENT '规则类型:dir=规则目录,menu=菜单项,node=权限节点',
  `title` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '规则标题',
  `name` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '规则名称',
  `path` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '菜单路由路径',
  `icon` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '图标',
  `open_type` enum('tab','link','iframe') CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL DEFAULT NULL COMMENT '菜单打开方式:tab=选项卡,link=链接,iframe=Iframe',
  `url` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '菜单 Url',
  `component` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '菜单组件路径',
  `keepalive` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '缓存:0=关闭,1=开启',
  `extend` enum('none','add_route_only','add_menu_only') CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT 'none' COMMENT '扩展属性:none=无,add_route_only=只添加为路由,add_menu_only=只添加为菜单',
  `remark` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '备注',
  `weigh` int NOT NULL DEFAULT 0 COMMENT '权重',
  `status` tinyint UNSIGNED NOT NULL DEFAULT 1 COMMENT '状态:0=禁用,1=启用',
  `update_time` bigint UNSIGNED NULL DEFAULT NULL COMMENT '更新时间',
  `create_time` bigint UNSIGNED NULL DEFAULT NULL COMMENT '创建时间',
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `pid`(`pid` ASC) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci COMMENT = '菜单和权限规则表' ROW_FORMAT = DYNAMIC;
```

2. 参考 `src\layouts\admin\index.vue` 的 `临时 MOCK 数据` 随便生成几行权限数据，直接打印出来，我手动插入。

生成的迁移 SQL 如下：

```sql
CREATE TABLE IF NOT EXISTS "__PREFIX__admin_rule" (
    "id"          bigserial PRIMARY KEY,
    "pid"         bigint DEFAULT NULL,
    "type"        varchar(50) NOT NULL DEFAULT '',
    "title"       varchar(50) NOT NULL DEFAULT '',
    "name"        varchar(50) NOT NULL DEFAULT '',
    "path"        varchar(255) NOT NULL DEFAULT '',
    "icon"        varchar(50) NOT NULL DEFAULT '',
    "open_type"   varchar(50) NOT NULL DEFAULT '',
    "url"         varchar(255) NOT NULL DEFAULT '',
    "component"   varchar(255) NOT NULL DEFAULT '',
    "keepalive"   smallint NOT NULL DEFAULT 0,
    "extend"      varchar(50) NOT NULL DEFAULT '',
    "remark"      varchar(255) NOT NULL DEFAULT '',
    "weigh"       bigint NOT NULL DEFAULT 0,
    "status"      smallint NOT NULL DEFAULT 1,
    "update_time" timestamptz,
    "create_time" timestamptz
);

CREATE INDEX IF NOT EXISTS "__PREFIX__idx_admin_rule_pid" ON "__PREFIX__admin_rule" ("pid");

COMMENT ON TABLE "__PREFIX__admin_rule" IS '菜单和权限规则表';
COMMENT ON COLUMN "__PREFIX__admin_rule"."id" IS 'ID';
COMMENT ON COLUMN "__PREFIX__admin_rule"."pid" IS '上级规则';
COMMENT ON COLUMN "__PREFIX__admin_rule"."type" IS '规则类型:dir=规则目录,menu=菜单项,node=权限节点';
COMMENT ON COLUMN "__PREFIX__admin_rule"."title" IS '规则标题';
COMMENT ON COLUMN "__PREFIX__admin_rule"."name" IS '规则名称';
COMMENT ON COLUMN "__PREFIX__admin_rule"."path" IS '菜单路由路径';
COMMENT ON COLUMN "__PREFIX__admin_rule"."icon" IS '菜单图标';
COMMENT ON COLUMN "__PREFIX__admin_rule"."open_type" IS '菜单打开方式:tab=选项卡,link=链接,iframe=Iframe';
COMMENT ON COLUMN "__PREFIX__admin_rule"."url" IS '菜单URL';
COMMENT ON COLUMN "__PREFIX__admin_rule"."component" IS '菜单组件路径';
COMMENT ON COLUMN "__PREFIX__admin_rule"."keepalive" IS '缓存:0=关闭,1=开启';
COMMENT ON COLUMN "__PREFIX__admin_rule"."extend" IS '扩展属性:add_route_only=只添加为路由,add_menu_only=只添加为菜单';
COMMENT ON COLUMN "__PREFIX__admin_rule"."remark" IS '备注';
COMMENT ON COLUMN "__PREFIX__admin_rule"."weigh" IS '权重';
COMMENT ON COLUMN "__PREFIX__admin_rule"."status" IS '状态:0=禁用,1=启用';
COMMENT ON COLUMN "__PREFIX__admin_rule"."update_time" IS '更新时间';
COMMENT ON COLUMN "__PREFIX__admin_rule"."create_time" IS '创建时间';
```

我手动改了改，让 CC ：基于我修改后的 `@internal/middleware/admin_auth.go` 重新整理管理员权限模型

结果 @ 的文件错了，这 @ 的是中间件，我勒个豆，直接给我整出了 `admin_group` 和 `admin_group_access` 表的迁移数据，然后把 `admin` 控制器里边又查了 `admin_group` 登录用户的权限等等逻辑。

我只是想根据迁移 SQL 更新一下 `AdminRule` 模型而已😂，等了半天才发现它走了歪路，还浪费大量 tokens...

最后生成的 `AdminRule` 模型如下：

```go
// AdminRule 菜单和权限规则模型
type AdminRule struct {
	ID         uint      `gorm:"comment:ID;primarykey;autoIncrement" json:"id"`
	Pid        *uint     `gorm:"comment:上级规则" json:"pid"`
	Type       string    `gorm:"comment:规则类型:dir=规则目录,menu=菜单项,node=权限节点;type:varchar(50);not null;default:''" json:"type"`
	Title      string    `gorm:"comment:规则标题;type:varchar(50);not null;default:''" json:"title"`
	Name       string    `gorm:"comment:规则名称;type:varchar(50);not null;default:''" json:"name"`
	Path       string    `gorm:"comment:菜单路由路径;type:varchar(255);not null;default:''" json:"path"`
	Icon       string    `gorm:"comment:菜单图标;type:varchar(50);not null;default:''" json:"icon"`
	OpenType   string    `gorm:"comment:菜单打开方式:tab=选项卡,link=链接,iframe=Iframe;type:varchar(50);not null;default:''" json:"open_type"`
	URL        string    `gorm:"comment:菜单URL;type:varchar(255);not null;default:''" json:"url"`
	Component  string    `gorm:"comment:菜单组件路径;type:varchar(255);not null;default:''" json:"component"`
	Keepalive  uint8     `gorm:"comment:缓存:0=关闭,1=开启;not null;default:0" json:"keepalive"`
	Extend     string    `gorm:"comment:扩展属性:add_route_only=只添加为路由,add_menu_only=只添加为菜单;type:varchar(50);not null;default:''" json:"extend"`
	Remark     string    `gorm:"comment:备注;type:varchar(255);not null;default:''" json:"remark"`
	Weigh      int       `gorm:"comment:权重;not null;default:0" json:"weigh"`
	Status     uint8     `gorm:"comment:状态:0=禁用,1=启用;not null;default:1" json:"status"`
	UpdateTime time.Time `gorm:"comment:更新时间" json:"update_time"`
	CreateTime time.Time `gorm:"comment:创建时间" json:"create_time"`
}

func (AdminRule) TableName() string {
	return "admin_rules"
}
```