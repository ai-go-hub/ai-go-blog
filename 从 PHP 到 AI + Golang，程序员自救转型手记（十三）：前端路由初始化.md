# 从 PHP 到 AI + Golang，程序员自救转型手记（十三）：前端路由初始化

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “前端状态商店、多语言初始化”，本期将完成：前端路由初始化

# 前端路由初始化

vue 里边实现路由，使用 vue-router，核心部分大概只需要以下三行代码：

```ts
const router = createRouter({
    history: createWebHashHistory(),
    routes: staticRoutes,
})
```

如上，路由本身很简单，麻烦的地方在于建立第一个路由时，一般需要同时配置好：静态路由的自动发现、首屏 loading、路由切换进度条

### 静态路由的自动发现

路由功能规划：
1. 有一些预设好的静态路由，比如首页、登录页、404页，全部定义于 `/@/router/static.ts`，对应的组件，在合理位置建立空白 vue 组件即可
2. 系统能自动加载 `/@/router/static` 文件夹内的所有 ts 路由定义文件，自动合并到预设静态路由数组中，最终完成所有静态路由注册
3. 创建路由实例的逻辑写在 `/@/router/index.ts` 文件中，并于 `mian.ts` 完成路由插件注册

直接将规划发给 cc，基本一次性实现了所有需求，但还是人工整理了一下，因为写提示词还不如自己改的小细节，最终核心代码如下：

```ts
// src\router\static.ts

/*
 * 静态路由（支持自动扩展）
 * 系统会自动加载 ./static 目录及其子目录中的所有 .ts 文件
 * 每个模块的 default 导出可以是 RouteRecordRaw 或 RouteRecordRaw[]，自动 push 到以下 staticRoutes 数组
 */
const staticRoutes: Array<RouteRecordRaw> = [
    {
        path: '/',
        name: '/',
        component: () => import('/@/views/index.vue'),
        meta: {
            title: `pageTitles.Home`,
        },
    },
    // ...登录页，404 等其他静态路由
]

// 静态路由自动扩展逻辑
const staticFiles = import.meta.glob('./static/**/*.ts', { eager: true })
for (const path in staticFiles) {
    const module = staticFiles[path] as any
    if (!module.default) {
        console.warn(`[Router] Static route module ${path} does not export default, skipped`)
        continue
    }
    const route = module.default as RouteRecordRaw | RouteRecordRaw[]
    if (Array.isArray(route)) {
        staticRoutes.push(...route)
    } else {
        staticRoutes.push(route)
    }
}

export default staticRoutes
```

创建 `src\router\static\adminBase.ts` 测试静态路由自动扩展功能：

```ts
// src\router\static\adminBase.ts

import type { RouteRecordRaw } from 'vue-router'

/**
 * 后台基础路由路径
 */
export const adminBaseRoutePath = '/admin'

/*
 * 后台基础静态路由
 */
const adminBaseRoute: RouteRecordRaw = {
    path: adminBaseRoutePath,
    name: 'admin',
    component: () => import('/@/layouts/admin/index.vue'),
    // 直接重定向到 loading 路由
    redirect: adminBaseRoutePath + '/loading',
    meta: {
        title: `pageTitles.Loading`,
    },
    children: [
        {
            path: 'loading/:to?',
            name: 'adminMainLoading',
            component: () => import('/@/layouts/common/loading.vue'),
            meta: {
                title: `pageTitles.Loading`,
            },
        },
    ],
}

export default adminBaseRoute
```

`src\router\static\adminBase.ts` 的 `default` 导出 `adminBaseRoute` 会自动 push 到 `staticRoutes` 数组，然后通过 `src\router\index.ts` 的 `createRouter` 创建路由实例时完成注册。

静态路由涉及的 vue 文件，如 `/@/layouts/admin/index.vue`、`/@/views/index.vue` 等都已经由 cc 创建完毕，里边填充了空白 vue 组件的代码，等待后续完善。

### 路由切换进度条

单页应用加载完毕之后，后续切换页面就不会刷新了，没有加载页面的过程，浏览器就不会显示自带的进度条（或其他 loading 态）；

所以需要开发者自己写一个进度条，此进度条不代表真实的网页资源下载进度且路由切换往往很快，所以只需要在路由加载前显示，路由加载后隐藏即可；

我们选择使用 `nprogress` 实现路由切换进度条，它会在网页顶端显示一个从左到右的蓝色加载条

