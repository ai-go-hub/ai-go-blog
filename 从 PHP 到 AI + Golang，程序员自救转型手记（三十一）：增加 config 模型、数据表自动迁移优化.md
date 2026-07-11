# 从 PHP 到 AI + Golang，程序员自救转型手记（三十一）：增加 config 模型、数据表自动迁移优化

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “实现管理员注销接口、API 请求函数命名规范”，本期将完成：增加 config 模型、数据表自动迁移优化

# 增加 config 模型

得先把 `config` 模型加进来，因为站点的名称、备案号这些站点基础配置是存储在里边的，未来后台的【系统配置】功能也主要就是管理这个表中的数据。

作者之前的项目已经设计过完善的 `config` 数据表，这里直接将建表 SQL 给它复制过来，让 CC 转换为 GORM 模型：

新建 `config` 数据模型，基于以下 `SQL` 的表结构：

```sql
CREATE TABLE `config`  (
  `id` int UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `name` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '变量名',
  `group` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '分组',
  `title` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '变量标题',
  `tip` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '变量描述',
  `type` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '变量输入组件类型',
  `value` longtext CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL COMMENT '变量值',
  `content` longtext CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL COMMENT '字典数据',
  `rule` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '验证规则',
  `extend` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '扩展属性',
  `input_extend` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '' COMMENT '输入框扩展属性',
  `allow_del` tinyint UNSIGNED NOT NULL DEFAULT 0 COMMENT '允许删除:0=否,1=是',
  `weigh` int NOT NULL DEFAULT 0 COMMENT '权重',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `name`(`name` ASC) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci COMMENT = '系统配置' ROW_FORMAT = DYNAMIC;
```

AI 生成的模型如下：

```go
func init() {
	Register(&Config{})
}

// Config 系统配置模型
type Config struct {
	ID          uint   `gorm:"comment:ID;primarykey;autoIncrement" json:"id"`
	Name        string `gorm:"comment:变量名;type:varchar(50);uniqueIndex;not null;default:''" json:"name"`
	Group       string `gorm:"comment:分组;type:varchar(50);not null;default:''" json:"group"`
	Title       string `gorm:"comment:变量标题;type:varchar(50);not null;default:''" json:"title"`
	Tip         string `gorm:"comment:变量描述;type:varchar(100);not null;default:''" json:"tip"`
	Type        string `gorm:"comment:变量输入组件类型;type:varchar(50);not null;default:''" json:"type"`
	Value       string `gorm:"comment:变量值;type:longtext" json:"value"`
	Content     string `gorm:"comment:字典数据;type:longtext" json:"content"`
	Rule        string `gorm:"comment:验证规则;type:varchar(100);not null;default:''" json:"rule"`
	Extend      string `gorm:"comment:扩展属性;type:varchar(255);not null;default:''" json:"extend"`
	InputExtend string `gorm:"comment:输入框扩展属性;type:varchar(255);not null;default:''" json:"input_extend"`
	AllowDel    uint8  `gorm:"comment:允许删除:0=否,1=是;not null;default:0" json:"allow_del"`
	Weigh       int    `gorm:"comment:权重;not null;default:0" json:"weigh"`
}
```

这时候发现了一个问题，`config` 表内是有初始数据的，而且 GORM 的自动迁移终究不是长久之计，只适用于开发环境（不支持删除字段、不支持字段改名、字段类型变更风险、无版本记录等）。

所以，此时我就得将 GORM 的自动迁移改为更加适合生产环境的，然后再向 `config` 表插入数据。

# 数据表自动迁移优化

我们需要一个更加适合生产环境的数据表迁移工具，最起码得有版本控制，看了很多工具：

| 工具                                        | star数 | 备注                                                                     |
| ------------------------------------------- | ------ | ------------------------------------------------------------------------ |
| https://github.com/golang-migrate/migrate   | 18.7k  | 纯 SQL 驱动的迁移工具，不支持通过模型生成 SQL，生态第一                      |
| https://github.com/ariga/atlas              | 8.5k   | 新兴的迁移工具，支持通过模型生成 SQL，支持的语言和框架很多，感觉杂而不精        |
| https://github.com/go-gormigrate/gormigrate | 1.2k   | star 太少了                                                              |

其他看过的就不一一列出了，感觉 `golang-migrate/migrate` 挺好的，一个是官方背书，二是用户多，原生 SQL 驱动也方便 DBA 审核 SQL，至于不支持通过模型生成 SQL，AI 时代不是问题。接下来做规划如下：

1. 使用 `https://github.com/golang-migrate/migrate` 实现生产级数据库迁移功能
2. 项目需要同时支持 `golang-migrate/migrate` 的 `CLI` 模式和 `库` 模式，库模式于 `cmd/migrate/main.go` 建立自定义命令来执行：迁移、回滚、查看状态、创建新的迁移文件
3. 于 `cmd/migrate/migrations` 建立迁移文件，请参考现有模型写好 `SQL`

同时支持 CLI 模式和库模式的意思是：

```bash
# 即可以这样执行迁移
migrate -database "postgres://user:pass@host:port/dbname?sslmode=disable" -path cmd/migrate/migrations up

# 也可以执行项目自定义命令执行迁移
tmp/aigo.exe migrate up
```

自定义命令也就是库模式下，不需要填写 `-database` 和 `-path` 参数，也不需要单独安装 `migrate CLI`，是我最喜欢的模式。

CLI 模式其实是天然支持的，只要你安装了 `migrate CLI` 即可使用，安装后能额外执行：`force`、`drop` 等命令。

