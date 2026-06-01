# CLAUDE.md

这份文件为 Claude Code 在当前仓库中工作时提供指导。请使用中文阅读和理解。

## 重要约定

### 代码注释

**所有代码注释使用自然的中文**，不要使用英文注释或机翻风格的中文。注释应当像真人在说话，简洁、自然、不啰嗦。好的注释解释"为什么这样做"而非"代码在做什么"——后者应该通过良好的命名自解释。

### 代码修改后

- **不要运行 `npm run dev`**，不需要启动开发服务器验证
- **必须运行 `npm run type-check`**，确保 TypeScript 类型检查通过
- 如果 type-check 报错，必须修复后再声称完成

## 可用命令

```bash
npm install              # 安装依赖
npm run dev              # 启动开发服务器（Vite HMR，修改代码后不需要运行）
npm run build            # 生产构建 → dist/
npm run type-check       # TypeScript 类型检查（vue-tsc --build）★ 每次修改代码后必跑
npm run preview          # 预览生产构建
```

项目没有配置测试、Linter 或本地 CI。GitHub Actions（`.github/workflows/deploy.yml`）在 `main`/`master` 分支 push 时自动构建部署到 GitHub Pages。

## 项目架构

Vue 3 单页面应用（Composition API + `<script setup lang="ts">`），Vite 构建。全部代码集中在 `src/App.vue` 一个文件中——无路由、无状态管理库、无子组件。

```
src/
├── App.vue        # 整个应用（脚本 + 模板 + 样式）
├── main.ts         # 入口：createApp(App).mount('#app')
└── env.d.ts        # Vite 客户端类型声明
```

### 页面布局

```
┌──────────────┬──────────────────────────┐
│  侧边栏      │  右侧面板                  │
│  (312px)     │  ┌────────────────────┐  │
│              │  │ 工具栏（1/2/3列切换）│  │
│  卡牌模式    │  ├────────────────────┤  │
│  名字输入    │  │                    │  │
│  字体设置    │  │  预览区             │  │
│  背景设置    │  │  .cards-grid       │  │
│  纸张边长    │  │    .card .card …   │  │
│  第三联(三角立牌)│  │                    │  │
│  打印分界线  │  │                    │  │
│              │  │                    │  │
│  [打印按钮]  │  │                    │  │
│  (吸底固定)  │  │                    │  │
└──────────────┴──────────────────────────┘
```

- **侧边栏**：所有设置面板 + 吸底打印按钮。`overflow-y: auto` 可滚动，打印按钮容器使用 `position: sticky; bottom: 0` 始终可见
- **工具栏**：唯一不在侧边栏的控件——1/2/3 列切换 + 3D 开关，置于右侧面板顶部
- **右侧面板**：`flex column` 包裹工具栏 + 预览区。打印时通过 `display: contents` 融入文档流

### 卡片 DOM 结构

```
.card
  .half.half-top     ← transform: rotate(180deg) 翻转文字和背景图
    span.name
  .divider           ← 虚线折叠辅助线（仅屏幕显示，mx-[8%] 不贯穿）
  .print-fold-line   ← 打印分界线（仅打印显示，贯穿卡片，可自定义线型/颜色）
  .half.half-bottom
    span.name
  （可选）.divider + .print-fold-line + .half.half-base  ← 第三联底部（也带分界线）
```

### 打印分界线

仅三角席卡模式可用（`cardMode === 'tent'`）。侧边栏面板包含折叠式设置：

- `showPrintFoldLine` 开关控制是否打印分界线
- `foldLineType` 线型：实线 / 虚线 / 点线
- `foldLineColor` 颜色选择
- 关闭时：屏幕仍显示不贯穿的虚线提示（`.divider`，`mx-[8%]`），打印不出现分界线
- 开启时：屏幕和打印均显示从左到右贯穿卡片的横线（`.print-fold-line`），线型和颜色自定义；第三联启用时底部也有一条分界线

### 卡牌模式

通过侧边栏底部的按钮组切换两种卡牌模式（`cardMode: 'tent' | 'flat'`）：

**三角席卡（`tent`）：** 双面折叠、等边三角形。卡片为正方形（`aspect-ratio: 1`），有第三联时为 2:3。上下两半分别显示名字，上半翻转 180°，保证折叠后正反两面均正向。

**姓名卡片（`flat`）：** 单面显示、2:1 矩形（`aspect-ratio: 2/1`）。仅一个 `.half` 显示名字和背景，无虚线、无翻转。切换到 flat 模式时自动关闭第三联（`showBase = false`），第三联面板也随之隐藏。

字体、颜色、背景等设置两种模式完全共用。`fontPctEffective` 公式中的 `cardWidth` 在推导中消去，百分比在两种模式下通用。

### 屏幕预览 vs 打印的双端渲染

同一个 DOM 产生两种视觉效果。**核心设计思想：预览用纯百分比、打印算真实 mm**。屏幕端字号只跟卡片宽度成固定比例，与列数（卡片像素大小）无关；打印时再用 `cardSize` 换算绝对 mm 值。

