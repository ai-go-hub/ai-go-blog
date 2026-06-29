# 现代前端的极致性能 icon 加载方案（死磕成功版）

中大项目中，有很多动态设置 `icon` 的场景，比如通过管理页面：动态设置菜单的 icon，条目的 icon

此时就会遇到一个问题，我们希望有很多 `icon` 可供使用，但是又希望这些 `icon` 是按需加载的。

这是两个矛盾的场景，即要在 `图标选择器组件` 加载所有图标，平时图标的显示又是按需加载的，目前已有的方案均不能实现需求。

你说不对啊，按需导入使用不就行了吗？

**按需导入写法：**

```vue
<script setup>
import { Smile } from "@lucide/vue";
</script>

<template>
    <Smile color="#3e9392" />
</template>
```

以上写法是不能用于动态设置 icon 的，因为 icon 的 name 被写入配置，你需要以 name 字符串渲染出 icon，而 import 语句不支持变量。

**目标写法：**

```vue
<script setup lang="ts">
const name = 'smile'
</script>

<template>
    <!-- lucide 图标 -->
    <Icon :name="`lucide-${iconName}`" color="#3e9392" size="24" />

    <!-- Element Puls 图标 -->
    <Icon :name="`el-${iconName}`" color="#3e9392" size="24" />
</template>
```

## `import()` 函数方案

我想到的第一个方案，是 import() 函数，像这样：

```ts
const mod = await import(`@lucide/vue/dist/icons/${name}.js`)
```

拿到 icon name 的 string 之后，过一遍单独的图标加载函数就行了，然而实测中发现这个方案问题太多了😂

1. **频繁网络请求**：我们的 `图标选择器组件` 需要加载所有的图标供用户选择，假设有 2000 个，那么该页会触发 2000 个请求，处理不好重复的图标会重复发起请求！
2. **打包产物碎片化**：打包工具会静态拆分独立的供后续异步加载的 chunk，2000 个图标会拆出来 2000 个 chunk，构建产物文件数量巨增！
3. **渲染卡顿、闪烁**：由于图标分的实在是太细了，await 异步加载的图标一多，容易出现卡顿问题。

## 其他常用方案

#### `unplugin-auto-import` 静态编译按需打包

只打包页面写死使用的图标，打包时就确定依赖，不能处理运行时变量图标名称，我们的 `图标选择器组件` 会导致图标提前全量加载。

#### 图标组件全局注册

这就是全量加载，只是使用方便，没有按需加载的特性。

#### import.meta.glob

效果和 `import()` 函数类似。

## 思考

自己结合 AI，想出了很多可能性，比如：

1. 在编译期间，确定使用了那些图标，然后写入到一个文件内记录，程序内再完成动态注册，是一种别样的按需加载。可惜突然发现这种方式我们的 `图标选择器组件` 还是会导致图标全量加载。
2. 写个脚本，将 icon 按首字母分组，提前制备 chunk 文件，然后 import 时，根据图标首字母按需加载不同的 chunk。这种方式还行，只是有日常维护成本。
3. ......

## 最终方案

提前制备 chunk 文件给了我灵感，我们可以做个 Vite 插件，将图标按首字母范围拆分为 5 个虚拟模块，每个虚拟模块被 import 时产生一个独立 chunk，不会存在维护成本，因为是虚拟模块，chunk 不需要落盘。

Lucide 图标库的拆分方案如下：

```bash
┌───────────────────────┬──────────────┬────────┬────────────────────┐
│         Chunk         │  min / gzip  │ 图标数  │      触发条件      │
├───────────────────────┼──────────────┼────────┼────────────────────┤
│ a-c-*.js              │ 140 / 36 KB  │ 514    │ 首字母 a–c          │
├───────────────────────┼──────────────┼────────┼────────────────────┤
│ d-l-*.js              │ 124 / 33 KB  │ 424    │ 首字母 d–l          │
├───────────────────────┼──────────────┼────────┼────────────────────┤
│ m-p-*.js              │ 73 / 20 KB   │ 262    │ 首字母 m–p          │
├───────────────────────┼──────────────┼────────┼────────────────────┤
│ q-s-*.js              │ 95 / 24 KB   │ 315    │ 首字母 q–s          │
├───────────────────────┼──────────────┼────────┼────────────────────┤
│ t-z-*.js              │ 62 / 18 KB   │ 223    │ 首字母 t–z          │
├───────────────────────┼──────────────┼────────┼────────────────────┤
│ createLucideIcon-*.js │ 1.3 / 0.8 KB │ —      │ 公共依赖（仅一次）   │
└───────────────────────┴──────────────┴────────┴────────────────────┘
```

使用了 `图标选择器组件` 时，所有的图标都会被加载（5 个 Chunk）；

