# 从 PHP 到 AI + Golang，程序员自救转型手记（四十八）：菜单规则管理实现

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “列表通用排序接口实现（增量重排法）”，本期将完成：菜单规则管理实现

# 菜单规则管理实现

我们之前已经写好了 `AdminRule` 模型，这里提供向 AI 详细的提供模型位置和新文件位置，并提供参考代码：基于 `@internal/model/admin.go` 的 `AdminRule` 模型在 `@internal/handler/admin/auth/` 建立 `AuthAdminRuleHandler` 控制器，并建立对应的仓储、服务等，可参考已有的 `AuthAdminHandler` 及其仓储、服务。

```bash
已完成 AdminRule 的 CRUD 骨架搭建，新增 4 个文件并通过 go build、go vet、go test 校验。

新增文件：
internal/repository/admin/rule.go — AdminRuleRepository
internal/service/admin/auth/rule.go — AuthAdminRuleService
internal/handler/admin/auth/rule.go — AuthAdminRuleHandler
internal/router/admin/auth/rule.go — 路由注册

# ......
```

先简单检查了一下还行，好家伙，考虑未来不做可视化 `CRUD` 了😂。

### 数据效验

目前 AI 写的效验，全部是在服务层，新增了一个 `validateRule` 函数，里边写了大量 `if`，用来检查 `name` 不能为空、`pid` 不能为自身等；

这当然不太合理，`gin` 的 `ShouldBindJSON` 方法本身就支持数据绑定同时完成效验，而且我们在 `基控制器` 已经调用过该方法了，所以 `必填、格式正确性` 等验证直接在模型上加 `binding` tag 即可。

也可以写 `validate` tag，它是专门做验证的包 `https://github.com/go-playground/validator` 的语法，该依赖已随 `gin` 框架内置，可直接使用。

还有一些涉及 `数据库、父子关系、唯一性、业务联动` 的效验，必须抽到服务层，这里有两个选择，使用 `validate` tag，然后在服务层完成效验，或直接代码里边 `if`，这里选择后者，因为这次验证不算多，且对验证消息的要求比较高，如下：

```go
// 主要就是因为这个验证，选择统一使用 if，这里的验证消息定制化较高
if entity.Type == "menu" && (entity.Path == "" || entity.Component == "") {
    return errors.New("规则类型为菜单时，菜单路由路径和菜单组件路径不能为空")
}

// 名称唯一校验（排除自身）
existing, err := s.repo.FindByName(c, entity.Name)
if err != nil && !errors.Is(err, gorm.ErrRecordNotFound) {
    return err
}
if existing != nil && strconv.FormatUint(uint64(existing.ID), 10) != pk {
    return errors.New("规则名称已存在")
}

// 路由路径的唯一性验证
// ......

// 不能将自身设为自己的上级 + 上级存在性检查
// ......
```

### 方法位置错误

`findByName`（根据规则名称查询）和 `findByPath`（根据菜单路由路径查询）方法，AI 给放在了服务层，可能是因为目前这两个方法仅用于 数据效验 时查数据。

这里让它移动至仓储层，一是因为职能规划，二是这两个方法，未来其他地方需要使用的可能性很大，仓储里边有它们非常合理。

完成移动只需要一句话，然后：

```bash
已移至 internal/repository/admin/rule.go：

FindByName 和 FindByPath 已移至 internal/repository/admin/rule.go。
两个仓储方法均使用 GORM Generics API。

服务层改为调用：
s.repo.FindByName(c, entity.Name)
s.repo.FindByPath(c, entity.Path)

删除服务层原有的 findByName、findByPath 私有方法。
服务层仍负责唯一性判断和业务错误消息。
已格式化并通过仓储层、服务层相关包测试。
```

### 菜单规则管理定制功能

菜单规则作为权限系统的基础，此功能当然不是一个 `CRUD` 就能搞定的，接下来基于已有 `CRUD` 进行一系列定制

#### 控制器层

