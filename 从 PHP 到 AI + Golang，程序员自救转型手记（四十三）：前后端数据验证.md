# 从 PHP 到 AI + Golang，程序员自救转型手记（四十三）：前后端数据验证

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “前端表格组件”，本期将完成：前后端数据验证

# 前后端数据验证

## 服务端数据验证

一共有三种数据验证方案：

### 方案一

模型层写 `binding` tag，然后在控制器层使用 `gin` 的 `ShouldBindJSON`，在绑定 `JSON` 数据同时完成效验（基控制器默认的 `Create`、`Update` 方法都已经调过 `ShouldBindJSON` 了，模型定义写好验证规则即可）示例如下：

```go
Username string `gorm:"comment:用户名;type:varchar(64);not null" binding:"required"`
```

`required` 规则意为 `必填`，完整可用规则有很多，可以参考：[go-playground/validator](https://github.com/go-playground/validator) | [gin 模型绑定和验证章节](https://gin-gonic.com/zh-cn/docs/binding/binding-and-validation/)

> go-playground/validator 包已随 gin 框架内置，可直接使用。

### 方案二

模型定义写 `validate` tag，如：

```go
Username string `gorm:"comment:用户名;type:varchar(64)" validate:"required"`
```

可用规则和 `方案一` 的 `binding` 一致，`validate` 的写法是 [go-playground/validator](https://github.com/go-playground/validator) 原生提供的。

然后，服务层或控制器层重写 `Create`、`Update` 等方法手动完成效验，示例如下：

```go
type User struct {
    FirstName string `validate:"required"`
    Age       uint8  `validate:"gte=0,lte=130"`
    Email     string `validate:"required,email"`
}

validate := validator.New()

user := &User{
    Age:   135,
    Email: "Badger.Smith@gmail.com",
}

err := validate.Struct(user)
```

### 方案三

增加 `DTO`，并定义以上的 `binding / validate` tag，然后重写控制器/服务，`AdminUpdateRequest` 就是一个自定义 `DTO`，不理解的可以全局搜索参考。

### 何时在控制器效验，何时在服务层效验？

**服务层**：涉及数据库、父子关系、唯一性、字段联动的校验。

**控制器层**：只校验字段自身独立规则，如必填、长度、有效值，不依赖数据库、不跨字段判断。

**例如 AdminRule 表**：`title、name、type` 的必填就应该放在控制器层，`name` 的重复、`pid` 不能是自身，就应该放在服务层；注意仓储层不做任何效验，仅使用 `not null`、唯一索引等兜底即可。

## 前端数据验证

前端主要是表单数据的验证，我们准备了一个验证工具包 `src\utils\validate.ts`，里边定义了各种验证规则，另外主要是导出了一个验证规则构建器：`buildValidatorRule` 函数，它可以非常方便的生成 `el-form` 表单的常用验证规则，示例如下：

> element plus 的表单验证，是基于 [async-validator](https://github.com/yiminghe/async-validator)

```vue
<template>
    <div>
        <el-form ref="formRef" :model="formItems" :rules="rules"></el-form>
    </div>
</template>

<script setup lang="ts">
import { regularPassword, buildValidatorRule } from '/@/utils/validate'
import type { FormItemRule } from 'element-plus'

const rules: Partial<Record<string, FormItemRule[]>> = reactive({
    username: [buildValidatorRule({ name: 'required', title: '用户名' }), buildValidatorRule({ name: 'account' })],
    nickname: [buildValidatorRule({ name: 'required', title: '昵称' })],
    email: [buildValidatorRule({ name: 'email', message: '邮箱错误' })],
    mobile: [buildValidatorRule({ name: 'mobile', message: '手机号错误' })],
    password: [
        {
            validator: (rule: any, val: string, callback: Function) => {
                if (props.manager.form.operate == 'create') {
                    if (!val) {
                        return callback(new Error('密码必填'))
                    }
                } else {
                    if (!val) {
                        return callback()
                    }
                }
                if (!regularPassword(val)) {
                    return callback(new Error('密码需要 6-32 位，禁止使用特殊符号'))
                }
                return callback()
            },
            trigger: 'blur',
        },
    ],
})
</script>
```