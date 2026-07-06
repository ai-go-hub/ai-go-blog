# 从 PHP 到 AI + Golang，程序员自救转型手记（二十四）：一行提示词抽出完美网站首页

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-admin（[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “登录接口整合点选验证码（AI 大翻车）”，本期将完成：一行提示词抽出完美网站首页

# 网站首页制作

本项目目前并不确定是否制作 PC 会员端，就算做应该也会使用更兼容 SEO 的 Nuxt，而不是原生 VUE3 代码编写。

但是访问站点，又不能直接跳转到后台登录页面（未来甚至需要自定义后台入口以达到隐藏后台的目的），所以此时我们要做的首页，其实就是个占位的页面，随便弄一下就行了。

这里让 CC 随便给我们做一个，先看看效果：作者于仓库 README 复制了一行介绍，然后要求它使用 `vue3` 和 `element plus`，如下：

```bash
帮我于 @src\views\index.vue 设计一个简洁大气的首页，使用当前已有依赖包，如 vue 和 element plus；项目是基于 Golang + GORM + TypeScript + Vite + Pinia + Element Plus 等流行技术栈的后台管理系统，目前开发工作以【AI执笔 + 人工逐行雕琢】的模式积极进行中，作者指挥左手 AI，右手同步手搓 Blog，全程记录开发细节。
```

我去，不错，真不错啊，我先看了代码，一共只动了三个文件，然后是效果，大家自己看吧：

![](/images/24/1.png)

![](/images/24/2.png)

![](/images/24/3.png)


还是响应式的，自然兼容不同屏幕的设备。

作者目检了每一行代码，保守一点，90% 以上无需调整，代码非常干净，找到这么几个问题：

#### scss 变量重复定义

AI 在 scss 部分开篇就定义了后续需要用的变量，不过 `$primary`、`$text-primary` 这些都是 `element plus` 自带有的，这边要求它直接使用 el 定义好的变量

```scss
<style scoped lang="scss">
$primary: #409eff;
$primary-dark: #337ecc;
$text-primary: #1f2937;
$text-secondary: #6b7280;
$text-muted: #9ca3af;
$bg-light: #f9fafb;
$bg-white: #ffffff;
$border: #e5e7eb;
// ...
</style>
```

```bash
● 最后全面检查：确认没有遗漏的非法 rgba(var(...)) 写法，也没有 SCSS 变量残留。

  Thought for 2s, searched for 1 pattern, called ide (ctrl+o to expand)

● 零诊断，全部干净。剩余的 rgba() 都是静态值（rgba(255,255,255, X) 和 rgba(0,0,0, X)），无需 CSS 变量参与，完全合法。
```

#### 文案调整

接下来几乎所有的调整，都是文案上的，比如它在介绍 TypeScript 时，提到了 Vue3 和 Vite，而下方又有单独的 Vue3 介绍，然后就是文案过长，和优化，比如 `AI 执笔 · 人工雕琢` 我改为了 `AI 做舟 · 人掌分寸`，都是很细枝末节的问题。

#### 页头页脚封装为组件？

否，正如本文开篇所述，首页都只是一个占位页面，主要的是后台，后台用不上这种页头页脚组件。

#### 代码开源

完整代码开源于：[github](https://github.com/ai-go-hub/ai-go-admin) | [gitee](https://gitee.com/ai-go-hub/ai-go-admin)

本次生成的首页完整代码如下，有兴趣的开发者，可以再帮忙 `review` 一遍：

```vue
<template>
    <div class="home-page">
        <!-- ==================== 导航栏 ==================== -->
        <header class="navbar">
            <div class="navbar-inner">
                <span class="logo">AI GO MALL</span>
                <nav class="nav-links">
                    <a href="https://github.com/ai-go-hub/ai-go-admin" target="_blank" class="nav-link">GitHub</a>
                    <a href="https://gitee.com/ai-go-hub/ai-go-admin" target="_blank" class="nav-link">Gitee</a>
                </nav>
            </div>
        </header>

        <!-- ==================== 主视觉区 ==================== -->
        <section class="hero">
            <div class="hero-bg-deco">
                <div class="deco-blob deco-blob-1" />
                <div class="deco-blob deco-blob-2" />
                <div class="deco-grid" />
            </div>
            <div class="hero-content">
                <div class="badge-row">
                    <span class="ai-badge">
                        <el-icon><Cpu /></el-icon>
                        AI 做舟，人掌分寸
                    </span>
                </div>
                <h1 class="hero-title">
                    <span class="title-line">AI GO MALL</span>
                </h1>
                <p class="hero-desc">基于 Golang + GORM + TypeScript + Vite + Element Plus 等流行技术栈</p>
                <div class="hero-actions">
                    <el-button size="large" type="primary" round @click="$router.push('/admin')"> 进入后台 </el-button>
                    <el-button size="large" round @click="scrollToTech"> 了解技术栈 </el-button>
                </div>
            </div>
            <div class="scroll-hint" @click="scrollToTech">
                <span>向下滚动</span>
                <el-icon class="bounce-arrow"><ArrowDown /></el-icon>
            </div>
        </section>

        <!-- ==================== 技术栈 ==================== -->
        <section ref="techSection" class="tech-section">
            <div class="section-header">
                <h2>技术栈</h2>
                <p>流行、稳定、高效的技术选型</p>
            </div>
            <div class="tech-grid">
                <div v-for="item in techStack" :key="item.name" class="tech-card">
                    <div class="tech-icon" :style="{ background: item.gradient }">
                        <el-icon :size="28"><component :is="item.icon" /></el-icon>
                    </div>
                    <h3>{{ item.name }}</h3>
                    <p>{{ item.desc }}</p>
                </div>
            </div>
        </section>

        <!-- ==================== 开发理念 ==================== -->
        <section class="philosophy-section">
            <div class="section-header">
                <h2>开发理念</h2>
                <p>AI 执笔快速构建，人掌分寸雕琢商用级电商工程</p>
            </div>
            <div class="philosophy-content">
                <div class="philosophy-card human">
                    <div class="card-icon">
                        <el-icon :size="40"><UserFilled /></el-icon>
                    </div>
                    <h3>人工指挥</h3>
                    <p>作者以丰富的开发经验把控项目架构与方向，确保每一行代码都经过深思熟虑，杜绝 AI 幻觉带来的隐患。</p>
                </div>
                <div class="philosophy-divider">
                    <div class="divider-line" />
                    <span class="divider-plus">
                        <el-icon :size="24"><Plus /></el-icon>
                    </span>
                    <div class="divider-line" />
                </div>
                <div class="philosophy-card ai">
                    <div class="card-icon">
                        <el-icon :size="40"><Cpu /></el-icon>
                    </div>
                    <h3>AI 执笔</h3>
                    <p>Claude Code 作为主力编码助手，高效完成重复性工作与代码生成，大幅提升开发效率与代码质量。</p>
                </div>
            </div>
        </section>

        <!-- ==================== 博客链接 ==================== -->
        <section class="blog-section">
            <div class="section-header">
                <h2>开源博客</h2>
                <p>全程记录开发细节，与社区共同成长</p>
            </div>
            <div class="blog-cards">
                <a href="https://github.com/ai-go-hub/ai-go-blog" target="_blank" class="blog-card">
                    <div class="blog-card-icon github">
                        <svg viewBox="0 0 24 24" fill="currentColor" class="github-svg">
                            <path
                                d="M12 0C5.37 0 0 5.37 0 12c0 5.3 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61-.546-1.385-1.335-1.755-1.335-1.755-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.605-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 21.795 24 17.295 24 12 24 5.37 18.63 0 12 0z"
                            />
                        </svg>
                    </div>
                    <div class="blog-card-body">
                        <h3>GitHub</h3>
                        <p>github.com/ai-go-hub/ai-go-blog</p>
                    </div>
                    <el-icon class="external-icon"><ArrowRight /></el-icon>
                </a>

                <a href="https://gitee.com/ai-go-hub/ai-go-blog" target="_blank" class="blog-card">
                    <div class="blog-card-icon gitee">
                        <svg viewBox="0 0 24 24" fill="currentColor" class="gitee-svg">
                            <path
                                d="M11.984 0A12 12 0 0 0 0 12a12 12 0 0 0 12 12 12 12 0 0 0 12-12A12 12 0 0 0 12 0zm6.09 5.333c.328 0 .593.266.592.593v1.482a.594.594 0 0 1-.593.592H9.778c-.328 0-.593-.266-.592-.593V5.926a.594.594 0 0 1 .593-.592h8.296zm-4.63 4.815h4.63c.328 0 .593.266.593.593v1.482a.594.594 0 0 1-.593.592H9.778a.594.594 0 0 1-.593-.592v-1.482a.594.594 0 0 1 .593-.593h3.667zm4.63 3.852c.328 0 .593.266.593.593v1.482a.594.594 0 0 1-.593.592H9.778a.594.594 0 0 1-.593-.592v-1.482a.594.594 0 0 1 .593-.593h8.296z"
                            />
                        </svg>
                    </div>
                    <div class="blog-card-body">
                        <h3>Gitee</h3>
                        <p>gitee.com/ai-go-hub/ai-go-blog</p>
                    </div>
                    <el-icon class="external-icon"><ArrowRight /></el-icon>
                </a>
            </div>
        </section>

        <!-- ==================== 页脚 ==================== -->
        <footer class="footer">
            <div class="footer-inner">
                <div class="footer-brand">
                    <span class="footer-logo">AI GO ADMIN</span>
                    <span class="footer-tagline">AI 做舟 · 人掌分寸</span>
                </div>
                <div class="footer-links">
                    <a href="https://github.com/ai-go-hub/ai-go-admin" target="_blank">GitHub</a>
                    <span class="footer-divider">·</span>
                    <a href="https://gitee.com/ai-go-hub/ai-go-admin" target="_blank">Gitee</a>
                    <span class="footer-divider">·</span>
                    <a href="http://beian.miit.gov.cn/" target="_blank">渝ICP备8888888号-1</a>
                </div>
                <p class="footer-copy">&copy; {{ new Date().getFullYear() }} AI GO ADMIN. All rights reserved.</p>
            </div>
        </footer>
    </div>
</template>

<script setup lang="ts">
import { ArrowDown, ArrowRight, Coin, Connection, Cpu, DataBoard, Grid, Monitor, Plus, Setting, UserFilled } from '@element-plus/icons-vue'
import { ref } from 'vue'

const techSection = ref<HTMLElement | null>(null)

function scrollToTech() {
    techSection.value?.scrollIntoView({ behavior: 'smooth' })
}

interface TechItem {
    name: string
    desc: string
    icon: any
    gradient: string
}

const techStack: TechItem[] = [
    {
        name: 'Golang',
        desc: '高性能后端语言，Gin 框架 + GORM + 泛型驱动的服务端架构',
        icon: Monitor,
        gradient: 'linear-gradient(135deg, #00ADD8, #00ADD8cc)',
    },
    {
        name: 'TypeScript',
        desc: '类型安全的 JavaScript 超集，前端顶流，Vue3 使用它编写',
        icon: DataBoard,
        gradient: 'linear-gradient(135deg, #3178C6, #3178C6cc)',
    },
    {
        name: 'PostgreSQL',
        desc: '强大的开源关系型数据库，GORM AutoMigrate 自动建表，读写分离',
        icon: Coin,
        gradient: 'linear-gradient(135deg, #4169E1, #4169E1cc)',
    },
    {
        name: 'Vue 3 + Pinia',
        desc: '渐进式前端框架搭配轻量级状态管理，setup 语法糖 + Vite 构建',
        icon: Connection,
        gradient: 'linear-gradient(135deg, #4FC08D, #4FC08Dcc)',
    },
    {
        name: 'Element Plus',
        desc: 'Vue 3 生态最流行的组件库，统一的 UI 风格，国际化开箱即用',
        icon: Grid,
        gradient: 'linear-gradient(135deg, #409EFF, #409EFFcc)',
    },
    {
        name: 'Claude Code',
        desc: 'AI 驱动的编码助手，负责代码生成、重构与审查，人机协同高效开发',
        icon: Setting,
        gradient: 'linear-gradient(135deg, #D97706, #D97706cc)',
    },
]
</script>

<style scoped lang="scss">
// ==================== 导航栏 ====================

.navbar {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    z-index: 100;
    background: rgba(255, 255, 255, 0.8);
    backdrop-filter: blur(12px);
    border-bottom: 1px solid rgba(0, 0, 0, 0.05);
}

.navbar-inner {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 32px;
    height: 64px;
    display: flex;
    align-items: center;
    justify-content: space-between;
}

.logo {
    font-size: 18px;
    font-weight: 700;
    color: var(--el-text-color-primary);
    letter-spacing: -0.5px;
}

.nav-links {
    display: flex;
    align-items: center;
    gap: 16px;
}

.nav-link {
    font-size: 14px;
    color: var(--el-text-color-regular);
    text-decoration: none;
    transition: color 0.2s;

    &:hover {
        color: var(--el-color-primary);
    }
}

// ==================== 主视觉区 ====================

.hero {
    position: relative;
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    overflow: hidden;
    padding: 96px 32px 64px;
}

.hero-bg-deco {
    position: absolute;
    inset: 0;
    pointer-events: none;
}

.deco-blob {
    position: absolute;
    border-radius: 50%;
    filter: blur(80px);
    opacity: 0.6;
}

.deco-blob-1 {
    top: -20%;
    right: -10%;
    width: 500px;
    height: 500px;
    background: color-mix(in srgb, var(--el-color-primary) 15%, transparent);
}

.deco-blob-2 {
    bottom: -10%;
    left: -10%;
    width: 400px;
    height: 400px;
    background: color-mix(in srgb, var(--el-color-primary) 10%, transparent);
}

.deco-grid {
    position: absolute;
    inset: 0;
    background-image:
        linear-gradient(to right, color-mix(in srgb, var(--el-color-primary) 4%, transparent) 1px, transparent 1px),
        linear-gradient(to bottom, color-mix(in srgb, var(--el-color-primary) 4%, transparent) 1px, transparent 1px);
    background-size: 60px 60px;
    mask-image: radial-gradient(ellipse at center, black 30%, transparent 70%);
}

.hero-content {
    position: relative;
    z-index: 10;
    text-align: center;
    max-width: 720px;
}

.badge-row {
    margin-bottom: 24px;
}

.ai-badge {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 16px;
    background: color-mix(in srgb, var(--el-color-primary) 8%, transparent);
    color: var(--el-color-primary);
    font-size: 13px;
    font-weight: 500;
    border-radius: 100px;
    border: 1px solid color-mix(in srgb, var(--el-color-primary) 15%, transparent);
}

.hero-title {
    margin-bottom: 24px;

    .title-line {
        display: block;
        font-size: 56px;
        font-weight: 800;
        color: var(--el-text-color-primary);
        letter-spacing: -1.5px;
        line-height: 1.2;
    }
}

.hero-desc {
    font-size: 16px;
    color: var(--el-text-color-regular);
    line-height: 1.8;
    margin-bottom: 40px;
    max-width: 560px;
    margin-left: auto;
    margin-right: auto;
}

.hero-actions {
    display: flex;
    gap: 16px;
    justify-content: center;
    flex-wrap: wrap;
}

.scroll-hint {
    position: absolute;
    bottom: 32px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    color: var(--el-text-color-secondary);
    font-size: 12px;
    cursor: pointer;
    z-index: 10;
    transition: color 0.2s;

    &:hover {
        color: var(--el-color-primary);
    }
}

.bounce-arrow {
    animation: bounce 2s infinite;
}

@keyframes bounce {
    0%,
    100% {
        transform: translateY(0);
    }
    50% {
        transform: translateY(6px);
    }
}

// ==================== 通用 section 头部 ====================

.section-header {
    text-align: center;
    margin-bottom: 56px;

    h2 {
        font-size: 32px;
        font-weight: 700;
        color: var(--el-text-color-primary);
        letter-spacing: -0.5px;
        margin: 0 0 12px;
    }

    p {
        font-size: 16px;
        color: var(--el-text-color-regular);
        margin: 0;
    }
}

// ==================== 技术栈 ====================

.tech-section {
    padding: 100px 32px;
    background: var(--el-fill-color-light);
}

.tech-grid {
    max-width: 1000px;
    margin: 0 auto;
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 24px;
}

.tech-card {
    background: var(--el-bg-color);
    border-radius: 16px;
    padding: 32px 28px;
    border: 1px solid var(--el-border-color-light);
    transition: all 0.3s ease;

    &:hover {
        transform: translateY(-4px);
        box-shadow: 0 12px 40px rgba(0, 0, 0, 0.08);
    }

    .tech-icon {
        width: 56px;
        height: 56px;
        border-radius: 14px;
        display: flex;
        align-items: center;
        justify-content: center;
        color: white;
        margin-bottom: 20px;
    }

    h3 {
        font-size: 18px;
        font-weight: 600;
        color: var(--el-text-color-primary);
        margin: 0 0 8px;
    }

    p {
        font-size: 14px;
        color: var(--el-text-color-regular);
        line-height: 1.7;
        margin: 0;
    }
}

// ==================== 开发理念 ====================

.philosophy-section {
    padding: 100px 32px;
}

.philosophy-content {
    max-width: 800px;
    margin: 0 auto;
    display: flex;
    align-items: center;
    gap: 32px;
}

.philosophy-card {
    flex: 1;
    text-align: center;
    padding: 40px 28px;
    border-radius: 16px;
    border: 1px solid var(--el-border-color-light);
    transition: all 0.3s ease;

    &:hover {
        transform: translateY(-4px);
        box-shadow: 0 12px 40px rgba(0, 0, 0, 0.08);
    }

    .card-icon {
        width: 80px;
        height: 80px;
        border-radius: 50%;
        display: flex;
        align-items: center;
        justify-content: center;
        margin: 0 auto 24px;
    }

    h3 {
        font-size: 20px;
        font-weight: 600;
        color: var(--el-text-color-primary);
        margin: 0 0 12px;
    }

    p {
        font-size: 14px;
        color: var(--el-text-color-regular);
        line-height: 1.8;
        margin: 0;
    }
}

.philosophy-card.human {
    background: var(--el-bg-color);

    .card-icon {
        background: color-mix(in srgb, var(--el-color-primary) 8%, transparent);
        color: var(--el-color-primary);
    }
}

.philosophy-card.ai {
    background: linear-gradient(135deg, #1f2937, #111827);

    h3 {
        color: #f9fafb;
    }

    p {
        color: #9ca3af;
    }

    .card-icon {
        background: rgba(255, 255, 255, 0.1);
        color: #f9fafb;
    }
}

.philosophy-divider {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 12px;
    flex-shrink: 0;
}

.divider-line {
    width: 1px;
    height: 40px;
    background: var(--el-border-color-light);
}

.divider-plus {
    width: 40px;
    height: 40px;
    border-radius: 50%;
    background: var(--el-fill-color-light);
    border: 1px solid var(--el-border-color-light);
    display: flex;
    align-items: center;
    justify-content: center;
    color: var(--el-text-color-secondary);
}

// ==================== 博客链接 ====================

.blog-section {
    padding: 100px 32px;
    background: var(--el-fill-color-light);
}

.blog-cards {
    max-width: 640px;
    margin: 0 auto;
    display: flex;
    flex-direction: column;
    gap: 16px;
}

.blog-card {
    display: flex;
    align-items: center;
    gap: 20px;
    padding: 24px 28px;
    background: var(--el-bg-color);
    border-radius: 16px;
    border: 1px solid var(--el-border-color-light);
    text-decoration: none;
    transition: all 0.3s ease;

    &:hover {
        transform: translateY(-2px);
        box-shadow: 0 8px 30px rgba(0, 0, 0, 0.06);
    }
}

.blog-card-icon {
    width: 52px;
    height: 52px;
    border-radius: 14px;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
    color: white;

    &.github {
        background: #24292e;
    }

    &.gitee {
        background: #c71d23;
    }
}

.github-svg,
.gitee-svg {
    width: 28px;
    height: 28px;
}

.blog-card-body {
    flex: 1;

    h3 {
        font-size: 16px;
        font-weight: 600;
        color: var(--el-text-color-primary);
        margin: 0 0 4px;
    }

    p {
        font-size: 13px;
        color: var(--el-text-color-secondary);
        margin: 0;
        font-family: 'SF Mono', 'Fira Code', monospace;
    }
}

.external-icon {
    color: var(--el-text-color-secondary);
    flex-shrink: 0;
    transition: all 0.2s;

    .blog-card:hover & {
        color: var(--el-color-primary);
        transform: translateX(2px);
    }
}

// ==================== 页脚 ====================

.footer {
    padding: 48px 32px;
    border-top: 1px solid var(--el-border-color-light);
    background: var(--el-bg-color);
}

.footer-inner {
    max-width: 1200px;
    margin: 0 auto;
    text-align: center;
}

.footer-brand {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 12px;
    margin-bottom: 16px;
}

.footer-logo {
    font-size: 16px;
    font-weight: 700;
    color: var(--el-text-color-primary);
}

.footer-tagline {
    font-size: 13px;
    color: var(--el-text-color-secondary);
    padding: 2px 10px;
    background: var(--el-fill-color-light);
    border-radius: 100px;
}

.footer-links {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    margin-bottom: 12px;

    a {
        font-size: 13px;
        color: var(--el-text-color-secondary);
        text-decoration: none;
        transition: color 0.2s;

        &:hover {
            color: var(--el-color-primary);
        }
    }
}

.footer-divider {
    color: var(--el-text-color-primary);
}

.footer-copy {
    font-size: 12px;
    color: var(--el-text-color-secondary);
    margin: 0;
}

// ==================== 响应式 ====================

@media (max-width: 900px) {
    .tech-grid {
        grid-template-columns: repeat(2, 1fr);
    }

    .philosophy-content {
        flex-direction: column;
    }

    .philosophy-divider {
        flex-direction: row;

        .divider-line {
            width: 40px;
            height: 1px;
        }
    }
}

@media (max-width: 640px) {
    .hero-title {
        .title-line {
            font-size: 36px;
        }
    }

    .hero-desc {
        font-size: 14px;
    }

    .tech-grid {
        grid-template-columns: 1fr;
    }

    .section-header h2 {
        font-size: 24px;
    }

    .hero-actions {
        flex-direction: column;
        align-items: center;
    }
}
</style>
```