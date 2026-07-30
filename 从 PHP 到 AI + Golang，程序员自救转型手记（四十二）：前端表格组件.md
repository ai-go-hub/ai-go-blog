# 从 PHP 到 AI + Golang，程序员自救转型手记（四十二）：前端表格组件

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “增加管理员账号管理接口”，本期将完成：前端表格组件

# 前端表格组件

这是基类改造之外的另外一个比较麻烦的工作，基类 + 表格组件（有 [BuildAdmin](https://gitee.com/wonderful-code/buildadmin) 已有代码参考），作者在 AI 的协助下，还是花了整整两个礼拜的时间。

不同于 `input`，`html` 本身没有 `table` 这个标签，所以表格组件名称可以直接叫做 `table`，不会有任何冲突。

接下来，我们将完成：表头各类操作按钮（刷新、添加、编辑选中行，删除选中行，折叠与展开行等）、表行级操作按钮（编辑、删除、拖拽排序等）、表格数据获取、翻页、页大小改变、默认排序、排序切换、双击表格、快速搜索、列显示切换、选中行记录、表格内单独验权、单元格值渲染前格式化函数、各类动作的钩子与拦截函数等等很多，而比较特别的主要有以下几点：

### TableAPI 请求类

一个 `class`，构造函数接受接口 URL，如 `/admin/auth/admin/`，`class` 内将自动拼接好 `get、list、create、delete` 的完整 URL，并提供 `get、list、create、delete` 等请求函数，调用就能向对应接口发送请求了。

`const api = new TableManagerAPI('/admin/auth/admin/')`

```ts
api.get(pk)
api.list(filter)
// ......
```

### 表格公共搜索

首先是需要根据表格列定义，动态的生成公共搜索表单，不需要开发者手动写公共搜索，只是需要配置某列是否开启公共搜索，搜索操作符号等即可。

其次，需要实现根据 `URL query` 初始化表格公共搜索数据，比如 `?id=1`，那么公共搜索组件就可以匹配是否有一个名为 `id` 的列，有的话直接预填其值为 `1`，再读取列表数据。

最后，开发者需要可以 `随时、方便` 的对公共搜索数据进行改动（覆盖或合并），所以我们额外提供了 `setComSearchData`、`getComSearchData`、`setFilterWheres`、`getQuickSearchData` 等筛选相关的公开方法。

### 表格表单

是的，表格自带 `新增、编辑` 表单，默认是弹窗式的，双击某行即可打开编辑，而且为了方便扩展，每个后台管理功能的表单定义在一个单独的 `vue` 文件中，里边就是一个非常普通的 `el-form + el-form-item` 的组合，主要是接受一个 `formItems` 绑定值和一个表格管家实例 `manager`，提交表单时，走 `manager.submitForm` 方法即可，组件内也可以通过管家实例 `manager` 访问所有上下文数据，比如列定义，新增表单默认值等等。

表单组件的位置大概是这样的：

```bash
auth/admin/              # 权限管理 > 管理员账号管理目录
    ├───────── index.vue # 导入并使用 `表头、表格、表单组件` 的文件，也是路由组件文件
    └───────── form.vue  # 表单组件实现
```

> 我们做后台管理系统很多年了，以前也遇到过一个小伙伴，说这样的表格设计有个大缺陷，最完美的应该是把 form.vue 和并到 index.vue，但这事我不以为然，vue项目，我不至于把 新增和编辑表单 拆开，但是表单是必需抽出来的，这样二开才方便，看起来也更加简洁，index.vue 只保留各组件的引入和表格列的定义、配置等即可。

### 单元格渲染器

我们准备了一个文件夹 `table\cellRenderer` 里边放了 `color、icon、datetime、images、tags` 等组件，**其中一个组件代表一个单元格渲染器**。

使用渲染器时，列定义写 `render` 字段即可：

```ts
{
    label: '创建时间',
    prop: 'createdAt',
    align: 'center',
    render: 'datetime',
    width: 160,
}
```

`render: '渲染器组件名'`，直接填组件名，无需额外引入组件等，内部会自动使用该组件进行渲染，并传递表格相关的大量上下文，比如列定义、单元格值等。

而且，我们内置了一个简洁的 Vite 插件，用于生成 `单元格渲染器的 TS 类型定义` 在开发者执行 `npm run dev` 命令时，会自动执行该生成器，读取 `cellRenderer` 文件列表，让开发者得到完整的类型支持，比如更友好的编辑器提示、类型检查等。

单元格渲染相关，还支持渲染前格式化、渲染函数、指定 `slot` 名称渲染、模板插槽渲染 等，非常完善，单元格渲染方面，没有满足不了的需求。

### 继承 el-table 特性的 table 组件

我们的表格是基于 `el-table` 进行开发的，所以我们需要尽量完美的继承 `el-table` 这个组件已有特性。

1. 内部使用 `<el-table v-bind="$attrs">`，绑定外层的所有属性到 `el-table` 上
2. 外层 `table` 组件，继承内部 `el-table` 组件的所有 `props` 类型提示，同时继承所有 `@cell-click` 这类事件名的类型提示
3. 继承 `el-table` 所有插槽，并且自定义两个插槽 `columnPrepend` 和 `columnAppend` 以方便定义一些对位置要求较高的列
4. 对外提供一个 `getElTableRef` 方法，以便获取内部 ref
5. ......

### 对比 BuildAdmin 表格组件

#### 表格管家类命名

在那边，和表格配套使用的是 `src\utils\baTable.ts`。

这一次由于可以从零开始，我们终于将其正式的命名为 `src\hooks\useTableManager.ts`，首先是放在了 `hooks` 目录，表示它和组件强绑定，其次就是名称直接带了 `Manager` 单词，表示它是 `表格管家`，管家这个概念非常重要，因为涉及表格的一切，都可以找它，比如：

1. 你可以找它拿数据，比如 `表格行数据、当前被编辑行的数据、公共搜索表单数据、快速搜索关键词、当前表单操作标识、加载状态、页码、每页显示数` 等，几乎与表格相关的所有数据都能从这里拿到
2. 你可以找它更新数据，你能拿到的数据自然也能直接修改，这些数据通常都具备响应性
3. 你可以找它做操作，比如 `刷新表格、发起公共/快速搜索、调整排序、分页、打开编辑表单` 等
4. 监控与拦截操作，几乎所有事件均可拦截或监控，比如 `获取表格数据前后、获取编辑行数据前后、双击单元格前后、打开/提交表单、刷新、删除` 等
5. 高度可定制化，您可以随时重写它的方法，添加自定义属性（方便数据随整个类在上下文中流通）

> 之所以没有放在 components/table 目录以内，首先是 table 相关本身就没法完全聚在一起，比如 API 请求函数，还是得放在 api 目录，“聚在一起”本身已经被破坏了。
> 其次是因为项目 hooks 目录太单薄了，目前还有个 useDark 而已，但是 hooks 目录又不能没有，以往是将 useDark 直接放在 utils 目录，而如果有 useDark + useTableManager，那么 hooks 的存在就非常合理了，开发者可以看的更加清晰，也方便扩展。

#### 管家初始化

以往管家类参数必需按顺序传递（虽然也可以初始化时不传递，然后后续立即单独赋值），签名是：`constructor(api: baTableApi, table: BaTable, form: BaTableForm = {}, before: BaTableBefore = {}, after: BaTableAfter = {})`；

而现在，管家直接接受一个 `object`，你想传递谁、先传递谁，都可以随心所欲了。

```ts
export interface UseTableManagerOptions {
    api: TableManagerAPI
    table?: TableInterface
    form?: FormInterface
    before?: TableManagerBefore
    after?: TableManagerAfter
    auth?: (node: string) => boolean
}

export function useTableManager(opts: UseTableManagerOptions): TableManagerInstance
```

#### 管家实例传递

**以往：**

通过 `provide` 传递，然后子组件使用 `inject` 接受实例：

```ts
provide('baTable', baTable)

// ------- 接受 -------

import type baTableClass from '/@/utils/baTable'

const baTable = inject('baTable') as baTableClass
```

这是非常好用的，但是小部分伙伴无法理解，是的，他们不能理解 `provide` 和 `inject`，理解到表格管家这个概念，可能已经用完了脑容量（甚至表格管家这个名字，也是为了理解而取的，以前我总是向一些开发者解释 `它是一个普通的 class，里边什么都有` 😂）。

以往的版本还有一个最大的缺点，需要一个页面放两个表格时，必需将表格单独抽到 `vue` 文件里边去，一个 `vue` 文件放一个表格，以便 `provide` 和 `inject`。

**新的：**

直接使用组件 `props` 传递管家实例：

```vue
<template>
    <div class="default-main">
        <TableHeader
            :manager="tableManager"
            v-model:com-search="tableManager.comSearch"
            :buttons="['refresh', 'add', 'edit', 'delete', 'comSearch', 'quickSearch', 'columnDisplay']"
        />
        <Table :manager="tableManager" />

        <PopupForm :manager="tableManager" v-model:form-items="tableManager.form.items!" />
    </div>
</template>

<script setup lang="ts">
const tableManager = useTableManager()
</script>
```

> 比较有意思的是，以往的版本，最开始其实也是使用 `props` 传递实例，当时传递为 `table`，组件内使用 `table.table、table.table.column` 等访问实例属性，放弃此方案的主要原因就是 `table.table` 太难看了，而现在有了 `表格管家` 的概念，可以使用 `manager.table`，非常舒服易理解。

#### 禁止双击编辑的列

以往是 `dblClickNotEditColumn: [undefined, 'status']`，对于表格的第一列 `多选框`，总是需要定义一个 `undefined`，现在不用了，自动禁用该列的双击编辑。

#### 其他

总体其实优化了很多细节，其他就不一一列出了，总而言之，我们从 BuildAdmin 继承并改善了 table 组件，以我目前的眼光来看，它是的设计是完美的。