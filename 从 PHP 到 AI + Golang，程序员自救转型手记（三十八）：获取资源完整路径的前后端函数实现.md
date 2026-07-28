# 从 PHP 到 AI + Golang，程序员自救转型手记（三十八）：获取资源完整路径的前后端函数实现

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “目录结构再一次调整、静态文件服务”，本期将完成：获取资源完整路径的前后端函数实现

# 获取资源完整路径的前后端函数实现

接下来我们需要一个 `获取资源完整路径` 的函数，分为前后端两个版本；

### 新增 CDN 相关配置项

于 `@config/config.yaml` 增加 `cdn_url` 和 `cdn_url_params` 配置项，类型均为字符串，注释如下：
  - `cdn_url`: 内容分发网络URL，末尾不带 `/`
  - `cdn_url_params`: 内容分发网络URL参数，将自动拼接到 `cdn_url` 的结尾（值如 `format/heif`）

```yaml
cdn:
  # 内容分发网络URL，末尾不带 `/`
  url:
  # 内容分发网络URL参数，将自动拼接到 url 的结尾（值如 `format/heif`）
  url_params:
```

### 后端 FullURL 函数实现

服务端这边的 `FullURL`，放在外层 `pkg` 并不合适，因为它不仅依赖配置，还依赖 `gin.Context`，能放 pkg 里边的起码要求和框架无强耦合才行。

所以我们选择新建 `kit/urlx` 包，函数名为 `FullURL`，要求如下：

1. 参数1 为 `c *gin.Context`，以便从中提取当前域名、端口和协议
2. 参数2 `resource`，即资源，类型为 string，可能是 `带协议的资源路径`、`资源相对路径`、`base64 资源`
3. `base64 资源` 和 `带协议的资源路径`，直接返回，否则返回资源带当前域名的完整路径
4. 若配置中的 `cdn_url` 和 `cdn_url_params` 有值，则不使用当前域名，而是 `cdn_url`（配置文件位置 `@config/config.yaml` ）

经过人工 `review` 和完善，最终代码如下：

```go
// FullURL 返回静态资源的完整访问 URL
func FullURL(c *gin.Context, resource string) string {
	if resource == "" {
		return ""
	}

	// 已是完整 URL 或 base64 资源，原样返回
	lower := strings.ToLower(resource)
	if strings.HasPrefix(lower, "http://") || strings.HasPrefix(lower, "https://") || strings.HasPrefix(lower, "data:") {
		return resource
	}

	cdn := config.Get().CDN

	// 选择前缀: CDN URL 优先，否则使用当前请求域名
	prefix := cdn.URL
	if prefix == "" {
		prefix = httpx.BaseURL(c)
	}

	// 前缀与路径之间保证恰好一个 /
	u := strings.TrimRight(prefix, "/") + "/" + strings.TrimLeft(resource, "/")

	// 拼接 CDN URL 参数（仅在使用 CDN 时）
	if cdn.URL != "" && cdn.URLParams != "" {
		sep := "?"
		if strings.Contains(u, "?") {
			sep = "&"
		}
		u += sep + cdn.URLParams
	}
	return u
}
```

### 前端 fullURL 函数实现

首先是需要将 `cdn_url` 和 `cdn_url_params` 配置项传递给前端，这一点可以利用已有的后台 `Init` 接口实现，额外传递两个字段即可，然后将这两项配置保存于 `配置状态商店`：`config.site.cdnUrl` 和 `config.site.cdnUrlParams`。

然后直接让 CC：

参考服务端 `FullURL` 函数，实现前端的 `fullURL` 函数：
1. `fullUrl` 函数封装在 `@web/src/utils/common.ts` 文件
2. `cdnUrl` 在 `config` 状态商店，`getBseeUrl()` 函数在 `src\utils\request.ts`

最终成品如下，功能基本是直接对齐服务端的 `FullURL`，额外多了个 `domain` 参数，开发者可以自行指定域名：

```ts
/**
 * 获取资源完整地址
 * @param resource 资源相对地址
 * @param domain 指定域名
 */
export const fullURL = (resource: string, domain = '') => {
    const config = useConfig()
    if (!domain) {
        domain = config.site.cdnUrl ? config.site.cdnUrl : getBaseUrl()
    }
    if (!resource) {
        return ''
    }

    const regUrl = new RegExp(/^http(s)?:\/\//)
    const regexData = new RegExp(/^data:/i)
    if (!domain || regUrl.test(resource) || regexData.test(resource)) {
        return resource
    }

    let url = domain + resource
    if (domain === config.site.cdnUrl && config.site.cdnUrlParams) {
        const separator = url.includes('?') ? '&' : '?'
        url += separator + config.site.cdnUrlParams
    }
    return url
}
```
