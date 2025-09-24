# JathonKatu's Blog - Hexo 博客项目

这是一个基于 Hexo 的技术博客项目，专注于 Java、Python、网络安全、分布式架构、AI 等技术领域的分享。

## 🚀 快速开始

### 环境要求

- **Node.js**: >= 14.0.0 (推荐使用 LTS 版本)
- **npm**: >= 6.0.0 (通常随 Node.js 一起安装)
- **Git**: 用于版本控制和主题管理

## 📦 新电脑环境搭建完整指南

### 1. 安装基础环境

#### Windows 系统：
```bash
# 1. 安装 Node.js
# 访问 https://nodejs.org/ 下载并安装 LTS 版本

# 2. 验证安装
node --version
npm --version

# 3. 安装 Git
# 访问 https://git-scm.com/ 下载并安装

# 4. 全局安装 Hexo CLI
npm install -g hexo-cli
```

#### macOS 系统：
```bash
# 使用 Homebrew 安装
brew install node
brew install git
npm install -g hexo-cli
```

#### Linux 系统：
```bash
# Ubuntu/Debian
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt-get install -y nodejs git
npm install -g hexo-cli

# CentOS/RHEL
curl -fsSL https://rpm.nodesource.com/setup_lts.x | sudo bash -
sudo yum install -y nodejs git
npm install -g hexo-cli
```

### 2. 克隆项目

```bash
# 克隆项目到本地
git clone https://github.com/JathonKatu/JathonKatu.github.io.git
cd JathonKatu.github.io
```

### 3. 安装项目依赖

```bash
# 安装所有依赖包
npm install

# 如果遇到网络问题，可以使用国内镜像
npm install --registry=https://registry.npmmirror.com
```

### 4. 启动本地开发服务器

```bash
# 生成静态文件
npm run build

# 启动本地服务器
npm run server
```

访问 http://localhost:4000 查看博客

## 🛠️ 开发工作流

### 创建新文章

```bash
# 创建新文章
hexo new "文章标题"

# 创建带分类的文章
hexo new post "文章标题"

# 创建草稿
hexo new draft "草稿标题"
```

### 编辑文章

1. 文章位于 `source/_posts/` 目录
2. 使用 Markdown 格式编写
3. 文章头部包含元数据：

```markdown
---
title: 文章标题
date: 2025-09-19 11:30:00
tags: [Java, Spring, 后端]
categories: [技术分享]
---

文章内容...
```

### 本地预览和调试

```bash
# 清理缓存
npm run clean

# 重新生成
npm run build

# 启动服务器（支持热重载）
npm run server

# 或者一键清理并启动
npm run clean && npm run build && npm run server
```

## 📁 项目结构

```
JathonKatu.github.io/
├── _config.yml          # 主配置文件
├── package.json         # 项目依赖配置
├── source/              # 源文件目录
│   ├── _posts/          # 博客文章
│   └── _drafts/         # 草稿文件
├── themes/              # 主题目录
│   └── landscape/       # 当前使用的主题
├── scaffolds/           # 文章模板
├── public/              # 生成的静态文件（自动生成）
└── node_modules/        # 依赖包（自动生成）
```

## ⚙️ 配置说明

### 主要配置文件 `_config.yml`

```yaml
# 网站信息
title: JathonKatu's Blog
subtitle: 技术分享与学习记录
description: 专注于Java、Python、网络安全、分布式架构、AI等技术领域的学习与分享
keywords: Java, Python, 网络安全, 技术博客, 分布式, 架构, AI
author: JathonKatu
language: zh-CN
timezone: Asia/Shanghai

# URL 配置
url: https://jathonkatu.github.io
permalink: :year/:month/:day/:title/

# 主题配置
theme: landscape

# 部署配置
deploy:
  type: git
  repo: https://github.com/JathonKatu/JathonKatu.github.io.git
  branch: master
```

## 🚀 部署到 GitHub Pages

### 1. 配置 Git

```bash
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"
```

### 2. 部署命令

```bash
# 生成并部署
npm run deploy

# 或者分步执行
npm run build
hexo deploy
```

## 📝 常用命令

| 命令 | 说明 |
|------|------|
| `npm run server` | 启动本地服务器 |
| `npm run build` | 生成静态文件 |
| `npm run clean` | 清理缓存和生成文件 |
| `npm run deploy` | 部署到远程仓库 |
| `hexo new "标题"` | 创建新文章 |
| `hexo new page "页面名"` | 创建新页面 |
| `hexo new draft "标题"` | 创建草稿 |
| `hexo publish "草稿名"` | 发布草稿 |

## 🔧 故障排除

### 常见问题

1. **端口被占用**
   ```bash
   # 指定其他端口
   hexo server -p 4001
   ```

2. **依赖安装失败**
   ```bash
   # 清理缓存重新安装
   npm cache clean --force
   rm -rf node_modules
   npm install
   ```

3. **主题显示异常**
   ```bash
   # 重新安装主题
   rm -rf themes/landscape
   git clone https://github.com/hexojs/hexo-theme-landscape.git themes/landscape
   ```

4. **生成文件异常**
   ```bash
   # 清理后重新生成
   npm run clean
   npm run build
   ```

## 🎯 开发建议

### 文章编写规范

1. **文件命名**: 使用英文和连字符，如 `java-concurrency-guide.md`
2. **标签使用**: 合理使用标签，便于分类检索
3. **图片管理**: 图片放在 `source/images/` 目录下
4. **代码高亮**: 使用三个反引号指定语言类型

### 性能优化

1. **图片优化**: 压缩图片大小，使用适当格式
2. **缓存利用**: 合理使用 Hexo 缓存机制
3. **插件选择**: 只安装必要的插件

## 📞 技术支持

如果在搭建或使用过程中遇到问题：

1. 查看 [Hexo 官方文档](https://hexo.io/docs/)
2. 检查 [GitHub Issues](https://github.com/JathonKatu/JathonKatu.github.io/issues)
3. 联系项目维护者

## 📄 许可证

本项目采用 MIT 许可证，详见 [LICENSE](LICENSE) 文件。

---

**Happy Blogging! 🎉**

 // "postinstall": "powershell -Command \"if (!(Test-Path -Path 'themes\\landscape' )) { New-Item -ItemType Directory -Path 'themes\\landscape' -Force }; Copy-Item -Recurse -Path 'node_modules\\hexo-theme-landscape\\*' -Destination 'themes\\landscape'\""
 // "postinstall": "powershell -Command \"if (!(Test-Path -Path 'themes\\landscape' )) { New-Item -ItemType Directory -Path 'themes\\landscape' -Force }; Copy-Item -Recurse -Path 'node_modules\\hexo-theme-landscape\\*' -Destination 'themes\\landscape'\""
