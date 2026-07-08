# 从 PHP 到 AI + Golang，程序员自救转型手记（二十六）：icon 组件封装

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “后台布局迁移尝试”，本期将完成：icon 组件封装

# icon 组件封装

上一期尝试让 AI 直接完成后台布局相关的迁移失败了，手动迁移过程发现，布局相关已经会使用很多的 `icon`，所以先来完成 `icon` 组件的封装。

核心需求其实就是使用 `name` 属性就能渲染出不同的 `icon`：

原来的写法：

```vue
<script setup>
import { Smile } from "@lucide/vue";
</script>

<template>
  <Smile color="#3e9392" />
</template>
```

目标的写法：

```vue
<template>
    <!-- lucide 图标 -->
    <Icon name="lucide-smile" color="#3e9392" />

    <!-- element puls 图标 -->
    <Icon name="el-smile" color="#3e9392" />
</template>
```

即单行代码即可显示图标，无需手动导入，这种实现的好处之一就是后台的菜单的图标可以随时修改。

#### 图标选型

让 AI 找了很多高星的图标库项目，其实挺看好 `iconpark` 的，可惜他们的 github 仓库已经归档了，最后选了 `lucide`，因为他 star 多，现代，vue3 兼容性好，图标总量不多不少。

#### 图标加载

我建立了 `@src\components\icon\index.vue` 图标组件，然后尝试使用 `resolveComponent("图标名")` 直接进行加载，失败了，很合理😂，接下来让 CC：

```bash
@src\components\icon\index.vue 中的 Smile 是来自 @lucide/vue 库的图标，我希望后续使用该库的图标，免去 import { Smile } from "@lucide/vue"，直接使用 resolveComponent('Smile') 之类的方式使用（即提供一个图标名即可渲染）
```

这么一个需求，对于 cc 来讲当然不在话下，它额外提醒我，构建产物体积会增加很多：

```bash
改造要点 @lucide/vue 库导出了一个 icons 对象，它是一个 图标名 → 组件 的映射表（如 { Smile: FunctionalComponent, ArrowRight: FunctionalComponent, ... }），直接用它即可按名渲染，无需全局注册或逐个 import。

导入 icons 会将 所有 Lucide 图标打入包中（约 3000 个），这是按名动态使用图标的必然代价。如果后续包体积成为问题，可考虑使用 Vite 的 import.meta.glob 做按需分块加载。
```

我又让它确定一下，具体会增加多大，看看在不在可接受范围以内：

```bash
  ● 实测结果

  ┌─────────────────────────────┬─────────┐
  │            指标             │  大小   │
  ├─────────────────────────────┼─────────┤
  │ 全部 1737 个图标源码         │ ~1.5 MB │
  ├─────────────────────────────┼─────────┤
  │ 构建后 minified              │ ~615 KB │
  ├─────────────────────────────┼─────────┤
  │ 构建后 gzip（实际网络传输）   │ ~134 KB │
  └─────────────────────────────┴─────────┘

  也就是说，引入 icons 映射后，用户实际下载的增量约为 134 KB（gzip），这在现代前端项目中是可以接受的。

  如果后续觉得 134 KB 还是偏大，可以考虑按需分块方案（import.meta.glob + defineAsyncComponent，只加载用到的图标），但实现复杂度会高不少。当前方案是简单和体积之间的合理折中。
```

很遗憾，我现在就觉得大，因为并不是所有用户都会开启 gzip，而且我不知道 AI 测试下的 gzip 压缩等级是多少，构建后的大小是 615KB，做到按需加载，那么大小会呈指数级下降（目前的构建后是 615 kb，且全量加载；改造后构建产物理论上不会小，但是是按需加载，可以理解为用的图标大小 5kb 就只加载 5kb）：

```bash
现在就完成按需分块方案（import.meta.glob + defineAsyncComponent）只加载用到的图标，而且将 import * as elIcons from '@element-plus/icons-vue' 的所有图标也注册为按需加载
```

失算了，花了 10 分钟，28k tokens，它还没搞定，这个可能有点复杂：

