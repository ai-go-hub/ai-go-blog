# 从 PHP 到 AI + Golang，程序员自救转型手记（二十）：前端点选验证码组件实现

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “点选验证码包逐行目检”，本期将完成：前端点选验证码组件实现

# 前端点选验证码组件实现

### 原理

这里额外讲一下点选验证码的流程和原理：

1. 服务端预设了多张 `350x200` 的背景图片
2. 服务端准备了 ICON（带中文名称的图标文件）、中文文字、英文大写字母 作为验证 `元素`
3. 服务端随机抽取 1 张 `背景图片` 和 2 个 正确的 `元素`，2 个用于混淆的 `元素`，将它们以随机的角度和颜色绘制在单张图片上，并返回图片的 `base64` 至前端

图片大概长这样：

![](/images//20/1.png)

实际使用场景中，每一次刷新：验证码的背景图、元素都会改变，人类可以轻松找到正确元素并完成点击（程序记录点击坐标和数据库中的正确元素坐标对比），最终达到区分人类与机器的目的。

为尽可能安全，所以我们后续还需要增加接口 `节流` 功能，防止机器快速且暴力的获取到所有可用的 `背景` 和 `元素`，然后还建议开发者定期更换 `背景` 和 `元素` 等。

以登录接口为例，验证数据检查一般会分为两步：
1. 前端预检：将点选数据发送至 `/common/captcha/verify` 接口，接口返回预检结果，正确则继续流程，失败则刷新验证码，需用户重新操作。
2. 验证码预检正确时，验证码组件内会调用 `callback(captchaData)` 继续流程，如下：登录内接口需要接受组件回调的 `captchaData` 参数，服务端对其进行二次验证后才执行实际的业务逻辑，即第 1 步的预检此时应被认为是不可靠的，不能因为有预验就直接执行登录接口内的密码验证等工作。

```ts
clickCaptcha((captchaData) => {
    // 将点选数据 captchaData，通过 adminLogin 请求函数发送到服务端
    adminLogin({ ...loginForm, captcha: captchaData })
        .then((res) => {
            adminInfo.dataFill(res.data.data, false)
            router.push('/admin')
        })
})
```

```go
// 服务端调用 captcha.Check，第二个参数传递 true 表示验证后彻底删除验证码
if ok, err := captcha.Check(req.Captcha, true); !ok {
    if err != nil {
        return nil, err
    }
    return nil, errors.New("验证码错误")
}

// 验证通过后，才执行密码验证等登录逻辑...
```

### 前端组件实现

前端这边，只需要渲染服务端提供的图片，然后记录好用户点击了图片的那几个点即可，后端会完成点击坐标的正确性及点击顺序的验证（验证数据检查）。

BuildAdmin/web 已经实现了 clickCaptcha 组件，这里直接先拷贝过来让 AI 完成点选验证码组件和服务端接口的整合：

把能发现的明显问题都描述一下即可：
1. 优化我复制过来的 `web\src\components\clickCaptcha` 组件，使其适用于当前项目
2. 建立缺少的语言翻译，建立缺少的接口请求函数或全局样式
3. 完成获取验证码和前端预验证接口的对接，即 `/common/captcha/create` 和 `/common/captcha/verify`

纯前端代码，而且是技术栈相同的组件迁移，AI 做起来得心应手，最终做出来最大的问题有：

#### 验证数据格式改变

BuildAdmin 原始的 clickCaptcha 组件中，点选数据是拼接为了字符串进行传输，大概格式如：`1x,1y;2x,2y;350,200;`

拼接为字符串再传输至服务端验证，这一点本身没有任何问题，不过本项目服务端的 `验证数据检查函数` 本身不支持这种字符串的解析，基于四层架构的设计，我们应该在服务端完成解析，然后传递给 `验证数据检查函数`，由于未来可能会有不少接口使用点选验证码，且考虑到这种字符串可读性很差也没有实际上的加密效果，在本项目中我们将其改为 `验证数据检查函数` 直接支持的 `json` 格式，并在前端定义好数据类型，如下：

```ts
/**
 * 点击坐标
 */
export interface ClickPoint {
    x: number
    y: number
    element: string
}

/**
 * 验证请求数据
 */
export interface ClickRequest {
    key: string
    clicks: ClickPoint[]
    // 渲染图片宽度
    rendered_width: number
    // 渲染图片高度
    rendered_height: number
}
```

现在起，验证码预检接口接受的是如上 `验证请求数据` 格式数据，未来使用点选验证码的接口，也都接受该格式数据以便服务端使用 `captcha.Check(req.Captcha, true)` 直接复验。

### 最终完整代码

```ts
// src\components\clickCaptcha\index.ts 文件

import { createVNode, render } from 'vue'
import ClickCaptchaConstructor from './index.vue'

/**
 * 点击坐标
 */
export interface ClickPoint {
    x: number
    y: number
    element: string
}

/**
 * 验证请求数据
 */
export interface ClickRequest {
    key: string
    clicks: ClickPoint[]
    // 渲染图片宽度
    rendered_width: number
    // 渲染图片高度
    rendered_height: number
}

/**
 * 验证成功回调
 */
export type ClickCaptchaCallback = (data: ClickRequest) => void

interface ClickCaptchaOptions {
    class?: string
    error?: string
    success?: string
    // 自定义 API 基础 URL，默认使用 VITE_AXIOS_BASE_URL
    apiBaseURL?: string
}

/**
 * 弹出点选验证码
 * @param callback 验证成功的回调
 */
const clickCaptcha = (callback?: ClickCaptchaCallback, options: ClickCaptchaOptions = {}) => {
    const container = document.createElement('div')
    const vnode = createVNode(ClickCaptchaConstructor, {
        callback,
        ...options,
        onDestroy: () => {
            render(null, container)
        },
    })
    render(vnode, container)
    document.body.appendChild(container.firstElementChild!)
}

export interface Props extends ClickCaptchaOptions {
    callback?: ClickCaptchaCallback
}

export default clickCaptcha
```

```vue
<!-- src\components\clickCaptcha\index.vue 文件 -->

<template>
    <div>
        <div class="ai-go-click-captcha" :class="props.class">
            <div v-if="state.loading" class="loading">{{ i18n.global.t('common.loading') }}...</div>
            <div v-else class="captcha-img-box">
                <img
                    ref="captchaImgRef"
                    class="captcha-img"
                    :src="state.captcha.image_base64"
                    :alt="i18n.global.t('common.Captcha loading failed, please click refresh button')"
                    @click.prevent="onRecord($event)"
                />
                <span
                    v-for="(item, index) in state.clicks"
                    :key="index"
                    class="step"
                    @click="onCancelRecord(index)"
                    :style="`left:${item.x - 13}px;top:${item.y - 13}px`"
                >
                    {{ index + 1 }}
                </span>
            </div>
            <div class="captcha-prompt" v-if="state.tip">
                {{ state.tip }}
            </div>
            <div v-else class="captcha-prompt">
                {{ i18n.global.t('common.Please click') }}
                <span v-for="(text, index) in state.captcha.elements" :key="index" :class="state.clicks.length > index ? 'clicked' : ''">
                    {{ text }}
                </span>
            </div>
            <div class="captcha-refresh-box">
                <div class="captcha-refresh-line captcha-refresh-line-l"></div>
                <span class="captcha-refresh-btn" :title="i18n.global.t('common.refresh')" @click="load">⟳</span>
                <div class="captcha-refresh-line captcha-refresh-line-r"></div>
            </div>
        </div>
        <div class="ai-go-mask" @click="onClose"></div>
    </div>
</template>

<script setup lang="ts">
import { reactive, ref } from 'vue'
import { checkClickCaptcha, getClickCaptcha } from '/@/api/common'
import type { ClickPoint, ClickRequest, Props } from '/@/components/clickCaptcha/index'
import i18n from '/@/lang'
import { SYSTEM_ZINDEX } from '/@/stores/constant/common'

const props = withDefaults(defineProps<Props>(), {
    class: '',
    callback: () => {},
    error: i18n.global.t('common.The correct area is not clicked, please try again!'),
    success: i18n.global.t('common.Verification is successful!'),
})

const captchaImgRef = ref<HTMLImageElement | null>(null)

const state = reactive({
    tip: '',
    loading: true,
    clicks: [] as ClickPoint[],
    captcha: {
        key: '',
        elements: [] as string[],
        image_width: 350,
        image_height: 200,
        image_base64: '',
    },
})

const emits = defineEmits<{
    (e: 'destroy'): void
}>()

const load = () => {
    state.loading = true
    getClickCaptcha(props.apiBaseURL)
        .then((res) => {
            state.tip = ''
            state.clicks = []
            state.captcha = res.data.data
        })
        .finally(() => {
            state.loading = false
        })
}

const onRecord = (event: MouseEvent) => {
    if (state.clicks.length < state.captcha.elements.length) {
        state.clicks.push({
            x: event.offsetX,
            y: event.offsetY,
            element: state.captcha.elements[state.clicks.length],
        })
        if (state.clicks.length === state.captcha.elements.length) {
            const data: ClickRequest = {
                key: state.captcha.key,
                clicks: [...state.clicks],
                rendered_width: captchaImgRef.value!.width,
                rendered_height: captchaImgRef.value!.height,
            }
            checkClickCaptcha(data, props.apiBaseURL)
                .then(() => {
                    state.tip = props.success
                    setTimeout(() => {
                        props.callback?.(data)
                        onClose()
                    }, 1500)
                })
                .catch(() => {
                    state.tip = props.error
                    setTimeout(() => {
                        load()
                    }, 1500)
                })
        }
    }
}

const onCancelRecord = (index: number) => {
    state.clicks.splice(index, 1)
}

const onClose = () => {
    emits('destroy')
}

load()
</script>

<style scoped lang="scss">
.ai-go-click-captcha {
    padding: 12px;
    border: 1px solid var(--el-border-color-extra-light);
    background-color: var(--el-color-white);
    position: fixed;
    z-index: v-bind('SYSTEM_ZINDEX');
    left: calc(50% - v-bind('state.captcha.image_width + 24') / 2 * 1px);
    top: calc(50% - v-bind('state.captcha.image_height + 200') / 2 * 1px);
    border-radius: 10px;
    box-shadow:
        0 0 0 1px hsla(0, 0%, 100%, 0.3) inset,
        0 0.5em 1em rgba(0, 0, 0, 0.6);
    .loading {
        color: var(--el-color-info);
        width: 350px;
        text-align: center;
        line-height: 200px;
    }
    .captcha-img-box {
        position: relative;
        .captcha-img {
            width: v-bind('state.captcha.image_width') px;
            height: v-bind('state.captcha.image_height') px;
            border: none;
            cursor: pointer;
        }
        .step {
            box-sizing: border-box;
            position: absolute;
            width: 20px;
            height: 20px;
            line-height: 20px;
            font-size: var(--el-font-size-small);
            font-weight: bold;
            text-align: center;
            color: var(--el-color-white);
            border: 1px solid var(--el-border-color-extra-light);
            background-color: var(--el-color-primary);
            border-radius: 30px;
            box-shadow: 0 0 10px var(--el-color-white);
            user-select: none;
            cursor: pointer;
        }
    }
    .captcha-prompt {
        height: 40px;
        line-height: 40px;
        font-size: var(--el-font-size-base);
        text-align: center;
        color: var(--el-color-info);
        span {
            margin-left: 10px;
            font-size: var(--el-font-size-medium);
            font-weight: bold;
            color: var(--el-color-error);
            &.clicked {
                color: var(--el-color-primary);
            }
        }
    }
    .captcha-refresh-box {
        position: relative;
        margin-top: 10px;
        .captcha-refresh-line {
            position: absolute;
            top: 16px;
            width: 140px;
            height: 1px;
            background-color: #ccc;
        }
        .captcha-refresh-line-l {
            left: 5px;
        }
        .captcha-refresh-line-r {
            right: 5px;
        }
        .captcha-refresh-btn {
            cursor: pointer;
            display: block;
            margin: 0 auto;
            width: 32px;
            height: 32px;
            font-size: 24px;
            line-height: 32px;
            text-align: center;
            color: var(--el-color-info);
        }
    }
}
</style>
```

```ts
// src\api\common.ts 文件

import type { ClickRequest } from '/@/components/clickCaptcha/index'
import request from '/@/utils/request'

export function getClickCaptcha(apiBaseURL?: string) {
    return request({
        url: '/common/captcha/create',
        method: 'GET',
        ...(apiBaseURL ? { baseURL: apiBaseURL } : {}),
    })
}

export function checkClickCaptcha(data: ClickRequest, apiBaseURL?: string) {
    return request({
        url: '/common/captcha/verify',
        method: 'POST',
        data,
        ...(apiBaseURL ? { baseURL: apiBaseURL } : {}),
        requestOptions: {
            showErrorMessage: false,
        },
    })
}
```