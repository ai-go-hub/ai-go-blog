# 从 PHP 到 AI + Golang，程序员自救转型手记（十五）：优化细节、网络请求封装

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “静态登录页制作”，本期将完成：优化细节、网络请求封装

# 优化细节

### 全局基本样式初始化

全局字体设定，还有部分浏览器标签默认样式消除等（比如 Chrome 浏览器的 body 标签，默认会有 8px 的 margin），建立 app.scss 文件，写入全局默认样式：

**src\styles\app.scss 文件**

```scss
*,
*::before,
*::after {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

html,
body {
    margin: 0;
    padding: 0;
    width: 100%;
    height: 100%;
    font-family:
        Helvetica Neue,
        Helvetica,
        PingFang SC,
        Hiragino Sans GB,
        Microsoft YaHei,
        SimSun,
        sans-serif;
    color: var(--el-text-color-primary);
    font-size: var(--el-font-size-base);
}
```

**再建立 `src\styles\element.scss` 文件**，写入全局的 element plus 样式优化代码：

```scss
/* 修复 Chrome 浏览器输入框内选中字符行高异常的问题 */
.el-input {
    .el-input__inner {
        line-height: calc(var(--el-input-height, 40px) - 4px);
    }
}
```

至此，我们的 `styles` 目录已经有 `loading.scss`、`element.scss`、`app.scss` 三个文件了，其中 `element.scss` 和 `app.scss` 都是需要全局引入的样式文件，我们可以直接于 `main.ts` 文件中逐一引入，但是考虑到未来这类全局样式文件还会增加，我们单独**建立一个 `index.scss` 文件**合并所有全局样式，然后 `main.ts` 内只导入它即可：

```scss
// src\styles\index.scss 文件

@use '/@/styles/app.scss';
@use '/@/styles/element.scss';
// 未来增加全局样式文件，在此增加一行即可，无需改动 main.ts 文件
```

**main.ts 增加一行代码即可**

```ts
import '/@/styles/index.scss'
```

### 路由切换更新浏览器标题

我们已经于静态路由配置中设定了各个路由的 title，接下来只需要在 `src\router\index.ts` 里边的 `路由加载后` 钩子中，将 `meta.title` 设置到浏览器标题栏即可：

```ts
// src\router\index.ts 文件

import { useTitle } from '@vueuse/core'

// 路由加载后
router.afterEach((to) => {
    if (window.loading) {
        loading.hide()
    }
    NProgress.done()

    // 设置浏览器标题
    const titleKey = to?.meta?.title as string | undefined
    const title = titleKey && i18n.global.te(titleKey) ? i18n.global.t(titleKey) : ''
    useTitle().value = title ? `${title} - AI GO MALL` : 'AI GO MALL'
})
```

先使用了 i18n.global.te 确定翻译 key 是存在的，然后才翻译，最后使用 `useTitle` 直接完成浏览器标题的设置。

# 网络请求封装

网络请求库当然是使用 axios，这是 90% 的项目在使用的网络请求库，我们主要需要封装的内容如下：

1. 请求 loading，可以配置一个网络请求是否显示全屏 loading 态，如果显示会在请求完成时自动关闭
2. 自动取消重复请求，根据请求的 get/param/data 等参数生成一个唯一的 key，如果一个请求未结束，再发起 key 相同的请求就会被自动取消，此功能也可以配置是否开启
3. 使用 TypeScript 实现，可以指定相应数据的类型
4. 操作成功/操作失败的自动提示开关，开启则自动使用 Element plus 的 API 弹出提示信息，成功默认不显示，失败默认显示
5. 自动携带用户 token（从 `stores\adminInfo.ts` 状态商店获取当前用户 token）

将以上需求发送给 cc，总共只建立了 `utils\request.ts` 一个文件，且未改动任何其他文件，人工 review 发现以下可以优化的地方：

1. 后端统一响应结构放到 `@types\common.d.ts` 里边去，定义参考服务端的 `@..\internal\response\response.go`（目前 AI 定义的响应结构是错误的）
2. 自定义成功/失败提示语去掉
3. 网络请求的 loading 不要使用首屏 loading，改用 Element Plus 的 ElLoading API
4. 封装单独的 getBaseUrl 和 getBaseUrlPort 函数，以便项目它处使用
5. 默认导出内部直接返回 axios 实例本身，不要：加一层 then 或将 request 函数改为异步函数
6. 请求的网络错误提示，单独一个开关，比如 404、500 错误，默认开启
7. 顶层变量归类，并使用对象存储，比如 loadingInstance 和 loadingCount 可以放到一个对象里边进行初始化

