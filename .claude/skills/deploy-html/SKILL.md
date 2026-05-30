---
name: deploy-html
description: 当用户想要部署 HTML 文件到 GitHub Pages 门户、添加新项目页面、或更新 yuanquan-pages 仓库中的已有页面时使用。触发词包括"部署"、"发布"、"添加页面"、"上线"、"更新门户"等。
---

# 部署 HTML 到 GitHub Pages

## 概述

将 HTML 文件复制到 `yuanquan-pages` 仓库中（按文件夹结构组织），自动更新导航首页，然后提交并推送到 GitHub Pages —— 一步完成。

## 项目信息

- **仓库根目录：** `/Users/yuanquan/Documents/my pages/`
- **远程仓库：** GitHub Pages（如未配置则询问用户）
- **目录结构：** 每个项目放在独立文件夹中，入口文件命名为 `index.html`

```
yuanquan-pages/
├── index.html              ← 导航首页（自动维护）
├── project-a/
│   └── index.html          ← 项目页面
├── project-b/
│   └── index.html
└── ...
```

## 工作流程

### 第一步：收集信息

询问用户以下信息：

1. **HTML 文件路径** — 要部署的源文件（必填）
2. **项目标识 (slug)** — URL 友好的文件夹名，如 `my-dashboard`（必填；仅限小写字母、连字符，不含空格）
3. **显示名称** — 导航链接中显示的可读名称（默认从 slug 推导）
4. **描述** — 导航链接的一行描述（默认为空）
5. **图标** — 导航链接的表情符号（默认：📁）

如果用户只提供了 HTML 文件路径，则逐项询问其余信息。

### 第二步：复制文件

```bash
# 创建项目文件夹
mkdir -p "/Users/yuanquan/Documents/my pages/<slug>/"

# 将 HTML 复制为 index.html
cp "<源文件.html>" "/Users/yuanquan/Documents/my pages/<slug>/index.html"
```

**资源文件处理：** 如果源 HTML 所在目录下有 `css/`、`js/`、`img/`、`assets/`、`fonts/`、`images/` 等兄弟文件夹，一并复制到项目文件夹中：

```bash
SOURCE_DIR="$(dirname "<源文件.html>")"
for dir in css js img assets fonts images; do
  if [ -d "$SOURCE_DIR/$dir" ]; then
    cp -r "$SOURCE_DIR/$dir" "/Users/yuanquan/Documents/my pages/<slug>/"
  fi
done
```

**路径修正：** 如果源 HTML 使用相对路径引用本地资源，由于项目文件夹内保持了相同的目录结构，路径无需修改即可正常工作。

### 第三步：更新导航首页

读取 `/Users/yuanquan/Documents/my pages/index.html`，在 `<ul id="projectList">` 内部添加新的 `<li>` 条目。格式：

```html
<li><a href="<slug>/"><span class="icon"><emoji></span><display-name><span class="desc"><description></span></a></li>
```

**规则：**
- 插入到 `id="projectList"` 的 `</ul>` 闭合标签之前
- 如果该项目已存在于列表中，则更新而非重复
- 如果是第一个项目，移除 `empty-hint` 提示区域
- 保持已有条目不变

### 第四步：Git 提交与推送

```bash
cd "/Users/yuanquan/Documents/my pages"
git add "<slug>/" index.html
git commit -m "deploy: 添加 <display-name> (<slug>)"
git push origin main
```

**如果未配置远程仓库，** 告知用户：
> 尚未配置 git remote。当前仓库地址：
> ```
> git remote add origin https://github.com/quany9348-source/yuanquan-pages.git
> git push -u origin main
> ```
> 接着在仓库 Settings → Pages → Source 中选择 `main` 分支、`/ (root)` 目录以启用 GitHub Pages。

### 第五步：确认结果

报告完成情况：
- 创建的项目文件夹和 slug
- 复制的文件列表
- 导航首页已更新
- Commit 哈希和推送状态
- 线上访问地址：`https://quany9348-source.github.io/yuanquan-pages/<slug>/`

## 更新已有项目

如果 slug 已存在，视为**更新操作**：
- 替换已有的 `index.html`（及资源文件，如果提供的话）
- 除非显示名称/描述/图标有变化，否则不修改导航条目
- 提交信息：`update: 更新 <display-name> (<slug>)`

## 删除项目

如果用户要求删除项目：
1. 删除项目文件夹：`rm -rf "/Users/yuanquan/Documents/my pages/<slug>/"`
2. 从导航首页中移除对应的 `<li>`
3. 提交：`remove: 移除 <display-name> (<slug>)`
4. 推送

## 常见错误

| 错误 | 修正 |
|------|------|
| 忘记复制 CSS/JS 资源 | 务必检查源文件旁的 `css/`、`js/`、`img/`、`assets/` 文件夹 |
| HTML 中使用绝对路径（`C:\...`、`file://`） | 提醒用户改用相对路径 |
| 导航条目重复 | 添加前先按 `href="<slug>/"` 查重 |
| 未配置 remote 就推送 | 先执行 `git remote -v` 检查，未配置则引导用户设置 |
| slug 含空格或特殊字符 | 自动转换：`我的页面` → `wo-de-ye-mian`（小写字母+连字符） |
