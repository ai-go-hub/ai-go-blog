# 从 PHP 到 AI + Golang，程序员自救转型手记（五十七）：agInput 统一入口，动态表单预备、AI时代对组件封装的理解

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “附件管理、增加根据文件后缀生成 SVG 文件图标的接口”，本期将完成：agInput 统一入口，动态表单预备

# agInput 统一入口，动态表单预备

下一个功能是系统配置，配置项支持后台动态新增，所以需要先考虑动态表单的事；

> 为什么输入组件要叫做 `agInput`？因为 `input` 不能用，被 `html` 占了，所以加了个前缀 `ag`，它是 `ai-go` 的简写。

我们目前已经封装了多个必要的输入组件：

1. `agUpload` 图、多图、文件、多文件上传
2. `areaSelect` 省份地区三联动选择
3. `array` 数组
4. `iconSelect` 图标选择器
5. `remoteSelect` 远程下拉

都是常用 + 组件库没有的（或组件库无法实现的），对于组件库已有的，比如 `el-input`、`el-radio`、`el-date-picker` 等，我们全部：**不做二次封装，AI 时代多余的封装造成的理解成本全部都是在浪费 token 和人力**，需要使用它们时，直接：

```vue
<template>
    <el-form-item label="昵称" prop="nickname">
        <el-input type="string" v-model="formItems.nickname"></el-input>
    </el-form-item>
</template>
```

甚至，对于我们已经封装的组件，也推荐直接导入并使用（不玩儿任何花活）：

```vue
<template>
    <el-form-item label="分组" prop="group">
        <RemoteSelect
            field="name"
            v-model="formItems.group"
            remote-url="/admin/auth/group/list"
        />
    </el-form-item>
</template>

<script setup lang="ts">
import RemoteSelect from '/@/components/agInput/components/remoteSelect.vue'
</script>
```

人类好理解，AI 也非常好理解，但是，有时还要额外考虑到 `动态表单` 的场景，所以我们现在来实现一个 `统一入口`，比如：

```vue
<!-- 字符串 -->
<AgInput type="string" v-model="..." />

<!-- 数字 -->
<AgInput type="number" v-model="..." />

<!-- 远程下拉 -->
<AgInput type="remoteSelect" v-model="..." />
```

即：传递一个`type`，就可以渲染出对应的输入组件（做动态表单及其方便），`type` 支持 `string、number` 这种组件库提供的，也支持 `RemoteSelect` 这种我们自己封装的。

> 本篇之所以开头写了很多不做输入组件封装的理念，就是为了防止 `AgInput` 被滥用，它虽然非常简洁，但不易理解，不易扩展（属性、插槽填写不便）个人只建议将它使用在动态表单场景。

最终，我们封装出来的动态表单用的 `agInput` 支持了以下输入组件类型，共计达 `25` 种：

```ts
/**
 * 支持的输入框类型
 */
export const inputTypes = [
    'text',
    'string',
    'password',
    'number',
    'radio',
    'checkbox',
    'switch',
    'textarea',
    'array',
    'datetime',
    'year',
    'date',
    'time',
    'select',
    'selects',
    'areaSelect',
    'iconSelect',
    'remoteSelect',
    'remoteSelects',
    'editor',
    'image',
    'images',
    'file',
    'files',
    'color',
]
```

本篇的 `agInput` 组件，主要是靠人工整理的，因为可以啃 `BuildAdmin` 老本，总体还算简单，主要是让 AI 整理了一个：各种输入组件 `最常用的属性` 列表，定义为 `ts` 类型，以便编辑器的智能提示、自动完成等能够很好的工作。
