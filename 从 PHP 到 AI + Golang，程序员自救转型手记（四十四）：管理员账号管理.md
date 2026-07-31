# 从 PHP 到 AI + Golang，程序员自救转型手记（四十四）：管理员账号管理

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “前后端数据验证”，本期将完成：管理员账号管理

# 管理员账号管理

管理员账号管理接口之前已经创建好了，同时受益于基类的极大通用性，准备好的现成的接口有：

1. `/admin/auth/admin/list`
2. `/admin/auth/admin/create`
3. `/admin/auth/admin/delete`
4. `/admin/auth/admin/get/pk`
5. `/admin/auth/admin/update/pk`

回到前端，我们可以利用 `src\api\table.ts` 中的 `TableManagerAPI` 表格API请求函数生成类，一次性生成好请求以上接口的 `请求函数`，如下：

`const api = new TableManagerAPI('/admin/auth/admin/')`，对应：

```ts
api.get(pk)
api.list(filter)
// ......
```

但是生成好的 `api` 实例，不是我们自己使用的，而是直接传递给 `表格管家` 即可，而初始化表格管家时，最核心的是写好 `table.column`（列定义数据），代码如下：

```ts
import PopupForm from './popupForm.vue'
import { TableManagerAPI } from '/@/api/table'
import { useTableManager } from '/@/hooks/useTableManager'

const api = new TableManagerAPI('/admin/auth/admin/')

const tableManager = useTableManager({
    api,
    table: {
        column: [
            { type: 'selection', align: 'center', operator: false },
            { label: 'ID', prop: 'id', align: 'center', operator: 'eq', width: 70 },
            { label: t('auth.admin.username'), prop: 'username', align: 'center', operator: 'eq', quickSearch: true },
            { label: t('auth.admin.nickname'), prop: 'nickname', align: 'center', operator: 'LIKE', quickSearch: true },
            { label: t('auth.admin.avatar'), prop: 'avatar', align: 'center', render: 'image', operator: false },
            { label: t('common.email'), prop: 'email', width: 180, align: 'center', operator: 'LIKE' },
            { label: t('common.mobile'), prop: 'mobile', align: 'center', operator: 'LIKE' },
            {
                label: t('common.operate'),
                align: 'center',
                width: '100',
                render: 'buttons',
                buttons: optButtons,
                operator: false,
            },
        ],
        filter: {
            sort: 'created_at',
            order: 'desc',
        },
        dblClickNotEditColumn: ['status'],
    }
})

tableManager.getData()
```

为保证可阅读性，以上代码略有简化，接下来，将 `表格管家` 实例，传递给 `table` 组件即可，如下：

```vue
<template>
    <div>
        <TableHeader
            :manager="tableManager"
            v-model:com-search="tableManager.comSearch"
            :buttons="['refresh', 'add', 'edit', 'delete', 'comSearch', 'quickSearch', 'columnDisplay']"
        />
        <Table :manager="tableManager" />

        <PopupForm :manager="tableManager" v-model:form-items="tableManager.form.items!" />
    </div>
</template>
```

这三个组件所需要的 `props`，基本都可以通过 `表格管家` 实例取得。

此时，如果忽略掉 `./popupForm.vue` 的代码，管理员账号管理的功能，最起码是基础的 CRUD 的功能，已经完成了，样子大概是这样的：

![](/images/44/1.png)

是的，增删改查齐备，之所以这么简单，全部是受益前几章的铺垫；接下来就是表格的 `新增 / 编辑` 表单了，但这更加简单，你基本上可以理解为，使用 `el-form + el-form-item` 写一个普普通通的表单就行了，绑定值与提交方法等，全部都提前准备好了，且如果需要表单验证，也可以使用 `buildValidatorRule` 快速生成，示例如下：

![](/images/44/2.png)

