# 从 PHP 到 AI + Golang，程序员自救转型手记（四十一）：增加管理员账号管理接口

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们进行了 “基类改造”，本期将完成：增加管理员账号管理接口

# 增加管理员账号管理接口

我们之前已经准备好了基控制器、仓储和服务，同时 `Admin` 模型也已经定义，此时可以直接让 AI 基于这些东西，直接完成接口的创建。

这里提供向 AI 详细的模型位置和新文件位置：创建 `/admin/auth/admin` 管理员账号管理 CRUD 接口，基于 `@internal/model/admin.go` 的 `Admin` 模型在 `@internal/handler/admin/auth/` 建立 `AuthAdminHandler` 控制器，并建立对应的服务，仓储可以直接使用 `internal\repository\admin\admin.go`（做管理员登录时建立的）。

## 账号管理接口零定制版

受益于我们创建的基类非常完善，创建这样一个 CRUD 接口，如果不做任何定制的话，代码量非常少：

**控制器：internal\handler\admin\auth\admin.go**

```go
package auth

import (
	"github.com/ai-go-hub/ai-go-admin/internal/handler"
	"github.com/ai-go-hub/ai-go-admin/internal/model"
	svcAuth "github.com/ai-go-hub/ai-go-admin/internal/service/admin/auth"

	"github.com/gin-gonic/gin"
)

// AuthAdminHandler 管理员账号管理控制器
type AuthAdminHandler struct {
	*handler.Handler[model.Admin]
	svc *svcAuth.AuthAdminService
}

// NewAuthAdminHandler 创建管理员账号管理控制器实例
func NewAuthAdminHandler(svc *svcAuth.AuthAdminService) *AuthAdminHandler {
	return &AuthAdminHandler{
		Handler: handler.NewHandler(svc,
			handler.WithOmitFields(handler.OmitFields{
				// 创建时忽略以下字段不入库
				Create: []string{"id", "login_failure", "last_login_at", "last_login_ip", "deleted_at"},
			}),
		),
		svc: svc,
	}
}

// RegisterRoutes 注册路由
func (h *AuthAdminHandler) RegisterRoutes(group *gin.RouterGroup) {
	// 这种写法可自动挂载重写后的方法
	handler.RegisterBaseRoutes(h, group)
}
```

**服务：internal\service\admin\auth\admin.go**

```go
package auth

import (
	"github.com/ai-go-hub/ai-go-admin/internal/model"
	repoAdmin "github.com/ai-go-hub/ai-go-admin/internal/repository/admin"
	"github.com/ai-go-hub/ai-go-admin/internal/service"
)

// AuthAdminService 管理员账号管理服务
type AuthAdminService struct {
	service.IService[model.Admin]
	repo *repoAdmin.AdminRepository
}

// NewAuthAdminService 创建管理员账号管理服务实例
func NewAuthAdminService(repo *repoAdmin.AdminRepository) *AuthAdminService {
	return &AuthAdminService{
		IService: service.NewService(repo),
		repo:     repo,
	}
}
```

## 接口个性化改造

管理员账号管理接口，当然不是零定制版就能搞定的，主要是有以下数据验证项：

### 数据验证

如果是单纯的字段必填的话，可以在 `Admin` 模型增加 `binding:"required"` 即可，如下：

```go
Password string `gorm:"comment:密码;type:varchar(255);not null" json:"-" binding:"required"`
```

但是管理员的新增接口，`password` 字段是必填的，而编辑接口，`password` 字段则是留空则不修改，而且还要在服务层，对密码进行加密入库，所以我选择自定义 `DTO`，如下：

```go
// AdminUpdateRequest 更新管理员请求参数
type AdminUpdateRequest struct {
	Username string `json:"username" binding:"required"`
	Nickname string `json:"nickname" binding:"required"`
	Avatar   string `json:"avatar"`
	Email    string `json:"email"`
	Mobile   string `json:"mobile"`
	Password string `json:"password"`
	Bio      string `json:"bio"`
	Status   string `json:"status" binding:"required"`
}

// AdminCreateRequest 创建管理员请求参数
type AdminCreateRequest struct {
	AdminUpdateRequest
	Password string `json:"password" binding:"required"`
}
```

考虑到服务层还得对密码加密入库，我们直接对控制器和服务层的 `新增、编辑` 两个方法进行重写：

一、方便使用以上 `DTO` 接受请求数据；
二、密码入库方便（新增是直接加密入库，编辑是有设定密码才入库）；
三、用户名重复检查也做到了服务层（涉及多字段关联的验证，一律建议放在服务层）。

### 管理员权限

目前的管理员管理还不支持设定 `管理员分组` 字段，有了这个字段才能决定管理员有那些权限，甚至能否新建管理员等，由于目前还没有管理员分组功能，只能留待后续完善。