未使用 `图标选择器组件` 的页面，根据图标首字母的不同，系统会按需的加载对应的 Chunk，没使用的 Chunk 不会加载，首屏不加载，总 Chunk 数非常少，不会造成打包产物碎片化。

> 开发环境下还是全量加载图标包即可，无需拆分。

## 测试

#### 对比数据准备

由于需要和未实现按需加载之前做对比，我将本次工作区的全部改动使用 `git stash` 暂存起来，恢复工作区至初始状态（未加载图标库的状态）。

未加载任何图标前，浏览器禁用缓存，总是使用同一 vue 页面测试：

开发环境下：47 个请求，传输 4.7MB，4.7MB 项资源。

构建产物：`assets` 内共 17 个项目，1.55MB；14 个请求，传输 1.6MB，1.6MB 项资源，同时记录下加载的较大文件以做对比：

http://localhost:8000/assets/index-Cww9--Nk.js `1078kb`
http://localhost:8000/assets/vue.runtime.esm-bundler-58Gmz7_F.js `111kb`
http://localhost:8000/assets/style-C1fZPW92.css `370kb`

> 以上数据是未加载图标库的，如果全量加载图标库，应该会增加 500-800kb（未启用 gzip）

#### 方案数据

> 首次测试渲染了一个 a 开头的图标

开发环境下：51 个请求，传输 5.5MB，5.5MB 项资源。

构建产物：`assets` 内共 24 个项目，2.17 MB；17 个请求，传输 1.9MB，1.9MB 项资源，上 100kb 的文件多了以下一个：

http://localhost:8000/assets/a-c-CdQL3xxB.js `144kb`

请求数合理，请求大小合理，宣布方案成功，接下来逐一使用 a d m q t 开头的图标，并重新编译，确定以下文件是按需加载的，文件大小分布也较为合理：

- http://localhost:8000/assets/a-c-Byxc3NbE.js `144kb`
- http://localhost:8000/assets/d-l-D6J0NQGY.js `127kb`
- http://localhost:8000/assets/m-p-Cti2KRmr.js `75.2kb`
- http://localhost:8000/assets/q-s-CGvOdbjT.js `97.3kb`
- http://localhost:8000/assets/t-z-d79zgzJ8.js `63.5kb`

## 代码分享

完整代码开源于：[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)

**types/lucide.d.ts 文件：**

```ts
declare module 'virtual:lucide-icons/*' {
    import type { Component } from 'vue'
    const icons: Record<string, Component>
    export = icons
}
```

**src\components\icon\vitePlugin.ts 文件：**

```ts
import { readdirSync } from 'fs'
import { camelCase, upperFirst } from 'lodash-es'
import { join } from 'path'
import type { Plugin } from 'vite'

/**
 * Lucide icon 模块拆分加载插件
 * Lucide 模块总大小约 615kb，gzip 压缩后 128kb
 * 插件实现了生产环境下 Lucide 模块的拆分加载，将 Lucide 图标按首字母范围拆分为 5 个虚拟模块
 * 每个虚拟模块被 import 时产生一个独立 chunk
 */
export function lucideIconSplitPlugin(): Plugin {
    const VIRTUAL_PREFIX = 'virtual:lucide-icons/'
    const RESOLVED_PREFIX = '\0' + VIRTUAL_PREFIX

    const BATCHES: Record<string, RegExp> = {
        'a-c': /^[a-c]/,
        'd-l': /^[d-l]/,
        'm-p': /^[m-p]/,
        'q-s': /^[q-s]/,
        't-z': /^[t-z0-9]/,
    }

    let iconFiles: string[]

    return {
        name: 'lucide-icon-batches',
        enforce: 'pre',

        configResolved() {
            const dir = join(process.cwd(), 'node_modules/@lucide/vue/dist/esm/icons')
            try {
                iconFiles = readdirSync(dir).filter((f) => f.endsWith('.mjs') && f !== 'index.mjs')
            } catch {
                iconFiles = []
            }
        },

        resolveId(id) {
            if (id.startsWith(VIRTUAL_PREFIX)) {
                return RESOLVED_PREFIX + id.slice(VIRTUAL_PREFIX.length)
            }
        },

        load(id) {
            if (!id.startsWith(RESOLVED_PREFIX)) return

            const batchName = id.slice(RESOLVED_PREFIX.length)
            const pattern = BATCHES[batchName]
            if (!pattern || !iconFiles) return 'export {}'

            const matched = iconFiles.filter((f) => pattern.test(f.replace('.mjs', '')))

            // 使用显式 import + 具名 export 而非 re-export，
            // 确保 Rolldown 将所有图标模块内联到同一个 chunk 中
            const imports: string[] = []
            const exports: string[] = []
            matched.forEach((f, i) => {
                const pascalName = upperFirst(camelCase(f.replace('.mjs', '')))
                const varName = `_${i}`
                imports.push(`import ${varName} from '@lucide/vue/dist/esm/icons/${f}'`)
                exports.push(`export const ${pascalName} = ${varName}`)
            })

            return imports.join('\n') + '\n' + exports.join('\n')
        },
    }
}
```

