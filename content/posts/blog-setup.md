---
title: "本博客的搭建流程"
date: 2022-01-22
categories: ["手册"]
tags: ["手册", "博客"]
summary: '第一次用 GitHub Pages + Hugo 搭建博客，在主题和外观配置上花了最长时间。本文对整个搭建流程进行了整理。'
---

[GitHub Pages](https://pages.github.com/) 是 Github 提供的一项服务，让你能不需要主机和域名，通过仓库就能建立个人网站。

用 GitHub Pages 搭建的大都是静态网站。静态网站是指提前编译好文件，访问时直接返回它们的网站。与之相对的是动态网站，每次访问时服务器需要重新生成文件。由于内容变化不频繁，个人博客基本都是静态网站。

通过编写 HTML/CSS/JavaScript，理论上能在静态网站上实现各种功能，比如样式、评论、标签等。但这费时费力，对于没有前端经验的人来说也不现实。而**静态网站生成器**能解决这个痛点——依靠它，你能用很简单的方式来定制化，最后生成前端文件即可，大大减少了工作量。

GitHub Pages 常搭配的静态网站生成器有 Jekyll、Hexo 和 Hugo。

- Jekyll 基于 Ruby，GitHub Pages 原生支持，最老牌；
- Hexo 基于 Node.js，定制能力强，主题丰富；
- Hugo 基于 Go，依赖简单，生成速度快。

我并不需求花哨的外观或功能，追求简单快速，因此选择 Hugo。GitHub Pages 托管网站，Hugo 提供样式和功能，两者结合能打造出酷炫好看的个人博客。

## 建站流程

本章介绍用 GitHub Pages + Hugo 来搭建本博客的流程。

### 1. 设置 GitHub Pages

用 GitHub Pages 搭建个人网站的流程非常简单：

1. [新建](https://github.com/new)一个名字为`<username>.github.io`的仓库，`<username>`必须是你的 Github 账户名。
2. 在仓库根目录下新建文件 `index.html`并提交修改。这个文件里可以是任意内容。
3. 在浏览器里访问 `<username>.github.io`，就能看到搭好的 GitHub Page。

访问 `https://<username>.github.io` 时，Github 会加载 main 分支下 `index.html`；如果这个文件不存在，就加载 `README.md`；否则，将显示 404。

### 2. Hugo 初始化博客

以 `index.html` 为入口，你可以搭配 CSS 或加上超链接，这样玩法就很多了。但手动编辑太麻烦，于是我们需要 Hugo 来「让它看起来像一个博客」。

首先照着[文档](https://gohugo.io/getting-started/installing/)在电脑上安装 Hugo，然后遵循以下步骤：

1. 克隆博客仓库到本地
2. 照着[官方的 Quick Start 文档](https://gohugo.io/getting-started/quick-start/)初始化仓库。这里不再赘述，但有几点要注意：
    - 命令 `hugo server -D` 让你能预览效果。
    - 必须先在 `config.tml` 里配置好主题，否则网站无法正常显示。
3. 在 `config.toml` 里加上配置 `publishDir = 'docs'`，然后用命令 `hugo` 生成静态文件。现在 `docs/` 被设置成生成目录了，所生成的静态文件会被放在这里。
4. 提交修改并 `git push`。
5. 修改仓库的设置：Settings -> Pages -> Source
    ![](/images/2022-01-22-github-settings-pages-main.png)

现在，在浏览器访问`https://<username>.github.io`，就能看到 Hugo 带来的变化。

默认设置下，Hugo 生成的静态文件放在 `public/` 目录里，而 GitHub Pages 读取 main 分支的 `index.html`。这样 GitHub Pages 没法找到 Hugo 生成的文件，页面会是空白的。步骤 3 和 5 就是为了解决这个问题：步骤 3 让 Hugo 将静态文件生成在 `docs/`下，步骤 5 让 GitHub Pages 读取 `docs/index.html`，使之能读到正确文件。

`config.toml` 是 Hugo 的配置文件。后面要介绍的[主题配置](#主题配置)大部分都会在这个文件里。

### 3. 绑定域名

到目前为止，一个完整博客的形态已经完成。但如果你不满足于此，想用自己的域名，那遵循以下步骤：

1. 购买域名，假设是 `example.com`。
2. 在域名注册商处配置 DNS 解析：类型是 CNAME，指向 `<username>.github.io`。
2. 在仓库的 `static/`（不存在就创建一个）下新建文件  `CNAME`，内容只有一行，为`example.com`。生成时，Hugo 会 复制`static/` 里的文件到生成目录下。
2. 执行 `hugo` 重新生成静态文件，然后提交更改并 `git push`。

等一小会让 DNS 解析生效。然后，通过 `example.com` 就能访问博客，而且还是 HTTPS 的。

步骤 2 设置 CNAME，是为了让 `example.com` 指向 `<username>.github.io`，使得前者所指向的 IP 与后者相同。此时，访问 `example.com` 就与访问 `<username>.github.io` 无异。DNS解析中，ANAME 和 CNAME 分别是不同的记录类型。ANAME 为域名指向 IP，DNS 服务器会把该域名解析成 IP；而 CNAME 为域名指向域名，DNS 服务器会先把该域名解析成所指向域名，再继续往下解析，直到获得 IP。这里使用 CNAME，就能避免 `<username>.github.io` 的 IP 改变时，需要重新设置 DNS 解析。

步骤 3 和 4 设置在每次执行 `hugo` 时，在生成目录 `docs/` 下生成 `CNAME` 文件，Github 会读取这个文件让 CNAME 生效。

## 发布文章

### 新建文章


文章以 Markdown 形式存放在 `content/posts/` 下。如果想发布新文章，在 `content/posts/` 下新建 Markdown 文件，然后撰写内容即可。但更推荐的做法是用命令 `hugo new posts/<article_name>.md`，这个命令会以 `archetypes/default.md` 为模版，在 `content/posts/` 下生成 Markdown 文件。

写完文章后，要用执行 `hugo` 重新生成静态文件，然后上传到 GitHub。这样就完成了一次文章发布。

### 元数据

可以在 Markdown 文章的开头添加 YAML 格式的元数据，供 Hugo 读取。比如：

```markdown
---
title: "我的第一篇文章"
date: 2020-03-04T15:58:26+08:00
draft: false
tag: ["foo", "bar"]
author: "xyz"
summay: "This is summary"
---
```

[这里](https://hugoloveit.com/zh-cn/theme-documentation-content/#front-matter)有详细的元数据列表。

### 模版

`archetypes/default.md` 文件是文章的模版。Hugo提供了一套[模版语法](https://gohugo.io/templates/)和[内置方法](https://gohugo.io/functions)，能执行逻辑，动态地生成内容。文章模版里最常用的就是设置新文章的元数据，比如我的模版：

```markdown
---
title: "{{ substr (replace .Name "-" " ")  11 | title }}"
date: {{ substr .Name 0 10 }}
tags: ["foo"]
summary: ""
---
```

用上面的模版，执行 `hugo new posts/2022-01-20-my-first-post.md` 所生成文件 `2022-01-20-my-first-post.md` 里的内容为：

```markdown
---
title: "My First Post"
date: 2022-01-20
tags: ["bar"]
summary: ""
---
```

模版中的 `.Name`  是变量，表示文件名。另一个常用变量是  `.Data`，表示当前日期。`replace` 和 `substr` 都是 Hugo 提供的方法。`title` 字段里，先用 `replace` 将 `-` 替换成空格，再用 `substr` 截取第 11 位到结尾的字符。`date` 字段里，直接用 `substr` 截取文件名的前 10 个字符。

不仅仅是生成文章，Hugo 模版还能生成各种 Html 和其他文件，功能强大。要精通 Hugo 就得熟知模版的开发。

### 自动化发布

发布文章的流程里有一个很麻烦的地方：每次都要用执行 `hugo` 重新生成一次静态文件，然后推到远程仓库。这一步很容易忘记，而且生成的文件比较大，推送也比较慢。

我们可以依靠 GitHub Actions，将这一步自动化，这里的逻辑为：

1. 在 main 分支修改内容，并推送到仓库。
2. GitHub Actions 发现新提交，就用 `Hugo` 编译，把生成的文件提交到另一个分支 gh-pages 上。
3. GitHub Pages 监控 gh-pages 分支的内容，发现新提交就重建网页。

具体的设置步骤如下：

1. 新建分支 gh-pages 并推送到远程仓库

```sh
git checkout --orphan gh-pages
git reset --hard
git commit --allow-empty -m "Initializing gh-pages branch"
git push origin gh-pages
git checkout main
```

2. 编写 GitHub Actions 脚本并提交，具体内容可以见[这个 commit](https://github.com/fjchen7/fjchen7.github.io/commit/1ce6f830c50c1e86e672ffec18acd72a98390bf6)。
3. 配置让 GitHub Pages 读取 gh-pages 分支的根目录：修改仓库的 Settings -> Pages -> Source
    ![](/images/2022-01-22-github-settings-pages-gh-pages.png)

现在，一旦推送修改到 main 分支，GitHub Actions 会生成静态文件并提交到 gh-pages 分支。这样就省去了一步，而且源文件和编译后的文件分开了，内容更好维护。

## 主题配置

博客的基本功能都搭建好了，剩下的就是主题配置。这是个苦力活，因为很难一下子找到自己喜欢的主题，而且需要慢慢实验配置项来看是否满意。

### LoveIt

找来找去，我最后选择了 [LoveIt](https://github.com/dillonzq/LoveIt)。除了它，类似主题还有 [LeaveIt](https://github.com/liuzc/LeaveIt)（最早）、[KeepIt](https://github.com/Fastbyte01/KeepIt)、[CodeIt](https://github.com/sunt-programator/CodeIT)、[DoIt](https://github.com/HEIGE-PCloud/DoIt) 等。这类 XXIt 主题有这么多，都是因为前人不再维护，其他人 Fork 新开了新项目。

我的 LoveIt 配置可以见[这里](https://github.com/fjchen7/fjchen7.github.io/blob/main/config.toml)，都是比较细节的改动：

- [基本配置](https://github.com/fjchen7/fjchen7.github.io)：包括标题、个人说明、导航栏、评论等

- 自定义代码字体：在 `themes/LoveIt/assets/css/_overrides.scss` 里添加
    ```scss
    @import url('https://fonts.googleapis.com/css?family=Fira+Mono:400,700&display=swap&subset=latin-ext');
    $code-font-family: Fira Code, Source Code Pro, Monaco, Menlo, Fira Mono, Consolas, monospace;
    ```

    `_overrides.scss` 的内容能覆盖默认的 CSS。

- 修改GitHub的图标：在 `themes/LoveIt/assets/data/social.yml` 里将 `fa-github-alt` 改为 `fa-github`。这里用的都是 fontawesome 提供的图标。

我这做了简单的配置。如果要做深入定制化，可以参考参考别人的经验：

- [使用Hugo框架搭建博客的过程 - 主题配置](https://www.cnblogs.com/langyao/p/14285886.html)
- [Hugo系列(3.0) - LoveIt主题美化与博客功能增强](https://lewky.cn/posts/hugo-3.html)

### 其他主题

除了 LoveIt，这里还有一些不错但我没有选择的主题：

- [hugo-PaperMod](https://github.com/adityatelange/hugo-PaperMod)：布局和 LoveIt 很像，但导航栏位于文章首页，所以没用上。
- [Hugo-coder](https://github.com/luizdepra/hugo-coder)：风格深得我心，但没有标签和导航栏，也不是为国人定制的。
- [hugo-theme-even](https://github.com/olOwOlo/hugo-theme-even)：好看，该有的功能都有，但比较烂大街。
- [hugo-theme-stack](https://github.com/CaiJimmy/hugo-theme-stack)：布局丰富，是另一种风格。[这里](https://mantyke.icu/categories/%E9%BA%BB%E7%93%9C%E5%BF%AB%E9%80%9F%E5%BF%B5%E5%92%92%E4%BA%92%E5%8A%A9%E5%B0%8F%E7%BB%84/)有一些该主题的配置文章。

如果你还还获取更多主题，可以在这些地方逛逛：

- [Hugo Themes](https://themes.gohugo.io/)：官方主题库。
- [Hugo Themes Free](https://hugothemesfree.com/)

## 小结

本文介绍 Github Pages + Hugo 搭建个人博客的流程，并梳理了文章发布的过程和主题配置的大致形式。Github Pages 是一个很良心的服务，为个人博主省去了很多麻烦。而对于 Hugo，我本着「内容为上」的原则，只简单使用了开源主题，没有进行深度定制。不考虑外观，初学者用这两个工具在1～2小时里就能搭起网站。但如果考虑更为细节的样式和功能，那就需要投入很大精力了。

许多作家对于写作入门者的建议是，不要拘于形式和文笔，遵循自己的内心和想法，马上开始动笔，你会慢慢发现自己写得越来越好。我想这个观点放在这里也适用——建站才只是开始，不要过于纠结外观和功能，专注于内容才是关键。


## 参考文档

- [GitHub Pages](https://pages.github.com/)：官方文档。
- [Hugo Getting Started](https://gohugo.io/getting-started/quick-start/)：官方文档。
- [如何使用Hugo在GitHub Pages上搭建免费个人网站](https://zhuanlan.zhihu.com/p/37752930)