**屏幕预览：**
- 卡片用 CSS Grid `repeat(N, minmax(0, 1fr))` 响应式排列，最大宽度 960px
- 每张 `.card` 为正方形（`aspect-ratio: 1`，有第三联时为 `2:3`，姓名卡片为 `2:1`），宽度填满网格单元格
- 字号用 CSS Container Queries：`.half` 设 `container-type: inline-size`，`.name` 用 `font-size: calc(1cqw * var(--font-scale))`
- `--font-scale` 由 `cardStyle` 预计算（`fontCqw * fontPctEffective / 100`），将比例因子在 JS 中一次算出，CSS 侧仅做一次乘法，杜绝浏览器逐级 calc 浮点积累误差
- `cardSize`（mm）不影响屏幕卡片大小，仅存为 CSS 变量供打印使用
- `.divider` 显示为 `border-top: 1px dashed #d0d0d0` 虚线

**打印（`@media print`）：**
- `@page { size: A4; margin: 0 }` 全出血打印
- 所有 `.no-print` 元素隐藏（侧边栏、工具栏）
- `.right-panel` 通过 `display: contents` 溶解
- `.cards-grid` 改为 `display: block`
- 每张 `.card` 用 `--print-w` / `--print-h` CSS 变量指定绝对 mm 尺寸
- 字号覆盖为 `--print-fs`（mm），计算方式：`(cardSize / 2) * fontPctEffective / 100`
- Container Queries 禁用（`container-type: unset`），让 mm 字号生效
- `.divider` 隐藏
- 每张卡片独立分页：`break-before: page`（第一张除外）

### 字号计算逻辑

`fontPctEffective` = 字号占半张卡片高度的百分比：

- **自动撑满模式**：`min(90, 170 / maxChars)` —— 由 `(cardWidth * 0.85 / n) / (cardWidth / 2) * 100` 推导而来
- **手动模式**：用户滑块值（10% ~ 95%）

屏幕端 `1cqw` = `.half` 宽度的 1%。所有模式下（正方形、2:3 含第三联、2:1 扁平），半卡高度恒为卡片宽度的 50%（正方形 W/2；2:3 时 1.5W/3=0.5W；扁平 0.5W/1=0.5W）。`--font-scale = 50 × fontPct / 100`，一个乘法、无中间舍入，比例与列数、卡片像素大小均无关。

### 背景图片处理流程

1. 文件选择 → `FileReader` → `Image` 解码
2. `cropTo21()` 用 Canvas 居中裁剪为 2:1 比例，渲染为 800×400
3. Canvas 导出 JPEG Data URL → 存入 `bgImage` ref
4. 以 `background-image` + `background-size: 100% 100%` 应用到两个 `.half` 元素
5. `.half-top` 的 180° 旋转同时翻转背景图——折叠后背面观感正确

### 名字输入处理

- 用正则 `/[\n,，、]+/` 分割（换行、英文逗号、中文逗号、顿号）
- HTML 转义（防 XSS），然后将空格替换为 `\u00A0`（不换行空格）
- 通过 `v-html` + `white-space: pre` 渲染，保留空格排版

## 设计理念与视觉风格

### 整体风格

走**简洁实用**路线，不花哨。设计参考 Notion / Linear 等现代工具的设计语言：

- **色彩**：Tailwind Slate 灰阶 + Blue 蓝色系作为强调色，整体低饱和度、高可读性
- **圆角**：统一 `--radius: 8px`，面板、按钮、输入框一致圆角
- **阴影**：只用轻微阴影（`box-shadow` 最多 16px 模糊），避免沉重的投影
- **间距**：6px ~ 28px 的节奏感间距，不拥挤也不疏散
- **字体**：系统字体栈（`-apple-system, PingFang SC, Microsoft YaHei`），名牌本身用楷体系列

### 布局哲学

- **左设右览**：左侧 312px 固定宽度侧边栏集中所有设置，右侧自适应预览
- **控件内聚**：相关联的设置放在同一个 `.panel` 卡片内
- **吸底操作**：打印按钮始终吸底可见，不需要用户滚动寻找
- **即时反馈**：所有滑块、颜色选择器、开关的调整都在右侧实时反映

### 动画风格

- **过渡**：统一 `0.15s ~ 0.35s` 的短过渡，缓动函数用 `cubic-bezier(0.25, 0.46, 0.45, 0.94)`（类似 ease-out）
- **展开折叠**：用 JS 钩子函数手动控制高度动画，避免 CSS 无法从 `auto` 过渡的问题
- **悬停**：按钮 hover 时微微上浮（`translateY(-1px)`）+ 蓝色阴影扩散
- **3D 拖拽**：拖拽时去掉 transition 保证跟手，松手后恢复实现惯性感

## 3D 立牌设计详解

### 数学模型

3D 立牌模拟的是实物三角名牌的几何结构。从侧面看，它是一个等边三角形：