```bash
构建成功。1760 个 chunk，Lucide 图标不再打入主包。

Element Plus 图标由于在单个文件中，不能实现按需加载。
```

这一轮：
1. element plus 图标无法按需加载，那就还是按我自己之前的方式，动态的注册全部组件即可，官方文档之前看过，几行代码，后续手搓。
2. 1760 个 chunk 当然是不行的，让 cc 配置分包规则，减少 chunk 数量：`能否配置打包的分包规则，减少 chunk 数量，这太多了`

```bash
● 理解，1737 个 chunk 部署时确实太多。最务实的方案：Lucide 图标合并为单个 lazy chunk，不在主包中，用到才加载。

# ......

● 这样 Rollup 将所有 Lucide 图标合并为一个 lazy chunk，配合 HTTP/2 一次性下载 128 KB，比 1737 个小文件逐个请求高效得多。
```

单个 `chunk 128kb`，看起来是在加载任意图标时，又回到了全量加载，其实我的想法是实现比如按图标前缀分组 chunk 的方式，只是这种方案也很复杂且并不完美；好像是卡瓶颈了，只能再多和 AI 沟通沟通，让它再想想是否还有其他办法，先切换到 `plan mode`：`我需要真正的按需加载，且 chunk 不要那么多，是否还有其他办法？`

```bash
Leavening… (7m 5s · ↓ 27.2k tokens)

首次使用任意 Lucide 图标时一次性下载 128 KB (gzip)，之后所有图标即时可用。这比首屏就加载 615 KB 要好得多，也比部署 1760 个文件实际可用得多。
```

看得出来 AI 自我感觉良好，它告诉我现在的方案非常好，但是实际情况是，首屏一般就有使用 Lucide 图标，所以这 128KB 会在首屏完成加载，这显得懒加载功能很多余...

这时候可能会有小伙伴说换图标库，有的图标库从第三方 cdn 按需加载图标，但那不是我想要的。

#### 整理现状和所有方案

这里确定是遇到瓶颈了，让 AI 总结一下现在遇到的问题，和可行的方案：

```bash
● Import 只是一个图标名，不是工具函数。所以 @lucide/vue 没有内置按需加载机制。
     
  ---
  核心矛盾在于 Vite 8 / Rolldown 的代码分割机制：
     
  - 每个 import() → 1 个独立 chunk（这是代码分割的基础，无法改变）
  - import.meta.glob → 为每个匹配文件生成一个 import() → 1737 个 chunk
  - manualChunks 在 Rolldown 中无法可靠合并这些 chunk（我们已验证过）
  - 单次 import() 且内部静态引用多个模块 → 1 个 chunk（barrel 方案的核心原理）

  所以能达成"真正按需 + chunk 数量可控"的路径只有：

  ┌──────────────────────┬──────────────────────────────────────────────────────────────┬──────────────────────────┬──────────────────────────────┐
  │         方案         │                             原理                             │         chunk 数         │           维护负担           │
  ├──────────────────────┼──────────────────────────────────────────────────────────────┼──────────────────────────┼──────────────────────────────┤
  │ A. Barrel 生成脚本    │ 脚本生成 5 个静态 re-export 文件，各一个 import()              │ 5                        │ 加一个生成脚本 + postinstall │
  ├──────────────────────┼──────────────────────────────────────────────────────────────┼──────────────────────────┼──────────────────────────────┤
  │ B. Vite 插件虚拟模块  │ 同 A，但不落盘，插件 load 钩子动态生成 barrel 内容              │ 5                        │ 加一个轻量 Vite 插件         │
  ├──────────────────────┼──────────────────────────────────────────────────────────────┼──────────────────────────┼──────────────────────────────┤
  │ C. 接受单 chunk       │ import('@lucide/vue')                                        │ 1                        │ 零                           │
  ├──────────────────────┼──────────────────────────────────────────────────────────────┼──────────────────────────┼──────────────────────────────┤
  │ D. 手动 import       │ 放弃按名动态使用，哪里用哪里 import { X } from '@lucide/vue'    │ 0（tree-shaking 到主包） │ 零                           │
  └──────────────────────┴──────────────────────────────────────────────────────────────┴──────────────────────────┴──────────────────────────────┘

  B（Vite 插件） 最优雅——不需要落盘文件、不污染代码库、始终与 @lucide/vue 版本同步，只需一个 ~30 行的 Vite 插件。

  你倾向哪个？

✻ Cogitated for 38s
```

