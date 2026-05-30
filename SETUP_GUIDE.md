# GitHub Pages 个人门户 + 自动部署 Skill 搭建指南

> 将本文档交给你的 AI Agent（Claude Code、Codex 等），它能引导你从零搭建一套完整的 GitHub Pages 个人门户，并创建一个自动部署 Skill —— 之后只需告诉 Agent "部署这个 HTML"，就能自动上线。

---

## 概述

**目标：** 把本地 HTML 页面变成在线可访问的网址，零成本、零后端。

**方案：** GitHub Pages + 项目级 Agent Skill，实现"输入 HTML 文件 → 自动部署上线"。

**最终效果：**

```
https://<你的用户名>.github.io/<仓库名>/              ← 导航首页
https://<你的用户名>.github.io/<仓库名>/project-a/     ← 项目A
https://<你的用户名>.github.io/<仓库名>/project-b/     ← 项目B
```

---

## 第一部分：创建 GitHub 仓库并启用 Pages

### 1.1 注册 GitHub 账号

访问 https://github.com ，点击 **Sign up**，按提示完成注册并验证邮箱。

### 1.2 创建仓库

1. 点击右上角 **+** → **New repository**
2. 仓库名填英文（如 `my-pages`），**必须选 Public**
3. 不要勾选 "Add a README file"（我们要从本地推送）
4. 点击 **Create repository**

### 1.3 在本地初始化项目

在电脑上新建一个空文件夹（如 `~/Documents/my-pages/`），然后告诉 Agent：

> 在 `/你的项目路径/` 下初始化 git 仓库，创建导航首页，并配置远程仓库地址为 `https://github.com/<用户名>/<仓库名>.git`

Agent 会执行：

```bash
cd /你的项目路径
git init
git checkout -b main
git remote add origin https://github.com/<用户名>/<仓库名>.git
```

### 1.4 启用 GitHub Pages

1. 进入仓库页面 → **Settings** → 左侧 **Pages**
2. **Source** 选 **Deploy from a branch**
3. **Branch** 选 `main`，目录选 `/ (root)`，点 **Save**
4. 等待一两分钟，页面地址会显示在设置页顶部

---

## 第二部分：创建导航首页

告诉 Agent：

> 创建导航首页 index.html，要求：暗色主题、支持分类（工作/个人）、支持折叠展开、响应式布局。

参考设计要点：

| 项目 | 说明 |
|------|------|
| 配色 | 深色背景 `#0d0f12`，金色工作分类点 `#e8b86d`，青色个人分类点 `#6db3a5` |
| 字体 | 系统中文字体栈：PingFang SC, Hiragino Sans GB, Microsoft YaHei |
| 布局 | 居中容器 max-width 680px，分类卡片式 |
| 结构 | 每个分类一个 `<section>`，内含 `<ul>` 项目列表 |
| 交互 | 点击分类头展开/折叠，CSS transition 动画 |

**关键 HTML 结构（Agent 需要识别的锚点）：**

```html
<!-- 工作分类 -->
<section class="category" data-category="work">
  <ul class="project-list" id="list-work">
    <!-- Agent 在此插入项目条目 -->
  </ul>
  <div class="empty-hint" id="empty-work">暂无项目</div>
</section>

<!-- 个人分类 -->
<section class="category" data-category="personal">
  <ul class="project-list" id="list-personal">
    <!-- Agent 在此插入项目条目 -->
  </ul>
  <div class="empty-hint" id="empty-personal">暂无项目</div>
</section>
```

**项目条目格式：**

```html
<li><a href="<slug>/"><span class="icon">📁</span><span class="name">显示名称</span><span class="desc">一行描述</span><span class="arrow">→</span></a></li>
```

---

## 第三部分：创建 deploy-html Skill

这是核心 —— 让 Agent 能自动完成部署流程。

告诉 Agent：

> 在项目的 `.claude/skills/deploy-html/` 下创建一个 SKILL.md，实现 HTML 自动部署到 GitHub Pages 的完整流程。

Skill 需要覆盖以下逻辑：

### 工作流程

```
用户提供 HTML 文件路径
       ↓
收集信息（slug、分类、显示名、描述、图标）
       ↓
创建项目文件夹 → 复制 HTML 为 index.html
       ↓
检测并复制 CSS/JS/图片等资源文件夹
       ↓
更新导航首页对应分类的项目列表
       ↓
更新计数 → 隐藏空状态提示
       ↓
git add → commit → push
       ↓
返回线上访问地址
```