- 前后两片叶片分别从顶边向下倾斜，两片夹角 = 60°（等边三角形的内角）
- 前叶片向观察者倾斜 +30°（离垂直面），后叶片远离观察者倾斜 -30°
- `30° + 30° = 60°` ✓ 符合等边三角形几何

### CSS 3D 变换层级

整个 3D 渲染从外到内有四个关键层级：

```
.standee-scene     → perspective: 900px（投影透视，模拟相机镜头）
  .standee-cam     → translateZ（缩放，相机拉近拉远）
    .standee-stage → rotateX / rotateY（旋转，用户拖拽调整视角）
      .standee     → 模型容器，定义宽高
```

**为什么相机缩放放在 stage 外面：**
`translateZ` 在 stage 的父级上改变相机到模型的距离，等价于缩放。它不影响 stage 自身的旋转轴心，所以缩放和旋转互不干扰。

### 叶片几何

```
.leaf (绝对定位, transform-style: preserve-3d)
  ├── .leaf-face-front (正面)
  └── .leaf-face-back (反面, rotateY(180deg))
```

- **前叶片** (`.leaf-front`)：绕 X 轴旋转 `+30°`，顶端为旋转原点。正面朝观察者
- **后叶片** (`.leaf-back`)：绕 X 轴旋转 `-30°`，顶端为旋转原点。正面背对观察者
- **底板** (`.leaf-base`)：先平移到底边位置，再绕 X 轴旋转 `-90°` 变为水平

### 正反面内容排列

这是整个 3D 效果最精妙的地方——用 `backface-visibility: hidden` 实现双面不同内容：

| 叶片 | 正面内容 | 反面内容 | 视觉效果 |
|------|----------|----------|----------|
| 前叶片 | 名字 + 背景 | 纯白色 | 从正面看：看到名字 |
| 后叶片 | 纯白色 | 名字 + 背景 | 绕到背面看：也看到名字 |

**为什么是对的：**
- 前叶片的正面（有字）朝向观察者 → 正确
- 后叶片的反面（有字）因为 `rotateY(180deg)` 翻转后朝向背面观察者 → 正确
- 两片的内侧（前叶片的反面、后叶片的正面）都是白色 → 站在侧面才能看到，这是合理的内侧

### 光照模拟

每个叶片正面覆盖一个伪元素渐变来模拟光照：

- **前叶片**：`linear-gradient(180deg, rgba(255,255,255,0.18) → transparent)` —— 顶部微亮，模拟正面受光
- **后叶片**：`linear-gradient(180deg, rgba(0,0,0,0.08) → transparent)` —— 顶部微暗，模拟背光面
- **底板**：`linear-gradient(0deg, rgba(0,0,0,0.04) → transparent)` —— 近端微暗

这些渐变很淡，只提供微妙的深度线索，不抢夺内容本身的视觉焦点。

### 像素尺寸计算

3D 立牌的像素尺寸与打印无关，固定为：

```
LEAF_BASE = 120px
leafW = LEAF_BASE × 1.6 = 192px    （叶片宽度）
leafH = LEAF_BASE × 0.8 = 96px     （叶片高度）
triH = leafH × cos(30°) ≈ 83.1px   （三角形总高度）
halfZ = leafH × sin(30°) = 48px    （半底边深度，用于放置底板）
```

这些值作为 CSS 自定义属性传入 `.standee`，驱动所有子元素的尺寸。

### 交互控制

- **拖拽旋转**：鼠标/触摸拖拽 → 修改 `rotateX3D` / `rotateY3D` → 应用到 `.standee-stage`
  - 拖拽灵敏度 0.4（`(delta) * 0.4`），手感偏沉稳
  - 拖拽期间去掉 CSS transition（`.dragging` 类），松手恢复
- **滚轮缩放**：`zoom3D` 范围 -600 ~ 600px，映射到 `translateZ`
- **回正按钮**：重置为默认视角 `(-25°, 35°, 0)`
  - 默认视角略微俯视 + 侧转，三角结构最清晰
- **双端支持**：`MouseEvent` + `TouchEvent` 统一处理，用 `'touches' in e` 区分

### 姓名卡片 3D

```
.standee--flat (height: --leaf-h)
  .leaf.leaf-flat (transform-origin: 50% 100%, rotateX(-8deg))
    .leaf-face-front (名字 + 背景)
    .leaf-face-back (纯白色, rotateY(180deg))
```

- 单叶片直立，底边为旋转轴，微微后仰 8° 模拟卡片站立在桌面
- 宽度和高度与三角牌单片叶片相同（192×96px，2:1 比例）
- 前后两面用 `backface-visibility: hidden` 区分内容
- 光照渐变与前叶片相同（正面顶部微亮）

### 地面阴影

`.standee-shadow` 是一个椭圆形径向渐变元素，位于 3D 模型下方，模拟地面投影。它不参与 3D 变换，始终保持在屏幕平面内。