> 我现在想到了一个新的方案，在编译期间，确定使用了那些图标，然后写入到一个文件内记录，程序内再完成动态注册，是一种别样的按需加载。

但是我后续要做图标选择器组件，该组件需要所有 lucide icon 的 name 和将图标渲染出来，供用户选择，最终还是全量加载；所以目前感觉方案 B 是最佳的，现在让 AI 实现：

```bash
最终方案：Vite 插件 + 虚拟模块
     
  不需要生成脚本、不需要 postinstall、不需要落盘文件。全部逻辑在 vite.config.ts 中一个 ~40 行的自定义插件 lucideIconBatches()。

  原理
 
  菜单配置写 "icon": 
    "Smile"                                                                                                                                                
         ↓
  <Icon name="Smile" />
         ↓
  iconLoader.ts → getBatchName('Smile') → 'q-s'
         ↓
  import('virtual:lucide-icons/q-s')   ← 动态 import，产生独立 chunk
         ↓
  Vite 插件 load 钩子 → 读取 node_modules，返回 315 个图标 re-export
         ↓
  Rolldown 将 315 个静态 re-export 打包为 1 个 chunk（95 KB / 24 KB gzip）

  构建产物

  ┌───────────────────────┬──────────────┬────────┬────────────────────┐
  │         Chunk         │  min / gzip  │ 图标数 │      触发条件      │
  ├───────────────────────┼──────────────┼────────┼────────────────────┤
  │ a-c-*.js              │ 140 / 36 KB  │ 514    │ 首字母 a–c         │
  ├───────────────────────┼──────────────┼────────┼────────────────────┤
  │ d-l-*.js              │ 124 / 33 KB  │ 424    │ 首字母 d–l         │
  ├───────────────────────┼──────────────┼────────┼────────────────────┤
  │ m-p-*.js              │ 73 / 20 KB   │ 262    │ 首字母 m–p         │
  ├───────────────────────┼──────────────┼────────┼────────────────────┤
  │ q-s-*.js              │ 95 / 24 KB   │ 315    │ 首字母 q–s         │
  ├───────────────────────┼──────────────┼────────┼────────────────────┤
  │ t-z-*.js              │ 62 / 18 KB   │ 223    │ 首字母 t–z         │
  ├───────────────────────┼──────────────┼────────┼────────────────────┤
  │ createLucideIcon-*.js │ 1.3 / 0.8 KB │ —      │ 公共依赖（仅一次） │
  └───────────────────────┴──────────────┴────────┴────────────────────┘

  - 首屏：0 KB Lucide 代码
  - 使用 Smile：仅加载 q-s chunk（24 KB gzip）
  - 总 chunk 数：27（之前 1760）
  - 构建时间：1.08s

  菜单配置用法

  {
    "menus": [
      { "title": "仪表盘", "icon": "LayoutDashboard" },
      { "title": "用户管理", "icon": "Users" },
      { "title": "设置", "icon": "Settings" }
    ]
  }

  改配置即可换图标，无需改任何代码。

✻ Sautéed for 1m 47s
```

# 测试

#### 对比数据准备

目前尚未对最终方案的代码 `review`，先测试一下是否达成需求，由于需要和未实现按需加载之前做对比，我将本次工作区的全部改动使用 `git stash` 暂存起来。

未加载任何图标前，浏览器禁用缓存，总是使用同一 vue 页面测试：

开发环境下：47 个请求，传输 4.7MB，4.7MB 项资源。
构建产物：`assets` 内共 17 个项目，1.55 MB、14 个请求，传输 1.6MB，1.6MB 项资源，上 100kb 的文件有以下几个：