### 收集信息

询问用户以下参数（有默认值的可省略）：

| 参数 | 说明 | 示例 | 默认 |
|------|------|------|------|
| HTML 文件路径 | 源文件位置 | `/path/to/page.html` | 必填 |
| slug | URL 路径标识符 | `my-project` | 必填，自动小写+连字符化 |
| 分类 | `work` 或 `personal` | `work` | `personal` |
| 显示名称 | 导航中显示的文字 | `我的项目` | 从 slug 推导 |
| 描述 | 一行简介 | `这是一个演示项目` | 空 |
| 图标 | emoji | `📁` | `📁` |

### 文件操作

```bash
# 创建项目文件夹
mkdir -p "<项目根目录>/<slug>/"

# 复制 HTML 为 index.html
cp "<源文件>" "<项目根目录>/<slug>/index.html"

# 检测并复制资源文件夹
SOURCE_DIR="$(dirname "<源文件>")"
for dir in css js img assets fonts images; do
  if [ -d "$SOURCE_DIR/$dir" ]; then
    cp -r "$SOURCE_DIR/$dir" "<项目根目录>/<slug>/"
  fi
done
```

### 更新导航首页

Agent 需要编辑 `index.html`：

1. 根据分类找到 `id="list-work"` 或 `id="list-personal"` 的 `<ul>`
2. 在 `</ul>` 前插入项目条目
3. 将对应 `empty-hint` 设为 `style="display:none"`
4. 更新对应 `count-work` 或 `count-personal` 计数

**查重：** 插入前检查 `href="<slug>/"` 是否已存在，如存在则更新而非新增。

### Git 操作

```bash
cd "<项目根目录>"
git add "<slug>/" index.html
git commit -m "deploy: 添加 <显示名称> (<slug>) [<category>]"
git push origin main
```

如果未配置 remote，提示用户先执行：

```bash
git remote add origin https://github.com/<用户名>/<仓库名>.git
git push -u origin main
```

### 部署后清理

部署成功后，删除根目录下的源 HTML 文件（已复制到项目文件夹中）。

### 其他操作

**更新已有项目：** slug 已存在时，替换 index.html 和资源文件，仅在有变化时更新导航条目。Commit 信息：`update: 更新 <显示名称> (<slug>)`

**删除项目：** 删除项目文件夹，从导航中移除对应 `<li>`，更新计数和空状态。Commit 信息：`remove: 移除 <显示名称> (<slug>)`

---

## 第四部分：部署第一个页面

一切就绪后，告诉 Agent：

> 部署 `/path/to/my-page.html`，分类选「工作」

Agent 会自动完成全流程并返回线上地址。

---

## 可选增强

以下是我们项目中的实际优化，作为参考：

### 个人分类隐藏（彩蛋）

将「个人」分类设为默认隐藏，通过特定交互触发显示：

- **触发方式：** 双击副标题中的特定字符（如 `·`）、长按（移动端）、键盘快捷键（`Ctrl+Shift+P`）
- **实现：** CSS `display: none` + JavaScript 双击/长按/键盘事件监听
- **效果：** 访客看不到个人项目，只有你知道如何触发

### 目录结构规范

```
my-pages/
├── index.html                 ← 导航首页
├── .gitignore                 ← 忽略 .DS_Store
├── .claude/
│   ├── settings.local.json    ← 项目设置（权限、语言等）
│   └── skills/
│       └── deploy-html/
│           └── SKILL.md       ← 自动部署 Skill
├── project-a/
│   └── index.html
├── project-b/
│   └── index.html
└── ...
```

### 推荐 settings.local.json

```json
{
  "language": "zh-CN",
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(mkdir *)",
      "Bash(cp *)",
      "Bash(ls *)",
      "Bash(find *)"
    ]
  }
}
```

---

## 总结

| 步骤 | 你做什么 | Agent 做什么 |
|------|---------|------------|
| 1 | 注册 GitHub，创建 Public 仓库 | - |
| 2 | 告诉 Agent 项目路径和仓库地址 | 初始化 git，创建首页，配置 remote |
| 3 | 在 GitHub Settings 中启用 Pages | - |
| 4 | 告诉 Agent "创建 deploy-html skill" | 编写完整的 SKILL.md |
| 5 | 告诉 Agent "部署这个 HTML" | 复制文件 → 更新导航 → 推送 → 返回 URL |

之后每次只需第 5 步，Agent 自动处理其余一切。
