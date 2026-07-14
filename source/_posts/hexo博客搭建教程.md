---
title: Hexo 博客搭建教程
date: 2026-07-13 15:15:49
tags:
  - Hexo
---

## 前言

Hexo 是一款基于 Node.js 的快速、简洁且高效的静态博客框架。它支持 Markdown 解析，一键生成静态站点，并且可以轻松部署到 GitHub Pages 等平台。本文将详细介绍如何从零开始搭建一个 Hexo 博客。

---

## 一、环境准备

### 1.1 安装 Node.js

Hexo 依赖于 Node.js，首先需要安装它。

前往 [Node.js 官网](https://nodejs.org/) 下载 LTS（长期支持）版本并安装。安装完成后，打开终端验证：

``` bash
node -v
npm -v
```

如果看到版本号输出，说明安装成功。

### 1.2 安装 Git

Git 用于版本管理和部署（比如部署到 GitHub Pages）。

前往 [Git 官网](https://git-scm.com/) 下载并安装。安装完成后验证：

``` bash
git --version
```

---

## 二、安装 Hexo

使用 npm 全局安装 Hexo 命令行工具：

``` bash
npm install -g hexo-cli
```

安装完成后验证：

``` bash
hexo version
```

---

## 三、初始化博客

### 3.1 创建博客目录

``` bash
hexo init my-blog
cd my-blog
npm install
```

### 3.2 目录结构

初始化完成后，会生成以下目录结构：

```
my-blog/
├── _config.yml          # 站点配置文件
├── package.json         # 依赖信息
├── scaffolds/           # 模板文件夹
│   ├── draft.md         # 草稿模板
│   ├── page.md          # 页面模板
│   └── post.md          # 文章模板
├── source/              # 资源文件夹
│   └── _posts/          # 文章存放目录
├── themes/              # 主题文件夹
└── node_modules/        # 依赖包
```

### 3.3 本地预览

``` bash
hexo server
```

打开浏览器访问 `http://localhost:4000` 即可预览博客。

---

## 四、站点配置

编辑根目录下的 `_config.yml` 文件，按需修改站点信息：

``` yaml
# Site
title: 我的博客                    # 站点标题
subtitle: '学习与分享'              # 站点副标题
description: '一个记录技术学习的博客'  # 站点描述
keywords: 技术,博客,前端             # 站点关键词
author: 你的名字                    # 作者
language: zh-CN                    # 语言
timezone: Asia/Shanghai            # 时区

# URL
url: https://yourname.github.io   # 站点 URL
permalink: :year/:month/:day/:title/  # 文章永久链接格式
```

---

## 五、写作与发布

### 5.1 创建新文章

``` bash
hexo new "我的第一篇文章"
```

这会在 `source/_posts/` 目录下生成 `我的第一篇文章.md` 文件。

### 5.2 文章 Front-matter

打开生成的 `.md` 文件，顶部有一段 Front-matter 配置：

``` yaml
---
title: 我的第一篇文章
date: 2026-07-13 15:15:49
tags:
  - 教程
categories:
  - 技术
---
```

常用字段说明：

| 字段 | 说明 |
|------|------|
| `title` | 文章标题 |
| `date` | 创建时间 |
| `tags` | 标签，支持多个 |
| `categories` | 分类 |
| `updated` | 更新时间 |

### 5.3 草稿功能

如果文章还没写完，可以先保存为草稿：

``` bash
hexo new draft "未完成的文章"
```

预览草稿：

``` bash
hexo server --draft
```

发布草稿：

``` bash
hexo publish "未完成的文章"
```

### 5.4 Markdown 写作

在 Front-matter 下方使用 Markdown 语法编写正文即可。Hexo 会自动解析并生成 HTML 页面。

### 5.5 生成静态文件

写作完成后，生成静态文件：

``` bash
hexo generate
# 或简写
hexo g
```

### 5.6 清理缓存

如果页面没有按预期更新，可以清理缓存后重新生成：

``` bash
hexo clean
hexo g
```

---

## 六、更换主题

Hexo 有丰富的主题生态。以最受欢迎的 Next 主题为例：

### 6.1 下载主题

``` bash
# 进入博客目录
cd my-blog

# 克隆 Next 主题到 themes 目录
git clone https://github.com/next-theme/hexo-theme-next themes/next
```

### 6.2 启用主题

修改站点 `_config.yml` 中的 `theme` 字段：

``` yaml
theme: next
```

### 6.3 主题配置

每个主题都有独立的配置文件，通常位于 `themes/主题名/_config.yml`。Next 主题的配置文件位于 `themes/next/_config.yml`。

推荐使用 **Alternate Theme Config** 方式：在站点根目录创建 `_config.next.yml` 文件来覆盖主题默认配置，这样主题更新时不会被覆盖。

---

## 七、部署到 GitHub Pages

### 7.1 创建 GitHub 仓库

在 GitHub 上创建一个名为 `你的用户名.github.io` 的仓库。

### 7.2 安装部署插件

``` bash
npm install hexo-deployer-git --save
```

### 7.3 配置部署信息

修改 `_config.yml` 中的 `deploy` 部分：

``` yaml
deploy:
  type: git
  repo: https://github.com/你的用户名/你的用户名.github.io.git
  branch: main
```

### 7.4 执行部署

``` bash
hexo deploy
# 或简写
hexo d
```

也可以一条命令完成生成和部署：

``` bash
hexo g -d
```

部署完成后，访问 `https://你的用户名.github.io` 即可看到你的博客。

---

## 八、创建独立页面

除了文章，还可以创建独立页面，比如「关于」页面：

``` bash
hexo new page about
```

这会在 `source/about/` 下生成 `index.md`，编辑该文件即可。访问路径为 `/about/`。

---

## 九、常用命令速查

| 命令 | 简写 | 说明 |
|------|------|------|
| `hexo init <folder>` | — | 初始化博客 |
| `hexo new <title>` | `hexo n` | 新建文章 |
| `hexo generate` | `hexo g` | 生成静态文件 |
| `hexo server` | `hexo s` | 本地预览 |
| `hexo deploy` | `hexo d` | 部署站点 |
| `hexo clean` | — | 清理缓存 |
| `hexo publish <title>` | `hexo p` | 发布草稿 |
| `hexo list post` | — | 列出所有文章 |

---

## 十、进阶技巧

### 10.1 资源文件夹

在 `_config.yml` 中开启 `post_asset_folder`：

``` yaml
post_asset_folder: true
```

这样创建文章时会同时创建一个同名文件夹，文章引用的图片等资源可以放在里面。

### 10.2 标签页和分类页

创建标签页：

``` bash
hexo new page tags
```

编辑 `source/tags/index.md`，添加：

``` markdown
---
type: "tags"
---
```

分类页同理，将 `type` 改为 `"categories"` 即可。

### 10.3 搜索功能

安装本地搜索插件：

``` bash
npm install hexo-generator-searchdb --save
```

在 `_config.yml` 中添加：

``` yaml
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```

---

## 结语

至此，一个完整的 Hexo 博客就搭建完成了。你可以专注于内容创作，Hexo 会帮你处理好剩下的事情。如果遇到问题，可以查阅 [Hexo 官方文档](https://hexo.io/docs/) 寻求帮助。祝你写作愉快！