**将以上内容发给 CC：**

后续 CC 询问以何种方式实现自定义命令，这里选择了 `spf13/cobra`，然后：

```bash
库模式下迁移文件以何种方式加载？
  
❯ 1. go:embed 内嵌（推荐）
     up/down/status 用 go:embed 内嵌的迁移文件（编译后二进制自包含，便于部署）；create 命令按 --dir 写入磁盘目录。开发时创建后需重新编译才能 up。
  2. 文件系统路径
     所有操作直接读写 cmd/migrate/migrations 目录，与 golang-migrate CLI 模式完全一致，创建后立即可用。但运行二进制时需能访问到该目录。
```

意思也很简单，第一个是内嵌到编译产物内，生产直接就能用，第二个是文件系统内存储，生产要用迁移，得先上传文件，所以这里当然是选择内嵌。

第一轮完成后，再将原有项目 config 数据表的数据导出来，整理了一下可能会用得上的配置项，让 CC 基于该 SQL 生成一个 seed 迁移，主要包含 `站点名称、自定义后台入口、备案号、邮件发送` 等系统基础配置：

```sql
INSERT INTO `config` VALUES (1, 'config_group', 'basic', '配置分组', '', 'array', '[{\"key\":\"basic\",\"value\":\"基础配置\"},{\"key\":\"mail\",\"value\":\"邮件配置\"},{\"key\":\"config_quick_entrance\",\"value\":\"快捷配置入口\"}]', NULL, 'required', '','', 0, -1);
INSERT INTO `config` VALUES (2, 'name', 'basic', '站点名称', '', 'string', 'AI GO ADMIN', NULL, 'required', '','', 0, 99);
INSERT INTO `config` VALUES (3, 'entrance', 'basic', '自定义后台入口', '', 'string', '/admin', NULL, 'required', '','', 0, 1);
INSERT INTO `config` VALUES (4, 'record_number', 'basic', '域名备案号', '', 'string', '渝ICP备8888888号-1', NULL, '', '','', 0, 0);
INSERT INTO `config` VALUES (5, 'version', 'basic', '系统版本号', '', 'string', 'v1.0.0', NULL, 'required', '','', 0, 0);
INSERT INTO `config` VALUES (6, 'timezone', 'basic', '时区', '', 'string', 'Asia/Shanghai', NULL, 'required', '','', 0, 0);
INSERT INTO `config` VALUES (7, 'no_access_ip', 'basic', '禁止访问 IP', '禁止访问站点的ip列表，一行一个', 'textarea', NULL, NULL, '', '','', 0, 0);

INSERT INTO `config` VALUES (8, 'smtp_server', 'mail', 'SMTP 服务器', '', 'string', 'smtp.com', NULL, '', '','', 0, 9);
INSERT INTO `config` VALUES (9, 'smtp_port', 'mail', 'SMTP 端口', '', 'string', '465', NULL, '', '','', 0, 8);
INSERT INTO `config` VALUES (10, 'smtp_user', 'mail', 'SMTP 用户', '', 'string', NULL, NULL, '', '','', 0, 7);
INSERT INTO `config` VALUES (11, 'smtp_pass', 'mail', 'SMTP 密码', '', 'string', NULL, NULL, '', '','', 0, 6);
INSERT INTO `config` VALUES (12, 'smtp_verification', 'mail', 'SMTP 验证方式', '', 'select', 'SSL', '{\"SSL\":\"SSL\",\"TLS\":\"TLS\"}', '', '','', 0, 5);
INSERT INTO `config` VALUES (13, 'smtp_sender_mail', 'mail', 'SMTP 发件人邮箱', '', 'string', NULL, NULL, 'email', '','', 0, 4);

INSERT INTO `config` VALUES (14, 'config_quick_entrance', 'quick_entrance', '快捷配置入口', '', 'array', '[]', NULL, '', '','', 0, 0);
```

完工后，人工 `review` 发现了以下问题：

### 缺少表前缀配置应用

`cmd\migrate\migrations` 中的 SQL，表名全是 `admins` 、`tokens` 这种不带表前缀的，但是我们的系统支持表前缀，只是默认为空。

这里让 AI 在 SQL 中以固定字符串 `__PREFIX__` 表示表前缀，执行迁移前将该字符串替换为项目数据库配置中的实际表前缀。

### 项目命令规划

现在多了一个迁移命令，将原来的 `cmd/api/main.go` 改名为 `cmd/serve/serve.go`（作为单独的服务启动命令），然后新增 `cmd/main.go` 文件作为统一命令入口，从现在起：

```bash
# 启动 API 服务
go run cmd/main.go serve

# 应用数据库迁移
go run cmd/main.go migrate up

# 或编译后再运行
go build -o ./tmp/aigo.exe ./cmd/main.go
tmp/aigo.exe serve
tmp/aigo.exe migrate up
```

同时 `serve` 作为默认命令，是可以省略的。

### 同步修改 AGENTS.md / README.md / .air.toml

这次对数据迁移做了改动，项目启动流程和迁移功能介绍都需要重新写，直接让 CC 帮忙完成，再人工整理一下。

### 放弃原生的 Migrate CLI 工具

`https://github.com/golang-migrate/migrate` 是支持 `CLI` 和 `库` 双模式的，正常情况下我整合了库模式，`CLI` 天然就能用，此时突然发现原生的 `Migrate CLI` 不支持传递表前缀啊，这里只能再看看 `CLI` 还有那些命令是我们当前不支持的，让 CC 全部实现一遍（version 和 drop 子命令），然后直接放弃 `CLI` 模式。