```bash
完成，变更如下：

types/common.d.ts — 新增全局接口 ApiResponse<T>，字段对齐后端 response.go（code/message/time/data），成功判断为 code === 0。

src/utils/request.ts — 四处修改：
1. Loading 换用 ElLoading.service({ fullscreen: true })，仍用引用计数管理并发请求
2. 去掉 successMsg / errorMsg 两个自定义提示选项，直接使用 res.message
3. ApiResponse 改用 types/common.d.ts 中的全局声明，不再本地 export
4. 新增 getBaseUrl / getBaseUrlPort 两个导出函数，用于在其他地方拼接完整请求地址（例如上传 action、文件下载链接等）
5. 默认导出内部直接返回 axios 实例本身
...
````

最后再对一些有顾虑的地方进行确定和补充：

1. 再次次分析 buildRequestKey，此函数建立的请求唯一 key 合理吗？如果不合理，如何优化？
2. 封装的部分自定义函数，逻辑，是否可以用 lodash-es 库中的函数代替

```bash
第1点完成。主要变更点：
- method → 统一 .toLowerCase()
- params → 普通对象按 key 排序后 stringify；URLSearchParams 调用 .sort() 再 .toString()
- data → 同 params 排序；字符串直接使用；FormData/Blob/ArrayBuffer 用 [object FormData] 等类型名标记（相同类型的请求仍会去重，若需区分并发上传请对该请求设 cancelDuplicate: false）
- 最终 key → 改用 JSON.stringify({ method, url, params, data }) 消除 & 分隔符碰撞问题


第2点：整个文件中只有一处值得替换：typeof config.data === 'string' → isString(config.data)，其余自定义逻辑用原生写法已是最简形式，引入 lodash-es 反而增加复杂度。单独为一行引入一个 import 成本大于收益，建议维持现状，不做替换。
```

利用 AI 封装网络请求完成，最终代码如下：

**types/common.d.ts** 增加了 ApiResponse 响应数据结构：

```ts
interface ApiResponse<T = any> {
    code: number
    message: string
    time: number
    data: T
}
```

对 **utils/request.ts** 的内容进行人工微调，比如：

```ts
const BASE_URL = import.meta.env.VITE_AXIOS_BASE_URL as string

export function getBaseUrl(): string {
    return BASE_URL
}

export function getBaseUrlPort(): string {
    return new URL(BASE_URL).port
}
```

修改为：

```ts
export function getBaseUrl(): string {
    return import.meta.env.VITE_AXIOS_BASE_URL as string
}

export function getBaseUrlPort(): string {
    return new URL(getBaseUrl()).port
}
```

避免顶层的 BASE_URL，不好看又徒增心智负担。`request.ts` 完整代码：

```ts
import type { AxiosInstance, AxiosRequestConfig, InternalAxiosRequestConfig } from 'axios'
import axios from 'axios'
import { ElLoading, ElMessage } from 'element-plus'
import i18n from '/@/lang'
import { useAdminInfo } from '/@/stores/adminInfo'

// ==================== 类型定义 ====================

export interface RequestOptions {
    // 是否显示全屏 loading，默认 false
    loading?: boolean
    // 是否自动取消重复请求，默认 true
    cancelDuplicate?: boolean
    // 是否显示操作成功提示，默认 false
    showSuccessMessage?: boolean
    // 是否显示业务错误提示（code !== 0），默认 true
    showErrorMessage?: boolean
    // 是否显示网络错误提示（HTTP 4xx/5xx 等），默认 true
    showNetworkErrorMessage?: boolean
}

export interface RequestConfig extends AxiosRequestConfig {
    requestOptions?: RequestOptions
}

interface InternalRequestConfig extends InternalAxiosRequestConfig {
    requestOptions?: RequestOptions
}

// ==================== Base URL 辅助函数 ====================

export function getBaseUrl(): string {
    return import.meta.env.VITE_AXIOS_BASE_URL as string
}

export function getBaseUrlPort(): string {
    return new URL(getBaseUrl()).port
}

// ==================== Loading ====================

const loadingState = {
    count: 0,
    instance: null as ReturnType<typeof ElLoading.service> | null,
}

