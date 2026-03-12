---
title: Hexo 博客从零搭建教程
date: 2026-03-11 17:30:00
updated: 2026-03-11 17:30:00
tags:
  - Hexo
  - 博客
  - 教程
  - 静态网站
categories:
  - 技术教程
---

## 前言

拥有一个自己的技术博客是程序员的标配。本文将手把手教你用 **Hexo + NexT 主题** 搭建一个简洁优雅的博客，并部署到 GitHub Pages。

**最终效果**：[你的博客地址]

---

## 环境准备

### 1. 安装 Node.js

Hexo 基于 Node.js，需要先安装：

```bash
# 访问 https://nodejs.org 下载 LTS 版本
# 或使用 nvm (推荐)
nvm install --lts
nvm use --lts
```

验证安装：
```bash
node -v
npm -v
```

### 2. 安装 Git

用于代码版本管理和部署：

```bash
# Windows: https://git-scm.com/download/win
# macOS: brew install git
# Linux: sudo apt install git
```

验证：
```bash
git --version
```

---

## 搭建博客

### 方法一：Hexo 官方 starter（推荐新手）

```bash
# 克隆官方 starter 项目
hexo init blog
cd blog
npm install
```

### 方法二：手动初始化

```bash
# 创建空目录
mkdir my-blog
cd my-blog

# 初始化 Hexo
npm init hexo .
```

---

## 安装 NexT 主题

NexT 是 Hexo 最受欢迎的主题之一，简洁优雅，高度可定制。

### 1. 下载主题

```bash
# 在博客根目录执行
git clone https://github.com/next-theme/hexo-theme-next themes/next --depth=1
```

### 2. 启用主题

编辑 `_config.yml`：

```yaml
# 找到这行
theme: landscape

# 改为
theme: next
```

### 3. 主题配置

编辑 `themes/next/_config.yml` 或博客根目录的 `_config.landscape.yml`：

```yaml
# 站点信息
title: 我的技术博客
subtitle: '记录学习与成长'
author: 你的名字

# 菜单配置
menu:
  home: / || fa fa-home
  archives: /archives/ || fa fa-archive
  tags: /tags/ || fa fa-tags
  categories: /categories/ || fa fa-th
  about: /about/ || fa fa-user

# 社交链接
social:
  GitHub: https://github.com/yourname || fab fa-github
  Email: mailto:your@email.com || fa fa-envelope
```

---

## 写第一篇文章

### 创建文章

```bash
hexo new post "我的第一篇博客"
```

文章位于 `source/_posts/我的第一篇博客.md`

### 文章格式

```markdown
---
title: 文章标题
date: 2026-03-11 17:30:00
tags:
  - 标签 1
  - 标签 2
categories:
  - 分类名
---

## 正文开始

这里是文章内容...
```

### 常用 Front-matter

```yaml
title: 标题
date: 2026-03-11 17:30:00      # 发布时间
updated: 2026-03-11 18:00:00   # 更新时间
tags: [标签 1, 标签 2]          # 标签
categories: [分类名]            # 分类
cover: /images/cover.jpg       # 封面图
toc: true                       # 是否显示目录
```

---

## 本地预览

```bash
hexo clean && hexo generate
hexo server
```

访问 `http://localhost:4000` 查看效果

---

## 部署到 GitHub Pages

### 1. 创建 GitHub 仓库

- 仓库名：`你的用户名.github.io`
- 设为公开仓库

### 2. 安装部署插件

```bash
npm install hexo-deployer-git --save
```

### 3. 配置部署

编辑 `_config.yml` 底部：

```yaml
deploy:
  type: git
  repo: https://github.com/你的用户名/你的用户名.github.io.git
  branch: main
```

### 4. 一键部署

```bash
hexo clean && hexo generate && hexo deploy
```

或简写：
```bash
hexo d -g
```

### 5. 访问博客

几分钟后访问：`https://你的用户名.github.io`

---

## 常用命令速查

| 命令 | 说明 |
|------|------|
| `hexo new post "标题"` | 新建文章 |
| `hexo new page "关于"` | 新建页面 |
| `hexo generate` | 生成静态文件 |
| `hexo server` | 本地预览 |
| `hexo deploy` | 部署到远程 |
| `hexo clean` | 清理缓存 |
| `hexo d -g` | 生成 + 部署 |

---

## 进阶配置

### 1. 自定义域名

在 GitHub 仓库添加 `CNAME` 文件：
```
your-domain.com
```

然后在 DNS 服务商处配置 CNAME 记录。

### 2. 添加评论

NexT 支持多种评论系统：

```yaml
# themes/next/_config.yml
livere:
  enable: true
  uid: 你的来必力 UID
```

### 3. 添加搜索

```bash
npm install hexo-generator-searchdb --save
```

配置：
```yaml
# _config.yml
search:
  path: search.xml
  field: post
```

### 4. 添加统计

```yaml
# themes/next/_config.yml
baidu_analytics: 你的百度统计 ID
google_analytics: 你的 GA ID
```

---

## 常见问题

### Q: 部署后页面 404
A: 检查仓库名是否为 `用户名.github.io`，等待几分钟让 GitHub 构建

### Q: 中文乱码
A: 确保文件编码为 UTF-8，编辑器保存时选择 UTF-8

### Q: 图片不显示
A: 图片放在 `source/images/` 目录，引用时用 `/images/xxx.jpg`

### Q: 主题不生效
A: 检查 `theme: next` 是否拼写正确，运行 `hexo clean` 清理缓存

---

## 结语

至此，你的 Hexo 博客就搭建完成了！

**下一步建议**：

1. ✍️ 写几篇高质量文章
2. 🎨 自定义主题样式
3. 📊 添加访问统计
4. 🔍 配置 SEO 优化
5. 📱 适配移动端

博客是程序员的第二张名片，用心经营，它会给你带来意想不到的收获。

---

> **参考资源**
> - [Hexo 官方文档](https://hexo.io/zh-cn/)
> - [NexT 主题文档](https://theme-next.js.org/)
> - [GitHub Pages 文档](https://pages.github.com/)
