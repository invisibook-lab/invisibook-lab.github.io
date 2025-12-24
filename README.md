# Invisibook

Invisibook Lab 技术博客 - 基于 Hugo 和 Clean White 主题构建

## 本地开发

### 前置要求

- [Hugo](https://gohugo.io/installation/) (扩展版本，Extended version)

### 安装步骤

1. 克隆仓库并初始化子模块（主题）：
```bash
git clone https://github.com/invisibook-lab/invisibook-lab.github.io.git
cd invisibook-lab.github.io
git submodule update --init --recursive
```

2. 启动本地开发服务器：
```bash
hugo server
```

3. 访问 http://localhost:1313 查看网站

### 添加新文章

```bash
hugo new post/文章标题.md
```

然后编辑 `content/post/文章标题.md` 文件。

### 构建网站

```bash
hugo
```

构建后的静态文件会生成在 `public/` 目录中。

## 部署

本项目使用 GitHub Actions 自动构建和部署到 GitHub Pages。

### 设置 GitHub Pages

1. 在 GitHub 仓库的 Settings -> Pages 中：
   - Source 选择 "GitHub Actions"
   - 不需要选择分支

2. 每次推送到 `main` 分支时，GitHub Actions 会自动：
   - 构建 Hugo 网站
   - 部署到 GitHub Pages

### 手动触发部署

你也可以在 GitHub 仓库的 Actions 标签页中手动触发工作流。

## 主题

本博客使用 [Clean White](https://github.com/zhaohuabing/hugo-theme-cleanwhite) 主题。

## 许可证

（根据需要添加许可证信息）

