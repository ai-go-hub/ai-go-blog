# 从 PHP 到 AI + Golang，程序员自救转型手记（三十四）：后台初始化请求实现

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “权限规则管理器”，本期将完成：后台初始化请求实现

# 后台初始化请求实现

之前我们已经建立好了 `config` 和 `admin_rule` 模型，加上以前就实现的 `admin` 模型，数据源基本齐备，我们可以开始初始化后台了。

流程是：管理员登录进入后台 > 跳转到 loading 页面 > 发送后台初始化请求并等待响应 > 注册动态路由 > 跳转到后台第一个路由

简单解释一下：后台有那些菜单可用，是在后台进行可视化配置的（菜单管理，目前还没做），而菜单管理的数据存储于 `admin_rule` 表（表已经有了，且插入了测试数据），管理员登录后，首先请求接口获取 `admin_rule` 表内的菜单数据，然后将菜单动态的注册为 `vue-router` 的路由，注册完成后就跳转到第一个路由去（一般是控制台页面）。

在等待初始化接口响应时，先跳转到 `admin/loading` 路由，显示一个加载动画，这是一个静态路由，直接就能访问。

初始化接口规划：

1. 接口地址 `/admin/init`，请求方式 `GET`，使用 `AdminAuth` 中间件
2. 响应当前管理员信息（`AdminAuth` 中间件已经解析 `token` 并在请求上下文中保存了当前管理员信息）
3. 响应站点配置，也就是 `config` 表中的数据，只返回 `name=name|record_number|version` 的行
4. 响应当前管理员拥有权限的菜单规则数据 `admin_rule`，可以通过上一章建立的权限规则管理器快速获取

写完规划我就想起：目前，时区配置放在了 config 表，以便未来后台内可以可视化修改，但是严格来讲，我们最好是项目启动时就设定好时区，而且 `pgsql` 的 `timestamptz` 字段类型也依赖于时区设定，配置位置：`config/config.yaml.database.write.timezone`

也就是说，存储于 `config` 表是不合适的，改为统一在代码 `文件配置` 中设定，这里让 CC 添加配置项，并在项目启动时，就完成时区初始化。

再研究了一下，直接让 AI 于 `config/config.yaml.app.timezone` 配置 `app` 级别的时区配置项，首先是确定了一下 `Asia/Shanghai` 这种写法是兼容 GORM + pgsql 的时区格式的，其次数据库的写库和读副本也不存在需要设定不同时区的情况（就算能也不推荐），所以目前只保留 `config/config.yaml.app.timezone` 一个时区配置项就可以了，让 CC 直接实现不再赘述。

> 初始化接口也读取 `config/config.yaml.app.timezone` 配置项，合并到站点配置中。

### 关键缺失

CC 实现初版之后，我注意到还缺少一个 `能组装出树状数据`（菜单数据）的包，以及缺少 dto 目录。

#### 建立 dto 目录

认证中间件中，`AdminSession` 管理员会话信息的结构体如下：

```go
// AdminSession 管理员会话信息，由认证中间件注入到请求上下文
type AdminSession struct {
	*model.Admin
	Token string
}
```

在制作 `/admin/init` 接口的过程中，发现管理员会话信息要传递到 `service` 层，传递的参数类型如果使用 `middleware.AdminSession`，HTTP 中间件的东西就入侵到 `service` 层了，这与我们的规划不符（未来假设要脱离 `HTTP` 层测试 `service`，是没有 `middleware.AdminSession` 的）

所以，现在起建立 dto 目录，用于专门存储数据传输对象，比如响应结构体，请求结构体，还有这种跨层传递的数据结构。

#### 建立 tree（树状数据管理）包

建立 `pkg/tree`（树状数据管理）包，用于处理树状数据。

提供以下对外方法：

一、根据 pid 和 id 组装 `children` 的方法：将数据源如：

```bash
[
    ['id' => 1, 'pid' => 0, title => '标题1'],
    ['id' => 2, 'pid' => 1, title => '标题1-1'],
]
```

转换为

```bash
[
    'id' => 1, 'pid' => 0, 'title' => '标题1',
    'children' => [
        'id' => 2, 'pid' => 1, 'title' => '标题1-1'
    ]
]
```

可设定 pid 和 id 字段名 可由方法参数传入

二、将数组某个字段渲染为树状的方法：

输出的数据如：

```bash
[
    'id' => 2,
    'pid' => 0,
    'title' => '权限管理',
    'name' => 'auth'
],
[
    'id' => 3,
    'pid' => 2,
    'title' => '    ├角色组管理',
    'name' => 'auth/group'
],
[
    'id' => 8,
    'pid' => 2,
    'title' => '    ├管理员管理',
    'name' => 'auth/admin'
]
```

`title` 字段为 `树枝` 字段，pid 和 id 为上下级根据，这三个字段都需要能够通过方法参数自定义


> 其实我的提示词已经非常详细了，不过还是和 AI 折腾了很久，比如 AI 以为一共只有两级等情况，不过好在最终结果是好的，完整代码位于 `pkg\tree\tree.go`

### 前端整合

直接要求 CC: 
1. `@web/src/api/admin/index.ts` 添加请求 `/admin/init` 的请求函数
2. `@web/src/layouts/admin/index.vue` 的 `init` 函数内，执行 `init` 请求函数，并填充数据至状态商店等，去除临时 MOCK 数据

得益于从 [BuildAdmin](https://gitee.com/wonderful-code/buildadmin)/web 迁移过来的前端，此时我们已经完成了后台布局前后两端的初始化工作，属于是有点样子了。