http://localhost:8000/assets/index-Cww9--Nk.js `1078kb`
http://localhost:8000/assets/vue.runtime.esm-bundler-58Gmz7_F.js `111kb`
http://localhost:8000/assets/style-C1fZPW92.css `370kb`

#### 方案数据

开发环境下：475 个请求，传输 5.4MB，5.3MB 项资源。

我勒个豆，一个 icon 一个请求？不测了，先让 AI 改：`将同一 Chunk 的 icon 放到一个文件可以吗？目前单 chunk 的请求数达到了 475 个！`

```bash
● 完美。每个 chunk = 1 个 HTTP 请求。
     
  a-c  140KB  0 个 import()  只有 1 个静态 import(共享 createLucideIcon)
  d-l  124KB  0 个 import()
  m-p   73KB  0 个 import()
  q-s   95KB  0 个 import()
  t-z   62KB  0 个 import()

  根因与修复

  export { default as X } from '...'（re-export）→ Rolldown 将其视为透传绑定，不内联模块，导致每个图标一个子请求。

  import X from '...' + export const Name = X（显式绑定）→ Rolldown 视为本地依赖，必须内联到同一个 chunk 文件。

  修改的就是 Vite 插件 load 函数中生成的代码格式，从一个 export from 变为 import + export const。其余代码（iconLoader.ts、Icon 组件）无需改动。

✻ Crunched for 1m 24s
```

完美个 der，还是 475 个请求，我开关 Vite 开发服务多次，确定数据没错。此时考虑到可能是因为开发环境的原因？但是也得改，因为大批量的文件加载，会让正常的调试和 log 变得很麻烦，这里提醒一下 AI，我在开发环境。

```bash
● 最终方案总结

  开发模式 (pnpm dev)：
    import('@lucide/vue') → 1 个请求，加载全部图标
    避免虚拟模块 475 个子请求

  生产模式 (pnpm build)：
    import('virtual:lucide-icons/a-c') → 1 chunk，1 请求，140KB / 38KB gzip
    import('virtual:lucide-icons/d-l') → 1 chunk，1 请求，127KB / 35KB gzip
    import('virtual:lucide-icons/m-p') → 1 chunk，1 请求，75KB / 21KB gzip
    import('virtual:lucide-icons/q-s') → 1 chunk，1 请求，97KB / 26KB gzip
    import('virtual:lucide-icons/t-z') → 1 chunk，1 请求，63KB / 19KB gzip
    createLucideIcon (共享)      → 1 chunk，1 请求，1.3KB

  首屏 HTML：0 KB Lucide 代码
  使用 <Icon name="Smile" />：触发 q-s chunk (26KB gzip) + createLucideIcon (0.8KB)

  import.meta.env.DEV 是编译时常量，Vite 在构建时将无效分支完全删除，零运行时开销。
```

#### 方案数据再次统计

开发环境下：51 个请求，传输 5.5MB，5.5MB 项资源。
构建产物：`assets` 内共 24 个项目，2.17 MB、17 个请求，传输 1.9MB，1.9MB 项资源，上 100kb 的文件多了以下一个：

http://localhost:8000/assets/a-c-CdQL3xxB.js `144kb`

请求数合理，请求大小合理，但是又发现了一个大问题，图标特喵根本没显示，空白的。

继续让 AI 改，幸好只是个图标大小写转换的小问题，修改好了之后再次测试：

逐一使用 a d m q t 开头的图标，确定以下文件是按需加载的，文件大小分布也较为合理，AI还是行的：

http://localhost:8000/assets/a-c-Byxc3NbE.js `144kb`
http://localhost:8000/assets/d-l-D6J0NQGY.js `127kb`
http://localhost:8000/assets/m-p-Cti2KRmr.js `75.2kb`
http://localhost:8000/assets/q-s-CGvOdbjT.js `97.3kb`
http://localhost:8000/assets/t-z-d79zgzJ8.js `63.5kb`

好了，宣布成功，后面再慢慢 `review` 吧，Icon 组件核心难点已经完美解决。