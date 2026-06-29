# 从 PHP 到 AI + Golang，程序员自救转型手记（十二）：前端状态商店、多语言初始化

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “前端工程初始化”，本期将完成：前端状态商店、多语言初始化

一些代码可直接从 [BuildAdmin](https://gitee.com/wonderful-code/buildadmin)/web 复制，部分代码就不需要 AI 再次生成了，这里只简单整理一遍，对于初学者可以跟着 git 提交顺序和文档去理解项目架构。

# 状态商店初始化

### pinia 注册

首先于 stores 文件夹建立 index.ts 用于初始化 pinia，并注册它的持久化插件 `pinia-plugin-persistedstate`：

```ts
// src\stores\index.ts
import { createPinia } from 'pinia'
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'

const pinia = createPinia()
pinia.use(piniaPluginPersistedstate)

export default pinia
```

在 main.ts 中，将 pinia 注册为 vue 插件：

```ts
// src\main.ts
import pinia from '/@/stores/index'

const app = createApp(App)
app.use(pinia)
```

### 第一个状态商店

我们已经设计好了服务端的管理员模型，此时不如直接将它对应的状态商店建好，后续管理员信息包括 token，都是直接使用 pinia 实现前端的持久化存储：

**定义 interface**

> PS：项目设计了 src\stores\interface 文件夹，专门存放状态商店的所有 interface，为了避免文件数量过多，项目自带的 interface，都尽量按分类放在已有的 ts 文件中，而不是建立很多文件，开发者自己的可以单独建立 ts 文件存放，比如自建 src\stores\interface\cms.ts 存放 cms 相关的 interface

如下的 adminInfo 接口是管理员模型定义的精简版，后续还需要额外的字段可以再加

```ts
// src\stores\interface\index.ts
export interface AdminInfo {
    id: number
    username: string
    nickname: string
    avatar: string
    last_login_at: string
    last_login_ip: string
    token: string
    // 是否是 superAdmin（用于判定是否显示超管级按钮，不做任何权限判断）
    super: boolean
}
```

**adminInfo 状态商店**

```ts
// src\stores\adminInfo.ts
import { defineStore } from 'pinia'
import { ADMIN_INFO } from '/@/stores/constant/cacheKey'
import type { AdminInfo } from '/@/stores/interface'

export const useAdminInfo = defineStore('adminInfo', {
    state: (): AdminInfo => {
        return {
            id: 0,
            username: '',
            nickname: '',
            avatar: '',
            last_login_at: '',
            last_login_ip: '',
            token: '',
            super: false,
        }
    },
    actions: {
        /**
         * 状态批量填充
         * @param state 新状态数据
         * @param [exclude=true] 是否排除某些字段（忽略填充），默认值 true 排除 token，传递 false 则不排除，还可传递 string[] 指定排除字段列表
         */
        dataFill(state: Partial<AdminInfo>, exclude: boolean | string[] = true) {
            if (exclude === true) {
                exclude = ['token']
            } else if (exclude === false) {
                exclude = []
            }

            if (Array.isArray(exclude)) {
                exclude.forEach((item) => {
                    delete state[item as keyof AdminInfo]
                })
            }

            this.$patch(state)
        },
        setToken(token: string) {
            this.token = token
        },
        removeToken() {
            this.token = ''
        },
    },
    persist: {
        key: ADMIN_INFO,
    },
})
```

# 多语言初始化

由于 [BuildAdmin](https://gitee.com/wonderful-code/buildadmin)/web 的多语言是根据路由按需加载的，目前本项目不需要此功能，所以让 ai 来实现多语言，提示词如下：

1. 使用已安装的 vue-i18n 依赖实现多语言功能，所有的语言包放置于 src/lang 目录下，分为中文和英文
2. 当前打包工具为 vite，多语言功能需要实现语言包的懒加载
3. 载入 element plus 的中英文语言包
4. 无需实现 legacy 模式的逻辑，语言包支持加载子级目录和文件

**结果**：
1. 由于项目目前还没有建立 config 状态商店，所以它给建立了一个 locale 状态商店专门存储当前语言，并且写了 `setLocale` 之类的函数来设置语言，暂时将这些都去掉，使用固定的 zh-cn，后续做了 config 再动态化
2. 它还写了一个 deepMerge 函数，实际上使用 lodash-es 的 merge 函数即可
3. 语言包按需加载的核心逻辑是 `en: () => import('./en')` 然后 `await en()`，据我所知在 Vite 中，这种方式是支持不了子目录的，只把 `en` 文件夹里边的所有 ts 文件加载到了，比如 `en/test.ts`，但 `en/test/test.ts` 加载不到，经过测试也确实如此

将整理出来的**结果**直接作为提示词发给 cc，新的一轮中，语言包加载改为了 `import.meta.glob('./zh-cn/**/*.ts')`，看起来没问题了，然后它又写了一个 `将文件路径转换为嵌套的 key 路径` 函数，如下：

```ts
/**
 * 将文件路径转换为嵌套的 key 路径
 * 例如：./zh-cn/test/test1.ts → ['test', 'test1']
 *      ./zh-cn/common.ts → ['common']
 *      ./zh-cn/index.ts → []（顶层合并）
 */
function filePathToKeys(locale: AppLocale, filePath: string): string[] {
    // 去掉语言目录前缀和 .ts 后缀
    const relativePath = filePath.replace(`./${locale}/`, '').replace('.ts', '')
    // index 作为顶层，其余按目录层级拆分
    if (relativePath === 'index') {
        return []
    }
    return relativePath.split('/')
}
```

路径转嵌套 key 之前自己写过，当时 AI 还没出生，算了，回头用自己的，这么多年了，稳；加上这哥们刚刚往我项目建了 30 多个文件用于测试和示例，给我气笑了，幸好立项就配好了 git，此时此刻放弃所有工作区的更改，毫无疑问是最合理的选择；多语言初始化最终由作者使用古法编程手搓，核心代码如下：

先建立了 `config` 状态商店，使用 `config.lang.active` 存储当前激活语言，然后于 `App.vue` 配置好了 `element plus` 的多语言：

```vue
<template>
    <el-config-provider :value-on-clear="() => null" :locale="elLocale">
        <router-view></router-view>
    </el-config-provider>
</template>

<script setup lang="ts">
import elEn from 'element-plus/es/locale/lang/en'
import elZhCn from 'element-plus/es/locale/lang/zh-cn'
import { computed } from 'vue'
import { useConfig } from '/@/stores/config'

const config = useConfig()

const elLocales: Record<string, typeof elEn> = {
    en: elEn,
    'zh-cn': elZhCn,
}

const elLocale = computed(() => elLocales[config.lang.active] || elLocales[config.lang.fallback])
</script>
```

项目的其他多语言加载核心逻辑如下：

```ts
// src\lang\index.ts
// 实现了语言包懒加载、子目录语言包加载、当前语言动态修改函数
import { merge, set } from 'lodash-es'
import type { App } from 'vue'
import { createI18n } from 'vue-i18n'
import { useConfig } from '/@/stores/config'

/**
 * 支持的语言类型
 */
export type LangKey = 'zh-cn' | 'en'

/**
 * 支持的语言列表
 */
export const langs: LangKey[] = ['zh-cn', 'en']

/**
 * 语言显示名称
 */
export const langNames: Record<LangKey, string> = {
    en: 'English',
    'zh-cn': '简体中文',
}

/**
 * i18n 实例
 */
const i18n = createI18n({
    legacy: false,
    locale: 'zh-cn',
    fallbackLocale: 'zh-cn',
    messages: {},
})

// 使用 vite import.meta.glob 批量导入 lang 目录下所有 .ts 文件（包括子目录）
const langGlobs: Record<LangKey, Record<string, () => Promise<{ default: any }>>> = {
    en: import.meta.glob('./en/**/*.ts') as Record<string, () => Promise<{ default: any }>>,
    'zh-cn': import.meta.glob('./zh-cn/**/*.ts') as Record<string, () => Promise<{ default: any }>>,
}

/**
 * 设置 i18n，并为 vue 安装 i18n 插件
 */
export async function setupI18n(app: App): Promise<void> {
    const config = useConfig()
    i18n.global.fallbackLocale.value = config.lang.fallback

    // 初始化当前语言包
    await setLang(config.lang.active)

    app.use(i18n)
}

/**
 * 设置语言
 * @param lang 语言标识
 */
export async function setLang(lang: LangKey): Promise<void> {
    await loadMessages(lang)

    const config = useConfig()
    i18n.global.locale.value = lang
    config.setLang(lang)
}

/**
 * 懒加载语言包
 * @param lang 语言标识
 */
export async function loadMessages(lang: LangKey): Promise<void> {
    // 如果已加载则跳过
    if (i18n.global.availableLocales.includes(lang)) {
        return
    }

    try {
        // 批量加载 lang 目录下所有 .ts 文件
        const glob = langGlobs[lang]
        const promises = Object.entries(glob).map(async ([path, loader]) => {
            const module = await loader()
            return { path, default: module.default }
        })
        const modules = await Promise.all(promises)

        // 按文件路径构建嵌套的 messages 结构
        const mergedMessages: Record<string, any> = {}
        for (const { path, default: moduleData } of modules) {
            if (typeof moduleData !== 'object' || moduleData === null) {
                continue
            }
            const keys = filePathToKeys(lang, path)
            if (keys.length === 0) {
                // 合并到顶层
                merge(mergedMessages, moduleData)
            } else {
                // 子模块 — 按路径嵌套
                merge(mergedMessages, set({}, keys, moduleData))
            }
        }

        i18n.global.setLocaleMessage(lang, mergedMessages)
    } catch (error) {
        console.error(`Failed to load lang: ${lang}`, error)
    }
}

const filePathToKeys = (lang: LangKey, path: string) => {
    const langPathPrefix = `/${lang}`
    const pathName = path.slice(path.lastIndexOf(langPathPrefix) + (langPathPrefix.length + 1), path.lastIndexOf('.'))
    const keys = pathName.split('/')

    // index.ts 作为顶层，其余按目录层级拆分
    if (keys.length === 1 && keys[0] === 'index') {
        return []
    }
    return keys
}

export default i18n
```