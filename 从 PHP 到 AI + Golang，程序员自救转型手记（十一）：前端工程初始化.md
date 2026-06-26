# 从 PHP 到 AI + Golang，程序员自救转型手记（十一）：前端工程初始化

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “交叉验证AI模型的工作成果”，本期将完成：前端工程初始化

# 前端工程初始化

下一步打算制作人机验证码与 token 生成，最终完成真实完整的管理员登录功能，这需要结合前端技术（要写页面了），虽然一开始的定位就是全栈的完整项目，但还是额外提一下全栈的必要性：

从项目层面而言，ai-go-mall 作为一个完整的商城项目，自然应该含有前后两端（而且后续可能还会有 uniapp 或其他移动端工程）。

从个人发展层面而言，你的根扎得越深，上面就越稳，不要去相信那些弱者所说的学多了杂而不精，作为程序员从 `硬件层 > 操作系统 > 网络 > 数据于编程语言 > 工程 > 部署` 等都值得了解，然后择自己兴趣或工作需要发展专精项即可（实不相瞒，作者修电脑也是一把好手），能独立完成项目的开发者完全是另外一个等级，比如在初创公司时，甚至可以一个人撑起一个研发部。

从大环境层面而言，如今的 AI 时代，纯前端开发者前景堪忧，特别是**只会写页面**的前端或许已经死了，这些工作将被后端工程师接手，同时后端工程师自动升级全栈工程师，高阶开发者没有一个是只会一种语言的，接下来，我们将以 vue3 为前端框架，初始化 ai-go-mall 的前端工程。

前端工程大范围复用作者的另外一个项目 [BuildAdmin](https://gitee.com/wonderful-code/buildadmin) 的前端代码（即 web 目录内的代码），会更加彻底的减少封装和黑盒，回归原生，让整个工程能和 AI 配合融洽。

如果你完全不会前端或完全不会 vue，可以对 typescript、vue 略作学习，它们的学习资料很全面，如果你要学的深入一点的话，可能需要将近一个礼拜的时间，本项中，作者会尽量克制，`渐进式` 的来写前端代码。

## 开始

于根目录创建 web 文件夹，后续前端相关的全部都放在此文件夹内（前后端分离，只是放在同一个代码仓库），接下来：

> 建议单独使用 vscode 打开 web 目录，服务端和前端各一个 vscode 窗口，更方便开发

#### 目录结构建立

WEB 工程的根目录结构如下，是一个经典的前端目录结构设计：

```bash
ai-go-mall/web
│
├─.vscode vscode 编辑器相关文件
│
├─public 公共文件
│  └─favicon.ico
│
├─src
│  │  App.vue
│  │  main.ts
│  │
│  ├─api 所有接口请求函数及接口相关
│  │
│  ├─assets 静态资源
│  │
│  ├─components 自定义组件
│  │
│  ├─lang 所有语言包
│  │
│  ├─layouts 布局
│  │
│  ├─router 前端路由
│  │
│  ├─stores 状态商店
│  │  │
│  │  ├─constant 常量定义
│  │  │      cacheKey.ts 缓存 Keys
│  │  │
│  │  └─interface 各种接口定义
│  │
│  ├─styles 各种全局样式表
│  │
│  ├─utils 工具库
│  │
│  └─views 页面
│
├─types 全局类型定义
│
│  .editorconfig IDE 风格统一配置
│  .env 基础环境变量定义
│  .env.development 开发环境变量定义
│  .env.production 生产环境变量定义
│  .prettierrc.js prettier 配置
│  eslint.config.js eslint 配置
│  index.html 入口文件
│  package.json 包配置文件
│  tsconfig.json ts 配置
└─ vite.config.ts vite 配置
```

目前文件夹内基本都是空的，只是搭了个结构（目录可以直接让 ai 按以上结构创建）；

主要是根目录的一些基本配置文件，我是直接复制于 `BuildAdmin/web`，并且仔细核对了一遍没有多余或不当的。