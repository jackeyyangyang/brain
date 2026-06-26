---
name: web-artifacts-builder
description: 一套用于创建复杂、多组件 claude.ai HTML artifact 的工具，使用现代前端技术栈（React、Tailwind CSS、shadcn/ui）。适用于需要状态管理、路由或 shadcn/ui 组件的复杂 artifact——不适合简单的单文件 HTML/JSX artifact。
license: Complete terms in LICENSE.txt
allowed-tools: 
disable: true
---

# Web Artifacts Builder

构建强大的前端 claude.ai artifact，遵循以下步骤：

1. 使用 `scripts/init-artifact.sh` 初始化前端项目
2. 通过编辑生成的文件来开发 artifact
3. 用 `scripts/bundle-artifact.sh` 将所有代码打包成单个 HTML 文件
4. 向用户展示 artifact
5.（可选）测试 artifact

**技术栈**：React 18 + TypeScript + Vite + Parcel（打包）+ Tailwind CSS + shadcn/ui

## 设计风格指南

**非常重要**：为避免俗称的"AI slop"，避免使用：
- 过度居中的布局
- 紫色渐变
- 统一的圆角
- Inter 字体

## 快速上手

### 步骤 1：初始化项目

运行初始化脚本，创建新的 React 项目：

```bash
bash scripts/init-artifact.sh <project-name>
cd <project-name>
```

这会创建一个完整配置的项目，包含：
- ✅ React + TypeScript（通过 Vite）
- ✅ Tailwind CSS 3.4.1 + shadcn/ui 主题系统
- ✅ 配置好的路径别名（`@/`）
- ✅ 预装 40+ 个 shadcn/ui 组件
- ✅ 包含所有 Radix UI 依赖
- ✅ 配置好的 Parcel 打包（通过 `.parcelrc`）
- ✅ Node 18+ 兼容性（自动检测并固定 Vite 版本）

### 步骤 2：开发 Artifact

通过编辑生成的文件来构建 artifact。参见下面的**常见开发任务**获取指导。

### 步骤 3：打包为单个 HTML 文件

将 React 应用打包成单个 HTML artifact：

```bash
bash scripts/bundle-artifact.sh
```

这会生成 `bundle.html`——一个自包含的 artifact，所有 JavaScript、CSS 和依赖都内联。这个文件可以直接在 Claude 对话中作为 artifact 分享。

**要求**：项目根目录必须有 `index.html`。

**脚本做的事**：
- 安装打包依赖（parcel、@parcel/config-default、parcel-resolver-tspaths、html-inline）
- 创建带路径别名支持的 `.parcelrc` 配置
- 用 Parcel 构建（无 source maps）
- 用 html-inline 将所有资源内联到单个 HTML

### 步骤 4：向用户分享 Artifact

最后，在对话中向用户分享打包好的 HTML 文件，让他们能作为 artifact 查看。

### 步骤 5：测试 / 可视化 Artifact（可选）

**注意**：这是完全可选的步骤。只有在必要或用户要求时才做。

要用 artifact 测试/可视化，使用可用工具（包括其他 Skills 或内置工具如 Playwright 或 Puppeteer）。总体上，避免提前测试 artifact——因为它会增加从请求到看到完成 artifact 之间的延迟。如果需要，在展示 artifact 之后再测试，如果有问题再处理。

## 常见开发任务

### 添加 shadcn/ui 组件

```bash
npx shadcn@latest add [component-name]
```

参考：https://ui.shadcn.com/docs/components

### 使用 Tailwind CSS

直接使用 Tailwind 类。确保使用 `@/` 路径别名引用项目内部模块。

### 状态管理

对于需要状态的 artifact，使用 React hooks（`useState`、`useReducer` 等）。

### 路由

如果 artifact 需要路由，可以使用 React Router：

```bash
npm install react-router-dom
```

## 调试技巧

- 开发模式下运行 `npm run dev` 查看实时更新
- 检查浏览器控制台错误
- 使用 React DevTools 调试组件状态

## 输出要求

最终 artifact 必须：
- 是单个自包含的 HTML 文件
- 不依赖外部 CDN（除 p5.js 等明确允许的库外）
- 在现代浏览器中正常工作
- 响应式设计，适配不同屏幕尺寸

## 参考

- **shadcn/ui 组件**：https://ui.shadcn.com/docs/components
- **Tailwind CSS 文档**：https://tailwindcss.com/docs
- **React 文档**：https://react.dev
