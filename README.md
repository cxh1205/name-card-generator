# 🏷️ 名牌生成器 · Tent Card Generator

<div align="center">

**一键生成可折叠三角立式名牌，支持批量打印**

[![Vue](https://img.shields.io/badge/Vue-3.5-4FC08D?logo=vue.js&logoColor=white)](https://vuejs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-6.0-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Vite](https://img.shields.io/badge/Vite-latest-646CFF?logo=vite&logoColor=white)](https://vite.dev/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![GitHub Stars](https://img.shields.io/github/stars/cxh1205/name-card-generator?style=social)](https://github.com/cxh1205/name-card-generator)

[在线使用](#-在线使用) · [功能特性](#-功能特性) · [本地开发](#-本地开发) · [技术架构](#-技术架构)

</div>

---

## 📖 项目简介

**名牌生成器**是一款纯前端 Web 工具，用于快速生成可折叠的三角立式姓名牌。输入名字列表，调整样式，一键打印——每张名牌独立分页，沿中线折叠即可立在桌面上。

适用于会议、宴会、考场、培训等需要大批量制作席卡 / 姓名牌的场合。

### 名牌结构

每张名牌是一张正方形纸张，横向一分为二：

- **上半部分**：名字倒置印刷（折叠后成为背面的正向文字）
- **下半部分**：名字正向印刷（正面视角）
- **第三联（可选）**：底部延长面板，可自定义背景

沿中线折叠后形成三角形立牌，从正面和背面均可看到正向的名字。

---

## 🚀 在线使用

### 方式一：直接访问（推荐）

> 打开 **[GitHub Pages 在线版](https://cxh1205.github.io/name-card-generator/)** 即可使用，无需安装任何软件。

### 方式二：本地运行

```bash
git clone https://github.com/cxh1205/name-card-generator.git
cd name-card-generator
npm install
npm run dev
```

浏览器访问 `http://localhost:5173` 即可使用。

### 打印步骤

1. 左侧输入名字，调整纸张边长、背景、字体等参数
2. 右侧实时预览效果（支持 1/2/3 列预览 + 3D 立牌视图）
3. 点击底部 **「打印」** 按钮或按 `Ctrl + P`
4. 浏览器打印对话框中边距设为 **无**
5. 打印后沿正方形边缘裁剪，沿中线折叠即可

---

## ✨ 功能特性

### 名字管理
- **批量输入**：支持换行、逗号、中文逗号、顿号分隔
- **空格控制字间距**：在名字中插入空格即可调整字符间距
- **实时计数**：侧边栏显示当前名字数量

### 样式定制
- **纸张边长**：60mm ~ 200mm 可调，预览与打印同步
- **背景**：纯色 / 自定义图片（自动居中裁剪为 2:1 比例）
- **第三联（底座）**：可选扩展面板，独立设置背景
- **字体颜色 & 粗细**：支持 Light / Regular / Semi Bold / Bold / Black
- **文字描边**：颜色 + 粗细可调，增强可读性
- **自动撑满**：根据名字字数自动计算最佳字号；也可手动调节

### 预览 & 3D 立牌
- **1/2/3 列网格预览**：响应式布局，适配不同屏幕
- **3D 立牌视图**：实时渲染三角立牌的立体效果
  - 拖拽旋转视角
  - 滚轮缩放
  - 一键回正

### 打印输出
- **独立分页**：每张名牌独占一页
- **自动隐藏**：打印时自动去掉控制面板、分割线等非打印元素
- **精确尺寸**：打印尺寸完全按纸张边长设定

---

## 🛠 本地开发

### 环境要求

- [Node.js](https://nodejs.org/) ≥ 20.19 或 ≥ 22.12
- npm（随 Node.js 一起安装）

### 命令

```bash
# 安装依赖
npm install

# 启动开发服务器（支持热更新）
npm run dev

# 类型检查
npm run type-check

# 生产构建
npm run build

# 预览生产构建
npm run preview
```

### 技术栈

| 类别 | 技术 |
|------|------|
| 框架 | Vue 3（Composition API + `<script setup>`） |
| 语言 | TypeScript |
| 构建 | Vite |
| 样式 | 原生 CSS（CSS Grid · Container Queries · CSS 3D Transforms） |
| 图片处理 | Canvas API（前端裁剪压缩） |
| 部署 | GitHub Pages（GitHub Actions 自动部署） |

---

## 🏗 技术架构

### 整体设计

单页面应用，全部代码集中在 `src/App.vue` —— 无路由、无状态管理库、无子组件。采用**同一 DOM 双端渲染**的策略：屏幕预览使用 CSS Container Queries 实现响应式字号，打印模式使用 `@media print` 切换到绝对毫米尺寸。

### 屏幕 vs 打印双端渲染

| 渲染端 | 字号方案 | 尺寸方案 |
|--------|----------|----------|
| 屏幕预览 | `calc(1cqh * var(--font-pct))` | Grid 自适应 |
| 打印输出 | `var(--print-fs)`（mm 绝对值） | `var(--print-w/h)`（mm 绝对值） |

### 3D 立牌实现

使用纯 CSS 3D Transform 构建三角立牌的立体模型：

- 两片叶片分别绕 X 轴旋转 ±30°，夹角 60°（等边三角形内角）
- `backface-visibility: hidden` + `rotateY(180deg)` 实现正反面内容不同
- 光照渐变伪元素增加立体感和深度
- 拖拽 / 滚轮控制旋转和缩放，无任何 3D 库依赖

---

## 🤝 参与贡献

欢迎提交 Issue 和 Pull Request！

1. Fork 本仓库
2. 创建特性分支：`git checkout -b feature/amazing-feature`
3. 提交更改：`git commit -m 'feat: add amazing feature'`
4. 推送到分支：`git push origin feature/amazing-feature`
5. 提交 Pull Request

---

## ⭐ 支持项目

如果这个工具对你有帮助，欢迎：

- 🌟 **点亮 [Star](https://github.com/cxh1205/name-card-generator)**，让更多人看到
- 🐛 提交 [Issue](https://github.com/cxh1205/name-card-generator/issues) 反馈问题或建议
- 📢 分享给需要制作名牌的朋友

你的 Star 是这个项目持续改进的动力！

---

## 📄 License

本项目基于 [MIT License](LICENSE) 开源，可自由使用、修改和分发。