**自定义扩展数据和列表数据适配器**：

1. 菜单规则管理的菜单列表，需要组装为可以展开折叠的树状表格数据，使用 `tree.Build` 即可
2. 选择器（如远程下拉）读取菜单列表时，需要组装为另外一种树状列表（在 `title` 字段上渲染树状分支符号），使用 `tree.Render` 即可

**只读取当前登录管理员拥有权限的菜单规则**：

- 使用 `handler.WithExtension`（自定义扩展数据）传递当前登录管理员的数据到服务层供后续使用

```go
// NewAuthAdminRuleHandler 创建菜单和权限规则管理控制器实例
func NewAuthAdminRuleHandler(svc *svcAuth.AuthAdminRuleService) *AuthAdminRuleHandler {
	return &AuthAdminRuleHandler{
		Handler: handler.NewHandler(svc,
			handler.WithAdapter(handler.Adapter{
				// 定义控制器层的数据适配器（就不需要重写控制器层的 List 方法了）
				List: func(data any, opts service.Options) (any, error) {
					rules, ok := data.([]model.AdminRule)
					if !ok {
						return data, nil
					}

					ruleData := make([]map[string]any, len(rules))
					for i := range rules {
						ruleData[i] = rules[i].ToMap()
					}

					if opts.Selector {
						return tree.Render(ruleData, "id", "pid", "title"), nil
					} else {
						return tree.Build(ruleData, "id", "pid", "children"), nil
					}
				},
			}),
			handler.WithExtension(func(c *gin.Context) any {
				return &svcAuth.AuthAdminRuleExtension{
					AdminSession: middleware.GetAdmin(c),
				}
			}),
		),
		svc: svc,
	}
}
```

#### 服务层

**List 方法**：

读取自定义扩展中的管理员数据，并使用 `Permission` 包读取当前登录管理员拥有权限的菜单规则。

```go
// List 覆写通用查询全部记录方法: 根据管理员权限过滤
func (s *AuthAdminRuleService) List(c *gin.Context, opts service.Options) ([]model.AdminRule, error) {
	// 从控制器传来的 `管理员信息` 扩展数据
	extension, ok := opts.Extension.(*AuthAdminRuleExtension)
	if !ok || extension.AdminSession == nil {
		return nil, errors.New("参数错误，缺少 AdminSession 扩展数据")
	}

	perm := permission.New()

    // 是否超级管理员
	super, err := perm.IsSuperAdmin(c.Request.Context(), extension.AdminSession.ID)
	if err != nil {
		return nil, err
	}

	// 树状表格无翻页，设定无限 limit
	opts.Limit = 999999

	// 来自选择器则只查 dir、menu，不查 node
	if opts.Selector {
		opts.Wheres = append(opts.Wheres, service.WhereGroup{
			Wheres: []service.Where{{
				Field:    "type",
				Operator: "IN",
				Value:    []string{"dir", "menu"},
			}},
		})
	}

	// 非超管，读取当前管理员拥有的权限规则 IDs
	if !super {
		ruleIDs, err := perm.GetRuleIds(c.Request.Context(), extension.AdminSession.ID, nil)
		if err != nil {
			return nil, err
		}

		if len(ruleIDs) == 0 {
			return nil, nil
		}

		// 添加 IN IDs 的 where 以确保只读取到当前管理员拥有的权限规则
		opts.Wheres = append(opts.Wheres, service.WhereGroup{
			Wheres: []service.Where{{
				Field:    "id",
				Operator: "IN",
				Value:    ruleIDs,
			}},
		})
	}

	rules, err := s.repo.List(c, s.BuildRepoOpts(opts))
	return rules, err
}
```

**Count 方法**：做类似以上的改造，只读取拥有权限的菜单规则的数量

### 前端

受益于我们的 `table` 组件和 `useTableManager.ts` 表格管家，按部就班的写好列定义及表单即可自动完成对服务端数状表格的兼容，没啥特殊的，不再赘述。