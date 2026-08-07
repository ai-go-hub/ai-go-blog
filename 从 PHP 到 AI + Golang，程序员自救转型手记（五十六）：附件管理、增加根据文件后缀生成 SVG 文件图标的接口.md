# 从 PHP 到 AI + Golang，程序员自救转型手记（五十六）：附件管理、增加根据文件后缀生成 SVG 文件图标的接口

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “增加 area 接口和区域选择组件”，本期将完成：附件管理、增加根据文件后缀生成 SVG 文件图标的接口

# 增加根据文件后缀生成 SVG 文件图标的接口

这也是一个工具接口，作用是传入一个文件后缀，比如 `zip`，接口直接输出一张 `svg` 图片，这样的：

![](./images/56/1.png)

这个接口的意义是放在后续要做的 `附件管理` 里边，作为 `图片` 以外类型的文件的预览图。

这里还是让 AI 参考了 BuildAdmin 相应的接口，直接实现 GO 版本：参考 `@../ba238/app/api/controller/Ajax.php` 的 `buildSuffixSvg` 方法及其内部实现，在 `@internal\handler\common\util.go` 制作一个能够直接返回 `svg` 的方法，注册为 `/common/file-svg` 路由


AI做完，人工 `review` 时发现在 GO 里边，直接输出 SVG 非常简单（以前我也没写过）：

```go
c.Header("Content-Type", "image/svg+xml; charset=utf-8")
c.Header("Cache-Control", "public, max-age=604800")
c.String(http.StatusOK, buildSuffixSvg(suffix, background))
```

如上，设置两个 `Header`，其中一个还是设置缓存的，然后就直接 `c.String` 就行了，`buildSuffixSvg` 本身返回的是 `SVG` 的 `string` 内容，核心代码如下：

```go

r, g, b := hsvToRGB(float64(hue)/360.0, 0.3, 0.9)

if background == "" {
    background = fmt.Sprintf("rgb(%d,%d,%d)", r, g, b)
}

return fmt.Sprintf(`<svg version="1.1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px" viewBox="0 0 512 512" style="enable-background:new 0 0 512 512;" xml:space="preserve">
            <path style="fill:#E2E5E7;" d="M128,0c-17.6,0-32,14.4-32,32v448c0,17.6,14.4,32,32,32h320c17.6,0,32-14.4,32-32V128L352,0H128z"/>
            <path style="fill:#B0B7BD;" d="M384,128h96L352,0v96C352,113.6,366.4,128,384,128z"/>
            <path style="fill:%s;" d="M416,416c0,8.8-7.2,16-16,16H48c-8.8,0-16-7.2-16-16V256c0-8.8,7.2-16,16-16h352c8.8,0,16,7.2,16,16 V416z"/>
            <g><text><tspan x="220" y="380" font-size="124" font-family="Verdana, Helvetica, Arial, sans-serif" fill="white" text-anchor="middle">%s</tspan></text></g>
        </svg>`, background, suffix)
```

可以看到额外还调用了一个 `hsvToRGB` 函数，它的作用是将 `HSV` 颜色转换（映射）为固定的 `RGB` 整数值，目的为了实现：传入相同后缀时，图片背景颜色的一致性，比如：传入 ZIP 背景是紫色，下次传入还得是相同的紫色，不能随机生成背景颜色，不然页面上多显示几个 `SVG`，就花里胡哨的了。

# 附件管理

用户上传的文件就是附件，我们之前实现的 `上传接口` 已经带有：`向附件表写入上传记录的功能` 了，本次只是实现对应数据表的后台管理功能，提示词如下：

参考 `@../ba238/web/src/views/backend/routine/attachment/index.vue` 实现后台附件管理 `CRUD`，可以参考本项目中的：服务端 `@internal/handler/admin/auth/admin.go` 前端 `@web/src/views/admin/auth/admin/index.vue`，附件管理的数据迁移和模型位于 `@cmd/migrate/migrations/000002_common.up.sql` `@internal/model/common.go`

这次的 `CRUD` 不尽人意：

1. 前端缺少了英文的语言包
2. `created_at` 字段的翻译也没有使用 `common.createdAt`，而是额外在语言包定义了一次
3. 它还给文件位置放错了：路由、服务、控制器等，全部放在了 `common`，应该放在 `admin` 才对，我们毕竟是在做后台管理功能，不过附件的仓储放在 `common` 是合理的，因为上传接口也在用

以上问题让 AI 逐一改正，并且人工 `review` 之后，下一步我们需要实现：在后台删除一个附件记录时，同时删除对应的文件，让 AI 覆写服务层 Delete 方法：

覆写 `internal\service\admin\routine\attachment.go` 的 `Delete` 方法，删除附件时，同时实例化 `driver` 字段对应的驱动，使用驱动删除对应的文件：
1. 附件模型位于 `internal\model\common.go` 的 `Attachment`
2. 附件全部驱动位于 `internal\infra\upload\driver` 文件夹，不知道怎么用驱动还可以参考 `internal\infra\upload\upload.go`

全部完工，人工 `review` 和测试通过：我们得到了如下附件管理功能：

![](./images/56/2.png)

![](./images/56/3.png)