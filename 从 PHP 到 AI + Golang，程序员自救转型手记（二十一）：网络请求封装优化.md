# 从 PHP 到 AI + Golang，程序员自救转型手记（二十一）：网络请求封装优化

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “前端点选验证码组件实现”，本期将完成：网络请求封装优化

# 网络请求封装优化

### 请求选项使用单独的参数传递

目前，如果需要使用我们自定义的请求选项，写法如下的 `requestOptions`：

```ts
export function checkClickCaptcha(data: ClickRequest, apiBaseURL?: string) {
    return request({
        url: '/common/captcha/verify',
        method: 'POST',
        requestOptions: {
            loading: true,
            cancelDuplicate: true,
            showErrorMessage: false,
        },
    })
}
```

即，使用 `request` 函数的第一个参数嵌套一个 `requestOptions` 来完成配置，我希望改为使用单独的参数进行传递，如下：

```ts
export function checkClickCaptcha(data: ClickRequest) {
    return request(
        {
            url: '/common/captcha/verify',
            method: 'POST',
            data,
        },
        {
            loading: true,
            cancelDuplicate: true,
            showErrorMessage: false,
        }
    )
}
```

参数 1 是 `axios` 的请求配置，参数 2 是我们自定义的请求选项，将其独立出来可读性、类型、扩展性、维护性全方位更优；以上内容是思路，同时是提示词，直接发送给 cc：

```bash
● 重构完成。改动总结如下：

修改的文件

1. src/utils/request.ts

- 删除了 RequestConfig 接口（已不需要，直接使用 AxiosRequestConfig）
- request 函数签名从 request(config: RequestConfig) 改为 request(config: AxiosRequestConfig, options?: RequestOptions)
- 内部将 options 合并到 config 的 requestOptions 字段上，拦截器逻辑无需任何改动

2. src/api/common.ts

- checkClickCaptcha 中的 requestOptions 从第一个参数嵌套中提取为独立的第二个参数：
// 之前
request({ url: '...', requestOptions: { showErrorMessage: false } })
// 之后
request({ url: '...' }, { showErrorMessage: false })

无需改动的文件

- src/api/admin/index.ts — 未使用 requestOptions，现有的单参数调用完全兼容
- 其他不涉及 requestOptions 的调用方同样无需改动，因为 AxiosRequestConfig 就是之前第一个参数去掉 requestOptions 后的类型

✻ Baked for 50s
```

人工 `review` 没发现不对的，其实总共也没改几行，反而提示词写了一大堆，失算了，😂😂

### 后端接口响应中的字段命名方式自动转为 camelCase

各个语言的服务端 json 字段命名，还有网络传输的惯例字段命名方式都是 `snake_case`，但是前端一般使用 `camelCase` 访问/存储字段值，所以在网络请求封装中对命名方式进行转换是比较常见的做法，这里要求 CC 增加 `转换字段命名方式` 的开关，默认打开，然后实现这个功能：

**全部完成。以下是变更总结：**

变更清单
`src/utils/request.ts` — 核心改造

|变更|说明|
|-|-|
|import { camelCase }|	复用已有的 lodash-es，无需新依赖|
|convertCase?: boolean|	新增请求选项，默认 true（开关打开）|
|keysToCamelCase()|	递归转换函数，处理对象、数组、嵌套结构|
|响应拦截器 L190-193|在 response.data.data 上自动执行转换|

接口 + Store + 模板统一改名

|文件|变更|
|-|-|
|src/stores/interface/index.ts|	last_login_at → lastLoginAt，last_login_ip → lastLoginIp|
|src/stores/adminInfo.ts|初始 state 同步改为 camelCase|
|src/layouts/admin/components/navMenu.vue|模板变量引用改为 adminInfo.lastLoginAt|

使用方式：

// 默认：自动转换（开关打开）

`request({ url: '/admin/init', method: 'GET' })`

// 某个接口不需要转换时，显式关闭

`request({ url: '/some/api', method: 'GET' }, { convertCase: false })`

现在前端所有接口的响应字段命名都是统一的 `camelCase` 了，不再有风格不一致的问题。

`review` 时，我直接让它把请求时的 `camelCase` 也反向转为 `snake_case`（受同一开关控制），至此前后端命名风格彻底解耦，本来是大型项目才标配的功能，也是让 AI 随手就搞了。

现在起：
1. `API` 响应 `snake_case` 将被转换为 `camelCase`；
2. 请求 `API` 时，传递 `camelCase`（包括 `data` 和 `params`）会转换为 `snake_case`。

前端可以总是使用 `camelCase` 了，非常舒服，比如：

1. 访问响应数据时，接口返回了：`res.data.data.admin_info.last_login_at` 字段，可以使用 `res.data.data.adminInfo.lastLoginAt` 访问响应数据；
2. 发起请求时，`data` 是：`{data: {adminInfo: {lastLoginAt: '2023-05-01'}}`，将被转换为：`{data: {admin_info: {last_login_at: '2023-05-01'}}` 发送。