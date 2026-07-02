# 从 PHP 到 AI + Golang，程序员自救转型手记（十九）：点选验证码代码逐行目检

这是一个系列 Blog，作者将以一个 PHP 全栈工程师的身份，利用 AI 工具（claude code、codex、deepseek、豆包等）：从零开始学习 golang 语言，并最终完成 ai-go-mall（[github](https://github.com/ai-go-hub/ai-go-mall) | [gitee](https://gitee.com/ai-go-hub/ai-go-mall)）开源项目的制作，全程记录分享。

在上一期，我们已经完成 “点选验证码初版、封装随机数与文件系统包”，本期将完成：点选验证码代码逐行目检

# 点选验证码代码逐行目检

### 命名细节改动

以下结构能用，但是 `ImageBase64` 有 Image 前缀，而 Width 和 Height 也代表了最终图片的宽高，却不带 Image 前缀，让 AI 补上

```go
// ClickResult 创建验证码的返回结果
type ClickResult struct {
	Key         string   `json:"key"`
	Width       int      `json:"width"`
	Height      int      `json:"height"`
	Elements    []string `json:"elements"`
	ImageBase64 string   `json:"image_base64"`
}
```

### 可封装的函数

有一个用于读取背景图片和图标的 `readDir` 函数，可将其迁移至 `pkg/filesystem` 包，并改为支持递归读取子目录，获取所有文件的列表

```go
func readDir(dir string) ([]string, error) {
	entries, err := os.ReadDir(dir)
	if err != nil {
		return nil, err
	}
	var paths []string
	for _, e := range entries {
		if !e.IsDir() {
			paths = append(paths, filepath.Join(dir, e.Name()))
		}
	}
	return paths, nil
}
```

### 变量异常流转

如下提取代码，有个 `iconPaths` 变量，在 `Create` 函数内读取到之后，会多处传递使用

```go
iconPaths, err := readDir(cfg.IconDir)

correctElements := generateElements(cfg.Length, cfg.Elements, iconPaths)
confusionElements := generateElements(cfg.ConfusionLength, cfg.Elements, iconPaths)
info, rect, drawErr := drawElement(bg, elem, bgW, bgH, iconPaths, placedRects)
drawIcon(bg, elem, bgW, bgH, iconPaths, placed)
```

1. 改为验证码配置文件中（icon 路径配置处）声明一个 `icons` 数组，数组中存储 `icon` 文件名称、中文名称（通过文件名称翻译）、文件路径，全局读取后使用，无需四处传递
2. `Create` 函数输出的 `Elements` 若为 `icon`，应使用中文名称，名称使用 <> 包裹，如 <图标中文名称>、<汉字>、<大写字母>

预感这几点修改会比较大，先让它做，之后重新 `review`：

```bash
全部完成。四个问题的改动总结...
```

### 配置文件可读性优化

这一轮中，图标数据放到了配置文件中，但是不太好看：

```yaml
# 图标元数据
icons:
- name: "aeroplane"
    cn_name: "飞机"
- name: "apple"
    cn_name: "苹果"
```

让它重新以文件名为 `en_name`，中文名为 `cn_name`，且保留未来可能增加其他语言的 `name` 的可能性：

```yaml
  icons:
    - en_name: "aeroplane"
      cn_name: "飞机"
    - en_name: "apple"
      cn_name: "苹果"
    - en_name: "banana"
      cn_name: "香蕉"
    # ......
```

稍微好看了一点，不过还是将 icons 放在配置文件的最下面去，会好一些。

### 代码可读性优化

目前的 `internal\infra\captcha\click.go` 文件中，先于 `Create（创建验证码函数）` 内调用了 `清理过期验证码` 和 `图标初始化` 函数；

然后在文件较下面的 `init` 函数中初始化了中文字符池，内容大概如下：

```go
pool := "的一是在不了有和人这中大为上个国我以要他..."
chineseRunes = []rune(pool)
```

相当于有个 `init`，但是又将 `清理过期验证码` 和 `图标初始化` 这两样放在了 `Create` 才执行，这可能是我之前某句提示词误导了它，`图标初始化` 放到 `Create` 还勉强可以理解，`清理过期验证码` 其实是放 `init` 更合理的，这里干脆保持统一，初始化相关动作全部都放到 `init`，`Create` 不再单独初始化某些必要数据，而是直接使用，让其主打自己该做的工作。

同时，将 `init` 函数移到文件的最上面，提高人类的可阅读性（PS：`init` 是 `go` 语言中一个特殊的函数，当某包被导入时，会自动执行包的 init，有多个 init 函数时会按顺序执行）。

让 AI 修改时，它告诉我这样做是有原因的：

```bash
init() 执行时 config.Init() 和 database.Init() 尚未运行，但 icons 依赖配置、cleanExpired 依赖数据库，所以它们无法放入 init()。
最接近你意图的方案：用单个 sync.Once 引导函数统一三者，Create 只需一行调用。
```

AI 的意思是 init 时无法完成所有的初始化工作，因为 `config.Init()` 尚未调用，然后它实现了一个 `bootstrap` 函数，在 `Create` 开头调用一次，init 还是保留了中文字符池的初始化功能，相当于还是有两个初始化函数，这里干脆让它将 `init` 去掉，初始化工作全部放到 `bootstrap` 函数中完成，内部分为两部分，只需要执行一次的放入 `sync.Once`，`清理过期验证码` 需要重复执行则不能放入 `sync.Once`。

既然有统一的全局初始化函数了，再让 AI 将 `cfg := config.Get().Captcha` 中的 `cfg` 这类公共变量，也都提取为全局变量，不要四处单独赋值了。

### 理解代码

代码目测无结构问题，无多余封装，整体干净简洁，逻辑顺畅清晰，但是其中很多代码我也不能理解，比如：

```go
func loadImage(path string) (*image.RGBA, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	defer f.Close()
	img, err := png.Decode(f)
	if err != nil {
		return nil, err
	}
	rgba := image.NewRGBA(img.Bounds())
	draw.Draw(rgba, rgba.Bounds(), img, img.Bounds().Min, draw.Src)
	return rgba, nil
}
```

这是加载背景图片的函数，其中使用了 `image.NewRGBA`，`img.Bounds()`，`draw.Draw` 等我从来没见过的东西，这里直接复制出来，让豆包给我解释一下。

我们的目标是利用 AI 学习 go 语言，出现看不懂的代码，正好是学习的时候了，而且作为开发者，我们最终也需要实现整个文件所有代码的理解，先干了什么，后干了什么，怎么干的，甚至做一些微调或个性化的改变，先理解，再人工微调：

```go
img, err := png.Decode(f)
if err != nil {
	return nil, err
}

// 改为

img, _, err := image.Decode(f)
if err != nil {
	return nil, err
}

// ==============================================

rgba := image.NewRGBA(img.Bounds())
draw.Draw(rgba, rgba.Bounds(), img, img.Bounds().Min, draw.Src)
return rgba, nil

// 改为

bounds := img.Bounds()
rgba := image.NewRGBA(bounds)
draw.Draw(rgba, rgba.Bounds(), img, bounds.Min, draw.Src)
return rgba, nil
```