**src\components\icon\index.ts 文件：**

```ts
import * as elIcons from '@element-plus/icons-vue'
import { camelCase, kebabCase, upperFirst } from 'lodash-es'
import { App, defineAsyncComponent, type Component } from 'vue'

type IconMap = Record<string, Component>

const lucideCache: IconMap = {}

export function getLucideComponent(name: string): Component | null {
    const key = upperFirst(camelCase(name))
    if (lucideCache[key]) {
        return lucideCache[key]
    }

    let loader: () => Promise<Component | { render: () => null }>

    if (import.meta.env.DEV) {
        let iconsPromise: Promise<IconMap> | null = null
        loader = () => (iconsPromise ??= import('@lucide/vue').then((m) => m.icons as IconMap)).then((icons) => icons[key] || { render: () => null })
    } else {
        const batchLoaders: Record<string, () => Promise<IconMap>> = {
            'a-c': () => import('virtual:lucide-icons/a-c') as Promise<IconMap>,
            'd-l': () => import('virtual:lucide-icons/d-l') as Promise<IconMap>,
            'm-p': () => import('virtual:lucide-icons/m-p') as Promise<IconMap>,
            'q-s': () => import('virtual:lucide-icons/q-s') as Promise<IconMap>,
            't-z': () => import('virtual:lucide-icons/t-z') as Promise<IconMap>,
        }

        const first = key.charAt(0).toLowerCase()
        const batch = 'abc'.includes(first)
            ? 'a-c'
            : 'defghijkl'.includes(first)
              ? 'd-l'
              : 'mnop'.includes(first)
                ? 'm-p'
                : 'qrs'.includes(first)
                  ? 'q-s'
                  : 't-z'

        loader = () => batchLoaders[batch]().then((icons) => icons[key] || { render: () => null })
    }

    const asyncComp = defineAsyncComponent(loader)
    lucideCache[key] = asyncComp
    return asyncComp
}
```


**src\components\icon\index.vue 文件：**

```vue
<script lang="ts">
import { createVNode, defineComponent, h, resolveComponent } from 'vue'
import { getLucideComponent } from './index'

export default defineComponent({
    name: 'Icon',
    props: {
        name: {
            type: String,
            required: true,
        },
        size: {
            type: [Number, String],
            default: 24,
        },
        color: {
            type: String,
            default: undefined,
        },
        strokeWidth: {
            type: [Number, String],
            default: undefined,
        },
    },
    setup(props, { attrs }) {
        // Element Plus 的 icon，通过 registerIcons 全局批量注册为 `el-icon-kebabCase(name)` 的组件
        if (props.name.indexOf('el-') === 0) {
            const name = props.name.replace('el-', 'el-icon-')
            return () => {
                return createVNode(
                    resolveComponent('el-icon'),
                    { class: 'icon ai-go-icon', ...props, ...attrs },
                    { default: () => h(resolveComponent(name)) }
                )
            }
        }

        // lucide 的 icon，lucideIconSplitPlugin 将按首字母将全部 lucide icon 分为虚拟包，并按需加载
        if (props.name.indexOf('lucide-') === 0) {
            const name = props.name.replace('lucide-', '')
            return () => {
                const component = getLucideComponent(name)
                if (component) {
                    return h(component, { ...props, ...attrs })
                }
            }
        }
    },
})
</script>
```

**vite.config.ts 文件（增加 lucideIconSplitPlugin 插件的注册）：**

```ts
import vue from '@vitejs/plugin-vue'
import { resolve } from 'path'
import type { ConfigEnv, UserConfig } from 'vite'
import { loadEnv } from 'vite'
import { lucideIconSplitPlugin } from './src/components/icon/vitePlugin'

// https://vitejs.cn/config/
const viteConfig = ({ mode }: ConfigEnv): UserConfig => {
    const { VITE_PORT, VITE_OPEN, VITE_BASE_PATH, VITE_OUT_DIR } = loadEnv(mode, process.cwd())

    return {
        plugins: [vue(), lucideIconSplitPlugin()],
        root: process.cwd(),
        resolve: {
            alias: {
                '/@': resolve(__dirname, 'src'),
            },
        },
        base: VITE_BASE_PATH,
        server: {
            port: parseInt(VITE_PORT),
            open: VITE_OPEN != 'false',
        },
        build: {
            cssCodeSplit: false,
            sourcemap: false,
            outDir: VITE_OUT_DIR,
            emptyOutDir: true,
            chunkSizeWarningLimit: 1500,
        },
    }
}

export default viteConfig
```