function showLoading() {
    if (loadingState.count === 0) {
        loadingState.instance = ElLoading.service({ fullscreen: true })
    }
    loadingState.count++
}

function hideLoading() {
    loadingState.count = Math.max(0, loadingState.count - 1)
    if (loadingState.count === 0) {
        loadingState.instance?.close()
        loadingState.instance = null
    }
}

// ==================== 重复请求取消 ====================

interface PendingEntry {
    controller: AbortController
    hasLoading: boolean
}

const pendingMap = new Map<string, PendingEntry>()

function sortedStringify(obj: Record<string, any>): string {
    return JSON.stringify(Object.fromEntries(Object.entries(obj).sort(([a], [b]) => a.localeCompare(b))))
}

/**
 * 根据请求参数，为请求生成唯一标识
 */
function buildRequestKey(config: InternalAxiosRequestConfig): string {
    const method = (config.method ?? 'get').toLowerCase()
    const url = config.url ?? ''

    let params = ''
    if (config.params != null) {
        if (config.params instanceof URLSearchParams) {
            const copy = new URLSearchParams(config.params)
            copy.sort()
            params = copy.toString()
        } else {
            params = sortedStringify(config.params as Record<string, any>)
        }
    }

    let data = ''
    if (config.data != null) {
        if (typeof config.data === 'string') {
            data = config.data
        } else if (config.data instanceof FormData || config.data instanceof Blob || config.data instanceof ArrayBuffer) {
            // 无法稳定序列化，用类型名标记；如需区分多个并发上传请将 cancelDuplicate 设为 false
            data = Object.prototype.toString.call(config.data)
        } else {
            data = sortedStringify(config.data as Record<string, any>)
        }
    }

    return JSON.stringify({ method, url, params, data })
}

function addPending(config: InternalRequestConfig): void {
    const key = buildRequestKey(config)
    if (pendingMap.has(key)) {
        const { controller, hasLoading } = pendingMap.get(key)!
        controller.abort()
        if (hasLoading) hideLoading()
        pendingMap.delete(key)
        console.warn('[Request] The repeated request has been canceled:', key)
    }
    const controller = new AbortController()
    config.signal = controller.signal
    pendingMap.set(key, {
        controller,
        hasLoading: config.requestOptions?.loading ?? false,
    })
}

function removePending(config: InternalAxiosRequestConfig): void {
    pendingMap.delete(buildRequestKey(config))
}

// ==================== Axios 实例 ====================

const instance: AxiosInstance = axios.create({
    baseURL: getBaseUrl(),
    timeout: 10000,
})

instance.interceptors.request.use(
    (config: InternalRequestConfig) => {
        const opts = config.requestOptions ?? {}

        if (opts.cancelDuplicate !== false) addPending(config)
        if (opts.loading) showLoading()

        const adminInfo = useAdminInfo()
        if (adminInfo.token) {
            config.headers.set('Authorization', `Bearer ${adminInfo.token}`)
        }

        return config
    },
    (error) => Promise.reject(error)
)

instance.interceptors.response.use(
    (response) => {
        const config = response.config as InternalRequestConfig
        const opts = config.requestOptions ?? {}

        removePending(response.config)
        if (opts.loading) hideLoading()

        if (response.data.code !== 0) {
            if (opts.showErrorMessage !== false) {
                ElMessage.error(response.data.message || i18n.global.t('common.operationFailed'))
            }
            return Promise.reject(new Error(response.data.message || i18n.global.t('common.operationFailed')))
        }

        if (opts.showSuccessMessage) {
            ElMessage.success(response.data.message || i18n.global.t('common.operationSuccess'))
        }

        return response
    },
    (error) => {
        if (axios.isCancel(error)) return Promise.reject(error)

        const config = error.config as InternalRequestConfig | undefined
        const opts = config?.requestOptions ?? {}

        if (config) {
            removePending(error.config)
            if (opts.loading) hideLoading()
        }

        if (opts.showNetworkErrorMessage !== false) {
            const msg = (error.response?.data as ApiResponse | undefined)?.message ?? error.message ?? i18n.global.t('common.networkError')
            ElMessage.error(msg)
        }

        return Promise.reject(error)
    }
)

// ==================== 对外 API ====================

function request<T = any>(config: RequestConfig) {
    return instance<ApiResponse<T>>(config)
}

export default request
```