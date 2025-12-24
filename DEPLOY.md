# GitHub Pages 部署说明

## 首次部署设置

1. **推送代码到 GitHub**
   ```bash
   git add .
   git commit -m "Initial commit: Hugo site with Clean White theme"
   git remote add origin https://github.com/invisibook-lab/invisibook-lab.github.io.git
   git push -u origin main
   ```

2. **在 GitHub 仓库中启用 GitHub Pages**
   - 进入仓库 Settings -> Pages
   - 在 "Source" 部分，选择 "GitHub Actions"
   - 不需要选择分支

3. **等待 GitHub Actions 完成构建**
   - 推送到 main 分支后，GitHub Actions 会自动触发
   - 可以在仓库的 "Actions" 标签页查看构建进度
   - 构建完成后，网站会在几分钟内可用

## 网站地址

部署成功后，网站可以通过以下地址访问：
- https://invisibook-lab.github.io

## 添加内容

### 添加新文章

```bash
hugo new post/我的第一篇文章.md
```

编辑 `content/post/我的第一篇文章.md`，添加内容后提交：

```bash
git add content/post/我的第一篇文章.md
git commit -m "Add: 我的第一篇文章"
git push
```

### 添加静态资源（图片等）

将图片等静态资源放在 `static/img/` 目录下，例如：
- `static/img/logo.png` 可以通过 `/img/logo.png` 访问

### 修改配置

编辑 `hugo.toml` 文件来修改网站配置，包括：
- 网站标题、描述
- 社交链接
- 主题设置等

修改后提交并推送，GitHub Actions 会自动重新构建网站。

## 本地预览

在推送前，可以使用本地服务器预览：

```bash
hugo server
```

然后访问 http://localhost:1313

## 注意事项

- `public/` 目录已被 `.gitignore` 忽略，不需要提交
- 每次推送到 main 分支都会触发自动部署
- GitHub Actions 可能需要几分钟来完成构建和部署