```ts
// src\router\index.ts 文件，即 createRouter 的文件

import NProgress from 'nprogress'
import 'nprogress/nprogress.css'
import { createRouter, createWebHashHistory } from 'vue-router'
import staticRoutes from '/@/router/static'

const router = createRouter({
    history: createWebHashHistory(),
    routes: staticRoutes,
})

// 路由加载前
router.beforeEach(() => {
    // 显示进度条
    NProgress.configure({ showSpinner: false })
    NProgress.start()
})

// 路由加载后
router.afterEach(() => {
    // 隐藏进度条
    NProgress.done()
})

export default router
```

### 首屏 loading

vue 单页应用，首次加载可能会很慢，我们需要尽快显示一个 loading 动画，避免长时间白屏，而且系统首次加载完毕后，再不显示此 loading 动画（路由切换不显示）。

loading 的显示只需要放到 vue-router 的 `路由加载前` 钩子中即可，整个系统的第一个路由加载前，就能触发到 loading 的显示，loading 的实现则直接使用原生 js 创建 div 即可（就不要加载 vue 组件什么的了）。

另外一个核心点在于首次加载完毕，loading 不会重复触发，所以需要一个标记，此标记的最佳实践是挂载到 window 对象上，先声明全局类型定义：

```ts
// common.d.ts 文件
interface Window {
    loading: boolean
}
```

更新 `src\router\index.ts` 触发首屏 loading 显示：

```ts
import NProgress from 'nprogress'
import 'nprogress/nprogress.css'
import { createRouter, createWebHashHistory } from 'vue-router'
import staticRoutes from '/@/router/static'
import { loading } from '/@/utils/loading'

const router = createRouter({
    history: createWebHashHistory(),
    routes: staticRoutes,
})

// 路由加载前
router.beforeEach(() => {
    NProgress.configure({ showSpinner: false })
    NProgress.start()

    // 显示首屏 loading
    if (!window.loading) {
        loading.show()
        window.loading = true
    }
})

// 路由加载后
router.afterEach(() => {
    // 隐藏首屏 loading，只要不标记 window.loading 为 false，它就永远不会再触发了
    if (window.loading) {
        loading.hide()
    }
    NProgress.done()
})

export default router
```

直接使用原生 js 创建 loading 的 div，挂载到 body 节点下：

```ts
// src\utils\loading.ts 文件
import { nextTick } from 'vue'
import '/@/styles/loading.scss'

export const loading = {
    show: () => {
        const div = document.createElement('div')
        div.className = 'ai-go-page-loading'
        div.innerHTML = `
            <div class="container">
                <div class="main">
                    <div class="item"></div>
                    <div class="item"></div>
                    <div class="item"></div>
                    <div class="item"></div>
                    <div class="item"></div>
                    <div class="item"></div>
                    <div class="item"></div>
                    <div class="item"></div>
                    <div class="item"></div>
                </div>
            </div>
        `
        document.body.insertBefore(div, document.body.childNodes[0])
    },
    hide: () => {
        nextTick(() => {
            setTimeout(() => {
                const el = document.querySelector('.ai-go-page-loading')
                el && el.parentNode?.removeChild(el)
            }, 1000)
        })
    },
}
```

```scss
// src\styles\loading.scss 文件
.ai-go-page-loading {
    width: 100%;
    height: 100%;
    position: fixed;
    z-index: 2147483600;
    background-color: #f5f5f5;
    .container {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        .main {
            width: 80px;
            height: 80px;
            .item {
                width: 33.333333%;
                height: 33.333333%;
                background: #409eff;
                float: left;
                animation: ai-go-page-loading-animation 1.2s infinite ease;
                border-radius: 1px;
            }
            .item:nth-child(7) {
                animation-delay: 0s;
            }
            .item:nth-child(4),
            .item:nth-child(8) {
                animation-delay: 0.1s;
            }
            .item:nth-child(1),
            .item:nth-child(5),
            .item:nth-child(9) {
                animation-delay: 0.2s;
            }
            .item:nth-child(2),
            .item:nth-child(6) {
                animation-delay: 0.3s;
            }
            .item:nth-child(3) {
                animation-delay: 0.4s;
            }
        }
    }
}

@keyframes ai-go-page-loading-animation {
    0%,
    70%,
    100% {
        transform: scale3D(1, 1, 1);
    }
    35% {
        transform: scale3D(0, 0, 1);
    }
}
```