```vue
<template>
    <!-- 对话框表单 -->
    <el-dialog
        class="ag-operate-dialog"
        :close-on-click-modal="false"
        :model-value="['create', 'update'].includes(manager.form.operate!)"
        @close="manager.toggleForm"
        :destroy-on-close="true"
        :draggable="true"
    >
        <template #header>
            <div class="title">
                {{ manager.form.operate == 'create' ? t('common.add') : t('common.edit') }}
            </div>
        </template>
        <el-scrollbar v-loading="manager.form.loading" class="ag-table-form-scrollbar">
            <div
                class="ag-operate-form"
                :class="'ag-' + manager.form.operate + '-form'"
                :style="config.layout.shrink ? '' : 'width: calc(100% - ' + manager.form.labelWidth! / 2 + 'px)'"
            >
                <el-form
                    ref="formRef"
                    @keyup.enter="manager.submitForm(formRef)"
                    :model="formItems"
                    :label-position="config.layout.shrink ? 'top' : 'right'"
                    :label-width="manager.form.labelWidth + 'px'"
                    :rules="rules"
                    v-if="!manager.form.loading"
                >
                    <el-form-item :label="t('auth.admin.username')" prop="username">
                        <el-input
                            type="string"
                            v-model="formItems.username"
                            :placeholder="t('common.pleaseEnter', { field: t('auth.admin.username') })"
                        ></el-input>
                    </el-form-item>

                    <el-form-item :label="t('auth.admin.avatar')">
                        <AgUpload type="image" v-model="formItems.avatar" />
                    </el-form-item>

                    <el-form-item prop="bio" :label="t('auth.admin.bio')">
                        <el-input
                            @keyup.enter.stop=""
                            @keyup.ctrl.enter="manager.submitForm(formRef)"
                            v-model="formItems.bio"
                            type="textarea"
                            :placeholder="t('common.pleaseEnter', { field: t('auth.admin.bio') })"
                        ></el-input>
                    </el-form-item>

                    <!-- 部分略 -->

                    <el-form-item :label="t('common.status')">
                        <el-radio-group v-model="formItems.status">
                            <el-radio value="enable" :border="true">{{ t('common.enable') }}</el-radio>
                            <el-radio value="disable" :border="true">{{ t('common.disable') }}</el-radio>
                        </el-radio-group>
                    </el-form-item>
                </el-form>
            </div>
        </el-scrollbar>
        <template #footer>
            <div :style="'width: calc(100% - ' + manager.form.labelWidth! / 1.8 + 'px)'">
                <el-button @click="manager.toggleForm()">{{ t('common.cancel') }}</el-button>
                <el-button :loading="manager.form.submitLoading" @click="manager.submitForm(formRef)" type="primary">
                    {{ manager.form.operatePKs && manager.form.operatePKs.length > 1 ? t('common.saveAndContinue') : t('common.save') }}
                </el-button>
            </div>
        </template>
    </el-dialog>
</template>

<script setup lang="ts">
import { reactive, watch, useTemplateRef } from 'vue'
import { useI18n } from 'vue-i18n'
import { regularPassword, buildValidatorRule } from '/@/utils/validate'
import type { FormItemRule } from 'element-plus'
import { useConfig } from '/@/stores/config'
import AgUpload from '/@/components/agInput/components/agUpload.vue'

interface Props {
    manager: TableManagerInstance
}

const props = defineProps<Props>()
const formItems = defineModel<AnyObj>('formItems', { required: true })

const config = useConfig()
const formRef = useTemplateRef('formRef')

const { t } = useI18n()

const rules: Partial<Record<string, FormItemRule[]>> = reactive({
    username: [buildValidatorRule({ name: 'required', title: t('auth.admin.username') }), buildValidatorRule({ name: 'account' })],
})

watch(
    () => props.manager.form.operate,
    (newVal) => {
        // 创建密码字段必填，编辑非必填
        rules.password![0].required = newVal == 'create'
    }
)
</script>

<style scoped lang="scss"></style>
```