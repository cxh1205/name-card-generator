<script setup lang="ts">
import { ref, computed } from 'vue'

// 类型定义
interface DragStart {
  x: number
  y: number
  rx: number
  ry: number
}

type StyleMap = Record<string, string>
type CardStyleMap = Record<string, string | number>

// 响应式状态
const namesText = ref('张三\n李 四\n王老五')
const cardSize = ref(120)
const columns = ref(1)
const bgColor = ref('#ffffff')
const fontColor = ref('#1a1a1a')
const fontPct = ref(70)
const fontWeight = ref(700)
const strokeColor = ref('#000000')
const strokePct = ref(0)
const autoFill = ref(true)
const bgImage = ref<string | null>(null)
const imageInput = ref<HTMLInputElement | null>(null)

// 第三联（底部面板）
const showBase = ref(false)
const baseBgColor = ref('#ffffff')
const baseBgImage = ref<string | null>(null)
const baseImageInput = ref<HTMLInputElement | null>(null)

// 3D 立牌
const show3D = ref(false)
const rotateX3D = ref(-25)
const rotateY3D = ref(35)
const zoom3D = ref(0)
const isDragging3D = ref(false)
const dragStart3D = ref<DragStart>({ x: 0, y: 0, rx: 0, ry: 0 })

// 派生计算
const names = computed(() =>
  namesText.value
    .split(/[\n,，、]+/)
    .map(n => n.trim())
    .filter(Boolean)
)

function displayName(name: string): string {
  const escaped = name
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
  return escaped.replace(/ /g, '\u00A0')
}

const maxChars = computed(() =>
  Math.max(1, ...names.value.map(n => n.replace(/ /g, '').length))
)

const fontPctEffective = computed(() => {
  if (autoFill.value) {
    const n = maxChars.value
    return Math.round(Math.min(90, 170 / n))
  }
  return fontPct.value
})

const printFontSize = computed(() => {
  const halfH = cardSize.value / 2
  return `${(halfH * fontPctEffective.value / 100).toFixed(1)}mm`
})

const printStrokeWidth = computed(() => {
  const halfH = cardSize.value / 2
  return `${(halfH * strokePct.value / 100).toFixed(2)}mm`
})

const cardStyle = computed<CardStyleMap>(() => ({
  backgroundColor: bgColor.value,
  '--font-pct': fontPctEffective.value,
  '--stroke-pct': strokePct.value,
  '--stroke-color': strokeColor.value,
  '--print-w': `${cardSize.value}mm`,
  '--print-h': showBase.value ? `${(cardSize.value * 1.5).toFixed(1)}mm` : `${cardSize.value}mm`,
  '--print-fs': printFontSize.value,
  '--print-sw': printStrokeWidth.value,
}))

const halfStyle = computed<StyleMap>(() => {
  const s: StyleMap = {
    fontWeight: String(fontWeight.value),
    color: fontColor.value,
    fontFamily: '"KaiTi", "STKaiti", "楷体", "KaiTi SC", "AR PL UKai CN", serif',
  }
  if (bgImage.value) {
    s.backgroundImage = `url(${bgImage.value})`
    s.backgroundSize = '100% 100%'
    s.backgroundRepeat = 'no-repeat'
  }
  return s
})

const baseStyle = computed<StyleMap>(() => {
  const s: StyleMap = {
    backgroundColor: baseBgColor.value,
  }
  if (baseBgImage.value) {
    s.backgroundImage = `url(${baseBgImage.value})`
    s.backgroundSize = '100% 100%'
    s.backgroundRepeat = 'no-repeat'
  }
  return s
})

const gridStyle = computed<StyleMap>(() => ({
  gridTemplateColumns: `repeat(${columns.value}, minmax(0, 1fr))`,
  gap: '24px',
  padding: '28px',
  width: '100%',
  maxWidth: '960px',
}))

// 图片上传
function triggerUpload(): void { imageInput.value?.click() }

function handleImageUpload(e: Event): void {
  const input = e.target as HTMLInputElement | null
  const file = input?.files?.[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = (ev) => {
    const img = new Image()
    img.onload = () => cropTo21(img)
    img.src = ev.target?.result as string
  }
  reader.readAsDataURL(file)
  input.value = ''
}

function cropTo21(img: HTMLImageElement): void {
  const canvas = document.createElement('canvas')
  canvas.width = 800
  canvas.height = 400
  const srcW = img.naturalWidth
  const srcH = img.naturalHeight
  const srcRatio = srcW / srcH
  let sx: number, sy: number, sw: number, sh: number
  if (srcRatio > 2) {
    sw = srcH * 2; sh = srcH; sx = (srcW - sw) / 2; sy = 0
  } else {
    sw = srcW; sh = srcW / 2; sx = 0; sy = (srcH - sh) / 2
  }
  canvas.getContext('2d')!.drawImage(img, sx, sy, sw, sh, 0, 0, 800, 400)
  bgImage.value = canvas.toDataURL('image/jpeg', 0.9)
}

function removeBgImage(): void { bgImage.value = null }
function doPrint(): void { window.print() }

// 第三联图片上传
function triggerBaseUpload(): void { baseImageInput.value?.click() }

function handleBaseImageUpload(e: Event): void {
  const input = e.target as HTMLInputElement | null
  const file = input?.files?.[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = (ev) => {
    const img = new Image()
    img.onload = () => cropBaseImage(img)
    img.src = ev.target?.result as string
  }
  reader.readAsDataURL(file)
  input.value = ''
}

function cropBaseImage(img: HTMLImageElement): void {
  const canvas = document.createElement('canvas')
  canvas.width = 800
  canvas.height = 400
  const srcW = img.naturalWidth
  const srcH = img.naturalHeight
  const srcRatio = srcW / srcH
  let sx: number, sy: number, sw: number, sh: number
  if (srcRatio > 2) {
    sw = srcH * 2; sh = srcH; sx = (srcW - sw) / 2; sy = 0
  } else {
    sw = srcW; sh = srcW / 2; sx = 0; sy = (srcH - sh) / 2
  }
  canvas.getContext('2d')!.drawImage(img, sx, sy, sw, sh, 0, 0, 800, 400)
  baseBgImage.value = canvas.toDataURL('image/jpeg', 0.9)
}

function removeBaseBgImage(): void { baseBgImage.value = null }

// 3D 立牌计算属性
const displayName3D = computed(() => {
  return names.value.length > 0 ? names.value[0]! : '名字'
})

// 3D 立牌像素几何 —— 固定尺寸，与打印纸张大小无关
const LEAF_BASE = 120
const leafW = computed(() => LEAF_BASE * 1.6)
const leafH = computed(() => LEAF_BASE * 0.8)
const triH = computed(() => leafH.value * Math.cos(Math.PI / 6))
const halfZ = computed(() => leafH.value * Math.sin(Math.PI / 6))

const standeeVars = computed<StyleMap>(() => ({
  '--leaf-w': leafW.value + 'px',
  '--leaf-h': leafH.value + 'px',
  '--tri-h': triH.value.toFixed(1) + 'px',
  '--half-z': halfZ.value.toFixed(1) + 'px',
}))

const fontSize3D = computed(() => {
  return Math.round(leafH.value * fontPctEffective.value / 100) + 'px'
})

const strokeWidth3D = computed(() => {
  return (leafH.value * strokePct.value / 100).toFixed(1) + 'px'
})

const leaf3DContent = computed<StyleMap>(() => {
  const s: StyleMap = {
    backgroundColor: bgColor.value,
    color: fontColor.value,
    fontWeight: String(fontWeight.value),
    fontFamily: '"KaiTi", "STKaiti", "楷体", "KaiTi SC", "AR PL UKai CN", serif',
    fontSize: fontSize3D.value,
    WebkitTextStrokeColor: strokeColor.value,
    WebkitTextStrokeWidth: strokeWidth3D.value,
  }
  if (bgImage.value) {
    s.backgroundImage = `url(${bgImage.value})`
    s.backgroundSize = '100% 100%'
    s.backgroundRepeat = 'no-repeat'
  }
  return s
})

const base3DContent = computed<StyleMap>(() => {
  const s: StyleMap = { backgroundColor: baseBgColor.value }
  if (baseBgImage.value) {
    s.backgroundImage = `url(${baseBgImage.value})`
    s.backgroundSize = '100% 100%'
    s.backgroundRepeat = 'no-repeat'
  }
  return s
})

const camStyle = computed<StyleMap>(() => ({
  transform: `translateZ(${zoom3D.value}px)`,
}))

const stage3DStyle = computed<StyleMap>(() => ({
  transform: `rotateX(${rotateX3D.value}deg) rotateY(${rotateY3D.value}deg)`,
}))

// 3D 交互事件
function pointerDown3D(e: MouseEvent | TouchEvent): void {
  isDragging3D.value = true
  const pt = 'touches' in e ? e.touches[0]! : e
  dragStart3D.value = { x: pt.clientX, y: pt.clientY, rx: rotateX3D.value, ry: rotateY3D.value }
}

function pointerMove3D(e: MouseEvent | TouchEvent): void {
  if (!isDragging3D.value) return
  const pt = 'touches' in e ? e.touches[0] : e
  if (!pt) return
  rotateY3D.value = dragStart3D.value.ry + (pt.clientX - dragStart3D.value.x) * 0.4
  rotateX3D.value = dragStart3D.value.rx - (pt.clientY - dragStart3D.value.y) * 0.4
}

function pointerUp3D(): void {
  isDragging3D.value = false
}

function onWheel3D(e: WheelEvent): void {
  e.preventDefault()
  zoom3D.value = Math.max(-600, Math.min(600, zoom3D.value - e.deltaY * 0.5))
}

function reset3DView(): void {
  rotateX3D.value = -25
  rotateY3D.value = 35
  zoom3D.value = 0
}

// 展开 / 折叠过渡动画（Vue Transition 钩子）
function onSlideBeforeEnter(el: Element): void {
  const htmlEl = el as HTMLElement
  htmlEl.style.height = '0'
  htmlEl.style.opacity = '0'
  htmlEl.style.overflow = 'hidden'
}

function onSlideEnter(el: Element, done: () => void): void {
  const htmlEl = el as HTMLElement
  htmlEl.style.transition = 'height 0.35s cubic-bezier(0.25, 0.46, 0.45, 0.94), opacity 0.35s ease 0.08s'
  void htmlEl.offsetHeight // 强制回流，确保过渡从初始状态开始
  htmlEl.style.height = htmlEl.scrollHeight + 'px'
  htmlEl.style.opacity = '1'
  const onEnd = (e: TransitionEvent) => {
    if (e.target !== htmlEl) return
    htmlEl.removeEventListener('transitionend', onEnd)
    htmlEl.style.height = 'auto'
    htmlEl.style.overflow = ''
    htmlEl.style.transition = ''
    htmlEl.style.opacity = ''
    done()
  }
  htmlEl.addEventListener('transitionend', onEnd)
}

function onSlideBeforeLeave(el: Element): void {
  const htmlEl = el as HTMLElement
  htmlEl.style.height = htmlEl.offsetHeight + 'px'
  htmlEl.style.overflow = 'hidden'
  void htmlEl.offsetHeight // 强制回流，将显式高度写入渲染树
}

function onSlideLeave(el: Element, done: () => void): void {
  const htmlEl = el as HTMLElement
  htmlEl.style.transition = 'height 0.3s cubic-bezier(0.25, 0.46, 0.45, 0.94), opacity 0.22s ease, margin-top 0.3s cubic-bezier(0.25, 0.46, 0.45, 0.94), padding-top 0.3s cubic-bezier(0.25, 0.46, 0.45, 0.94)'
  void htmlEl.offsetHeight // 强制回流，触发收起动画
  htmlEl.style.height = '0'
  htmlEl.style.opacity = '0'
  htmlEl.style.marginTop = '0'
  htmlEl.style.paddingTop = '0'
  const onEnd = (e: TransitionEvent) => {
    if (e.target !== htmlEl) return
    htmlEl.removeEventListener('transitionend', onEnd)
    done()
  }
  htmlEl.addEventListener('transitionend', onEnd)
}
</script>

<template>
  <div class="app">
    <!-- 左侧边栏 -->
    <aside class="sidebar no-print">
      <a class="sidebar-header" href="https://github.com/cxh1205/name-card-generator" target="_blank" rel="noopener" title="查看开源代码">
        <svg width="22" height="22" viewBox="0 0 24 24" fill="currentColor" stroke="none"><path d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0 0 24 12c0-6.63-5.37-12-12-12z"/></svg>
        <span class="logo-text-wrap">
          <span class="logo-text-default">名牌生成器</span>
          <span class="logo-text-hover">查看开源代码</span>
        </span>
      </a>

      <!-- 名字输入 -->
      <div class="panel">
        <div class="panel-label">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/><path d="M23 21v-2a4 4 0 0 0-3-3.87"/><path d="M16 3.13a4 4 0 0 1 0 7.75"/></svg>
          名字输入
          <span class="badge" v-if="names.length">{{ names.length }}</span>
        </div>
        <textarea
          v-model="namesText"
          placeholder="每行一个名字，用空格控制字间距&#10;例如：张  三"
          rows="10"
          spellcheck="false"
          class="name-input"
        ></textarea>
        <div class="hint">支持换行、逗号、顿号分隔</div>
      </div>

      <!-- 纸张边长 -->
      <div class="panel">
        <div class="panel-label">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polygon points="12 2 22 8.5 22 15.5 12 22 2 15.5 2 8.5"/></svg>
          纸张边长
          <span class="val">{{ cardSize }} mm</span>
        </div>
        <div class="hint" style="margin-bottom:4px">屏幕预览与打印均使用此尺寸</div>
        <div class="slider-row">
          <span class="slider-end">60</span>
          <input type="range" v-model.number="cardSize" min="60" max="200" step="5" />
          <span class="slider-end">200</span>
        </div>
      </div>

      <!-- 背景设置 -->
      <div class="panel">
        <div class="panel-label">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 20h9"/><path d="M16.5 3.5a2.121 2.121 0 0 1 3 3L7 19l-4 1 1-4L16.5 3.5z"/></svg>
          背景
        </div>
        <div class="bg-row">
          <div v-if="bgImage" class="img-select" @click="removeBgImage">
            <img :src="bgImage" class="img-thumb-lg" />
            <div class="img-delete-overlay">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="3 6 5 6 21 6"/><path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"/></svg>
              <span>删除</span>
            </div>
          </div>
          <div v-else class="color-swatch">
            <input type="color" v-model="bgColor" class="color-input" />
            <span class="color-hex">{{ bgColor }}</span>
          </div>
          <button class="upload-btn" @click="triggerUpload" :class="{ 'has-image': bgImage }">
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8.5" cy="8.5" r="1.5"/><polyline points="21 15 16 10 5 21"/></svg>
            {{ bgImage ? '更换图片' : '上传图片' }}
          </button>
          <input ref="imageInput" type="file" accept="image/*" class="hidden-input" @change="handleImageUpload" />
        </div>
        <div class="hint" v-if="bgImage">图片已自动裁剪为 2:1 填充半张卡</div>
      </div>

      <!-- 第三联（底部面板） -->
      <div class="panel">
        <div class="panel-label panel-label-clickable" @click="showBase = !showBase">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><line x1="3" y1="9" x2="21" y2="9"/><line x1="3" y1="15" x2="21" y2="15"/></svg>
          第三联（底部）
          <span class="toggle-sw" style="margin-left:auto" @click.stop="showBase = !showBase">
            <span class="toggle-track" :class="{ active: showBase }"><span class="toggle-thumb"></span></span>
          </span>
        </div>
        <Transition
          @before-enter="onSlideBeforeEnter"
          @enter="onSlideEnter"
          @before-leave="onSlideBeforeLeave"
          @leave="onSlideLeave"
        >
          <div v-if="showBase" class="expand-inner" style="padding-top:8px;display:flex;flex-direction:column;gap:8px">
            <div class="bg-row">
              <div v-if="baseBgImage" class="img-select" @click="removeBaseBgImage">
                <img :src="baseBgImage" class="img-thumb-lg" />
                <div class="img-delete-overlay">
                  <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="3 6 5 6 21 6"/><path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"/></svg>
                  <span>删除</span>
                </div>
              </div>
              <div v-else class="color-swatch">
                <input type="color" v-model="baseBgColor" class="color-input" />
                <span class="color-hex">{{ baseBgColor }}</span>
              </div>
              <button class="upload-btn" @click="triggerBaseUpload" :class="{ 'has-image': baseBgImage }">
                <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8.5" cy="8.5" r="1.5"/><polyline points="21 15 16 10 5 21"/></svg>
                {{ baseBgImage ? '更换图片' : '上传图片' }}
              </button>
              <input ref="baseImageInput" type="file" accept="image/*" class="hidden-input" @change="handleBaseImageUpload" />
            </div>
            <div class="hint" v-if="baseBgImage">图片已自动裁剪为 2:1 填充底部</div>
          </div>
        </Transition>
      </div>

      <!-- 字体设置 -->
      <div class="panel">
        <div class="panel-label">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="4 7 4 4 20 4 20 7"/><line x1="9" y1="20" x2="15" y2="20"/><line x1="12" y1="4" x2="12" y2="20"/></svg>
          字体
        </div>
        <div class="font-grid">
          <div class="font-item">
            <span class="font-item-label">颜色</span>
            <input type="color" v-model="fontColor" class="color-input small" />
          </div>
          <div class="font-item">
            <span class="font-item-label">粗细</span>
            <select v-model.number="fontWeight" class="select">
              <option :value="300">Light</option>
              <option :value="400">Regular</option>
              <option :value="600">Semi Bold</option>
              <option :value="700">Bold</option>
              <option :value="900">Black</option>
            </select>
          </div>
        </div>
        <div class="font-grid" style="margin-top:8px">
          <div class="font-item">
            <span class="font-item-label">描边颜色</span>
            <input type="color" v-model="strokeColor" class="color-input small" />
          </div>
          <div class="font-item">
            <span class="font-item-label">描边粗细 {{ strokePct }}%</span>
            <div class="slider-row">
              <span class="slider-end">0</span>
              <input type="range" v-model.number="strokePct" min="0" max="8" step="0.5" />
              <span class="slider-end">8%</span>
            </div>
          </div>
        </div>
        <div class="toggle-row" @click="autoFill = !autoFill">
          <span class="toggle-label-text">文字自动撑满卡片</span>
          <span class="toggle-sw" @click.stop="autoFill = !autoFill">
            <span class="toggle-track" :class="{ active: autoFill }"><span class="toggle-thumb"></span></span>
          </span>
        </div>
        <Transition
          @before-enter="onSlideBeforeEnter"
          @enter="onSlideEnter"
          @before-leave="onSlideBeforeLeave"
          @leave="onSlideLeave"
        >
          <div v-if="!autoFill" class="expand-inner" style="padding-top:8px;display:flex;flex-direction:column;gap:4px">
            <div class="slider-row">
              <span class="slider-end">10%</span>
              <input type="range" v-model.number="fontPct" min="10" max="95" step="1" />
              <span class="slider-end">95%</span>
            </div>
          </div>
        </Transition>
        <span class="auto-badge">
          {{ autoFill ? '自动' : '手动' }}：字号占半卡高度的 <strong>{{ fontPctEffective }}%</strong>
        </span>
      </div>

      <!-- 打印按钮（吸底固定） -->
      <div class="print-sticky">
        <button class="print-btn" @click="doPrint" :disabled="names.length === 0">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="6 9 6 2 18 2 18 9"/><path d="M6 12H4a2 2 0 0 0-2 2v5a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2v-5a2 2 0 0 0-2-2h-2"/><rect x="6" y="14" width="12" height="8"/></svg>
          打印 {{ names.length }} 张名牌
        </button>
      </div>
    </aside>

    <!-- 右侧面板 -->
    <div class="right-panel">
      <!-- 列数切换 & 3D 开关 -->
      <div class="col-bar no-print">
        <span class="col-bar-label">预览列数</span>
        <div class="btn-group">
          <button v-for="n in [1,2,3]" :key="n" :class="{ active: columns === n }" @click="columns = n">{{ n }} 列</button>
        </div>
        <div class="col-bar-spacer"></div>
        <span class="toggle-3d" @click="show3D = !show3D">
          <span class="toggle-track" :class="{ active: show3D }"><span class="toggle-thumb"></span></span>
          <span class="toggle-label">3D 立牌</span>
        </span>
      </div>

      <!-- 预览区 -->
      <main class="preview" :class="{ 'preview-split': show3D && names.length > 0 }">
        <!-- 无名字时的空状态提示 -->
        <div v-if="!show3D && names.length === 0" class="empty-state">
          <div class="empty-icon">
            <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="#cbd5e1" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><path d="M12 8v8"/><path d="M8 12h8"/></svg>
          </div>
          <p class="empty-title">在左侧输入名字开始生成名牌</p>
          <p class="empty-desc">名字将自动排版为可折叠的三角形名牌</p>
        </div>

        <!-- 仅卡片预览（3D 关闭时） -->
        <div v-else-if="!show3D" class="cards-grid" :style="gridStyle">
          <div v-for="(name, index) in names" :key="index" class="card" :class="{ 'has-base': showBase }" :style="cardStyle">
            <div class="half half-top" :style="halfStyle">
              <span class="name" v-html="displayName(name)"></span>
            </div>
            <div class="divider"></div>
            <div class="half half-bottom" :style="halfStyle">
              <span class="name" v-html="displayName(name)"></span>
            </div>
            <template v-if="showBase">
              <div class="divider"></div>
              <div class="half half-base" :style="baseStyle"></div>
            </template>
          </div>
        </div>

        <!-- 3D + 卡片分屏预览 -->
        <template v-else-if="names.length > 0">
          <div class="preview-left">
            <div class="cards-grid" :style="gridStyle">
              <div v-for="(name, index) in names" :key="index" class="card" :class="{ 'has-base': showBase }" :style="cardStyle">
                <div class="half half-top" :style="halfStyle">
                  <span class="name" v-html="displayName(name)"></span>
                </div>
                <div class="divider"></div>
                <div class="half half-bottom" :style="halfStyle">
                  <span class="name" v-html="displayName(name)"></span>
                </div>
                <template v-if="showBase">
                  <div class="divider"></div>
                  <div class="half half-base" :style="baseStyle"></div>
                </template>
              </div>
            </div>
          </div>
          <div class="preview-right"
               @mousedown="pointerDown3D"
               @mousemove="pointerMove3D"
               @mouseup="pointerUp3D"
               @mouseleave="pointerUp3D"
               @touchstart.prevent="pointerDown3D"
               @touchmove.prevent="pointerMove3D"
               @touchend="pointerUp3D"
               @wheel.prevent="onWheel3D">
            <button class="reset-3d-btn no-print" @click="reset3DView" title="回正视角">
              <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="1 4 1 10 7 10"/><path d="M3.51 15a9 9 0 1 0 2.13-9.36L1 10"/></svg>
              回正
            </button>
            <div class="standee-scene">
              <div class="standee-cam" :style="camStyle">
                <div class="standee-stage" :style="stage3DStyle" :class="{ dragging: isDragging3D }">
                  <div class="standee" :style="standeeVars">
                  <!-- 前叶片：正面显示名字内容，背面为纯白色 -->
                  <div class="leaf leaf-front">
                    <div class="leaf-face leaf-face-front" :style="leaf3DContent">
                      <span class="standee-name" v-html="displayName(displayName3D)"></span>
                    </div>
                    <div class="leaf-face leaf-face-back leaf-face-white"></div>
                  </div>
                  <!-- 后叶片：正面为纯白色，背面显示名字内容 -->
                  <div class="leaf leaf-back">
                    <div class="leaf-face leaf-face-front leaf-face-white"></div>
                    <div class="leaf-face leaf-face-back" :style="leaf3DContent">
                      <span class="standee-name" v-html="displayName(displayName3D)"></span>
                    </div>
                  </div>
                  <!-- 底板：连接前后叶片底边的水平面 -->
                  <div v-if="showBase" class="leaf leaf-base">
                    <div class="leaf-face leaf-face-front" :style="base3DContent"></div>
                    <div class="leaf-face leaf-face-back leaf-face-white"></div>
                  </div>
                </div>
              </div>
              </div>
              <div class="standee-shadow"></div>
            </div>
            <p class="standee-hint">拖拽旋转 / 滚轮缩放</p>
          </div>
        </template>

        <!-- 仅 3D 立牌全宽显示（无名字时） -->
        <div v-else class="preview-3d-full"
             @mousedown="pointerDown3D"
             @mousemove="pointerMove3D"
             @mouseup="pointerUp3D"
             @mouseleave="pointerUp3D"
             @touchstart.prevent="pointerDown3D"
             @touchmove.prevent="pointerMove3D"
             @touchend="pointerUp3D"
             @wheel.prevent="onWheel3D">
          <button class="reset-3d-btn no-print" @click="reset3DView" title="回正视角">
            <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="1 4 1 10 7 10"/><path d="M3.51 15a9 9 0 1 0 2.13-9.36L1 10"/></svg>
            回正
          </button>
          <div class="standee-scene">
            <div class="standee-cam" :style="camStyle">
              <div class="standee-stage" :style="stage3DStyle" :class="{ dragging: isDragging3D }">
                <div class="standee" :style="standeeVars">
                  <div class="leaf leaf-front">
                    <div class="leaf-face leaf-face-front" :style="leaf3DContent">
                      <span class="standee-name" v-html="displayName(displayName3D)"></span>
                    </div>
                    <div class="leaf-face leaf-face-back leaf-face-white"></div>
                  </div>
                  <div class="leaf leaf-back">
                    <div class="leaf-face leaf-face-front leaf-face-white"></div>
                    <div class="leaf-face leaf-face-back" :style="leaf3DContent">
                      <span class="standee-name" v-html="displayName(displayName3D)"></span>
                    </div>
                  </div>
                  <div v-if="showBase" class="leaf leaf-base">
                    <div class="leaf-face leaf-face-front" :style="base3DContent"></div>
                    <div class="leaf-face leaf-face-back leaf-face-white"></div>
                  </div>
                </div>
              </div>
            </div>
            <div class="standee-shadow"></div>
          </div>
          <p class="standee-hint">拖拽旋转 / 滚轮缩放</p>
        </div>
      </main>
    </div>
  </div>
</template>

<style>
* { margin: 0; padding: 0; box-sizing: border-box; }

/* 滚动条 */
::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}
::-webkit-scrollbar-track {
  background: transparent;
}
::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 3px;
}
::-webkit-scrollbar-thumb:hover {
  background: #94a3b8;
}

:root {
  --slate-50:  #f8fafc;
  --slate-100: #f1f5f9;
  --slate-200: #e2e8f0;
  --slate-300: #cbd5e1;
  --slate-400: #94a3b8;
  --slate-500: #64748b;
  --slate-700: #334155;
  --slate-900: #0f172a;
  --blue-500:  #3b82f6;
  --blue-600:  #2563eb;
  --blue-50:   #eff6ff;
  --radius: 8px;
}

html, body, #app {
  height: 100%;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "PingFang SC", "Microsoft YaHei", sans-serif;
  font-size: 14px;
  color: var(--slate-900);
  background: var(--slate-100);
  -webkit-font-smoothing: antialiased;
}

/* 整体布局 */
.app { display: flex; height: 100vh; overflow: hidden; }

.right-panel {
  flex: 1;
  display: flex; flex-direction: column;
  overflow: hidden;
}

/* 侧边栏 */
.sidebar {
  width: 312px; min-width: 312px;
  background: #fff;
  border-right: 1px solid var(--slate-200);
  display: flex; flex-direction: column; gap: 6px;
  overflow-y: auto;
  padding: 0 16px 0;
  z-index: 10;
}

.sidebar-header {
  position: sticky; top: 0; z-index: 2;
  display: flex; align-items: center; gap: 10px;
  margin: 0 -16px;
  padding: 14px 20px;
  background: #fff;
  border-bottom: 1px solid var(--slate-200);
  text-decoration: none; cursor: pointer;
  transition: background 0.15s;
}
.sidebar-header:hover { background: var(--slate-50); }
.sidebar-header svg { color: #24292f; flex-shrink: 0; }

.logo-text-wrap {
  position: relative;
  font-size: 16px; font-weight: 700;
  letter-spacing: -0.2px;
  white-space: nowrap;
  color: var(--slate-900);
}

.logo-text-default, .logo-text-hover {
  transition: opacity 0.35s ease, filter 0.35s ease;
  white-space: nowrap;
}

.logo-text-default {
  opacity: 1;
  filter: blur(0);
}

.logo-text-hover {
  position: absolute;
  left: 0; top: 0;
  opacity: 0;
  filter: blur(8px);
}

.sidebar-header:hover .logo-text-default {
  opacity: 0;
  filter: blur(8px);
}

.sidebar-header:hover .logo-text-hover {
  opacity: 1;
  filter: blur(0);
}

/* 设置面板 */
.panel {
  background: var(--slate-50);
  border: 1px solid var(--slate-200);
  border-radius: var(--radius);
  padding: 12px 14px;
  display: flex; flex-direction: column;
  position: relative;
}
.panel > * + * { margin-top: 8px; }

.panel-label {
  display: flex; align-items: center; gap: 6px;
  font-size: 12.5px; font-weight: 600; color: var(--slate-700);
  letter-spacing: 0.3px;
}
.panel-label svg { color: var(--slate-400); flex-shrink: 0; }

.val { font-weight: 500; color: var(--blue-600); margin-left: auto; font-size: 12px; }

.badge {
  margin-left: auto;
  background: var(--blue-600); color: #fff;
  font-size: 11px; font-weight: 600;
  padding: 1px 7px; border-radius: 10px;
  min-width: 20px; text-align: center;
}

.hint { font-size: 11.5px; color: var(--slate-400); line-height: 1.4; }

/* 名字输入框 */
.name-input {
  width: 100%; padding: 10px 12px;
  border: 1px solid var(--slate-200); border-radius: 6px;
  font-size: 13.5px; font-family: inherit; line-height: 1.6;
  resize: vertical; background: #fff; white-space: pre-wrap;
  transition: border-color 0.15s, box-shadow 0.15s;
}
.name-input:focus {
  outline: none; border-color: var(--blue-500);
  box-shadow: 0 0 0 3px rgba(59,130,246,0.12);
}
.name-input::placeholder { color: var(--slate-300); }

/* 按钮组 */
.btn-group { display: flex; gap: 1px; background: var(--slate-200); border-radius: 6px; padding: 2px; }
.btn-group button {
  flex: 1; padding: 6px 0;
  border: none; background: transparent; border-radius: 5px;
  font-size: 13px; font-weight: 500; cursor: pointer; color: var(--slate-500);
  transition: all 0.15s;
}
.btn-group button:hover:not(.active) { color: var(--slate-700); background: rgba(255,255,255,0.5); }
.btn-group button.active {
  background: #fff; color: var(--slate-900); font-weight: 600;
  box-shadow: 0 1px 3px rgba(0,0,0,0.08);
}

/* 滑块 */
.slider-row { display: flex; align-items: center; gap: 8px; }
.slider-end { font-size: 11px; color: var(--slate-400); font-weight: 500; flex-shrink: 0; }
input[type="range"] { flex: 1; accent-color: var(--blue-600); height: 4px; }

/* 背景设置 */
.bg-row { display: flex; gap: 8px; align-items: center; }

.color-swatch {
  display: flex; align-items: center; gap: 8px;
  background: #fff; border: 1px solid var(--slate-200);
  border-radius: 6px; padding: 5px 8px; flex: 1;
}

.color-input {
  width: 26px; height: 26px;
  border: none; border-radius: 4px; cursor: pointer; padding: 0; background: none;
}
.color-input.small { width: 24px; height: 24px; }
.color-input::-webkit-color-swatch-wrapper { padding: 0; }
.color-input::-webkit-color-swatch { border-radius: 3px; border: 1px solid rgba(0,0,0,0.1); }

.color-hex {
  font-family: "SF Mono", "Consolas", "Menlo", monospace;
  font-size: 12px; color: var(--slate-500);
}

.upload-btn {
  display: flex; align-items: center; gap: 5px;
  padding: 7px 10px;
  border: 1px dashed var(--slate-300); border-radius: 6px;
  background: #fff; color: var(--slate-500);
  font-size: 12.5px; cursor: pointer;
  white-space: nowrap; transition: all 0.15s; font-family: inherit;
}
.upload-btn:hover { border-color: var(--blue-500); color: var(--blue-600); background: var(--blue-50); }
.upload-btn.has-image { border-style: solid; border-color: var(--slate-300); }

.hidden-input { display: none; }

/* 字体设置 */
.font-grid { display: flex; gap: 8px; }
.font-item { flex: 1; display: flex; flex-direction: column; gap: 4px; }
.font-item-label { font-size: 11px; color: var(--slate-400); font-weight: 500; }

.select {
  padding: 5px 6px; border: 1px solid var(--slate-200);
  border-radius: 5px; font-size: 12px; font-family: inherit;
  color: var(--slate-700); background: #fff; cursor: pointer;
}
.select:focus { outline: none; border-color: var(--blue-500); }

.auto-badge {
  font-size: 11.5px; color: var(--slate-500); margin-top: 2px;
}
.auto-badge strong { color: var(--blue-600); }

/* 打印按钮 */
.print-sticky {
  position: sticky; bottom: 0; background: #fff;
  margin-top: auto;
  padding: 12px 16px 16px; margin-left: -16px; margin-right: -16px;
  border-top: 1px solid var(--slate-200);
}

.print-btn {
  display: flex; align-items: center; justify-content: center; gap: 8px;
  width: 100%; padding: 12px;
  background: var(--blue-600); color: #fff;
  border: none; border-radius: var(--radius);
  font-size: 15px; font-weight: 600; cursor: pointer;
  transition: all 0.15s; font-family: inherit;
}
.print-btn:hover:not(:disabled) {
  background: #1d4ed8; transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(37,99,235,0.35);
}
.print-btn:active:not(:disabled) { transform: translateY(0); }
.print-btn:disabled { background: var(--slate-200); color: var(--slate-400); cursor: not-allowed; box-shadow: none; }

/* 顶部工具栏 */
.col-bar {
  display: flex; align-items: center; gap: 10px;
  padding: 8px 20px;
  background: #fff;
  border-bottom: 1px solid var(--slate-200);
  flex-shrink: 0;
}

.col-bar-label {
  font-size: 13px; font-weight: 600; color: var(--slate-700);
  white-space: nowrap;
}

.col-bar .btn-group button { padding: 5px 16px; }

.col-bar-spacer { flex: 1; }

.toggle-3d,
.toggle-sw {
  display: flex; align-items: center; gap: 8px;
  cursor: pointer; user-select: none;
  font-size: 13px; font-weight: 500; color: var(--slate-700);
  white-space: nowrap;
}

/* 通用开关：所有 toggle 共用此样式 */
.toggle-track {
  width: 36px; height: 20px;
  background: var(--slate-300);
  border-radius: 10px;
  position: relative;
  transition: background 0.2s;
  flex-shrink: 0;
}
.toggle-track.active {
  background: var(--blue-600);
}

.toggle-thumb {
  position: absolute;
  top: 2px; left: 2px;
  width: 16px; height: 16px;
  background: #fff;
  border-radius: 50%;
  transition: transform 0.2s;
  box-shadow: 0 1px 2px rgba(0,0,0,0.15);
}
.toggle-track.active .toggle-thumb {
  transform: translateX(16px);
}

/* 展开 / 折叠由 JS 钩子函数控制过渡 */

/* 可点击的面板标题 */
.panel-label-clickable {
  cursor: pointer;
  transition: background 0.15s;
  border-radius: 6px;
  padding: 3px 6px;
  margin: -3px -6px;
}
.panel-label-clickable:hover {
  background: var(--slate-200);
}

/* 切换行（如字体自动撑满开关） */
.toggle-row {
  display: flex;
  align-items: center;
  cursor: pointer;
  user-select: none;
  padding: 2px 0;
}
.toggle-row .toggle-label-text {
  flex: 1;
  font-size: 13px;
  color: var(--slate-700);
  font-weight: 500;
}

/* 图片缩略图（悬停出现删除按钮） */
.img-select {
  position: relative;
  flex: 1;
  border-radius: 6px;
  overflow: hidden;
}
.img-thumb-lg {
  width: 100%;
  height: 54px;
  object-fit: cover;
  display: block;
  border-radius: 6px;
  border: 1px solid var(--slate-200);
}
.img-delete-overlay {
  position: absolute;
  inset: 0;
  background: rgba(239, 68, 68, 0.78);
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 5px;
  opacity: 0;
  transition: opacity 0.2s;
  border-radius: 6px;
  color: #fff;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  pointer-events: none;
}
.img-select:hover .img-delete-overlay {
  opacity: 1;
  pointer-events: auto;
}

/* 预览区 */
.preview {
  flex: 1;
  overflow: auto;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  background: radial-gradient(circle, #d4d4d8 1px, transparent 1px);
  background-size: 20px 20px;
  background-color: #e8eaed;
}

.empty-state {
  display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  margin-top: 18vh; gap: 10px; text-align: center;
}
.empty-icon { margin-bottom: 8px; opacity: 0.6; }
.empty-title { font-size: 16px; font-weight: 600; color: var(--slate-500); }
.empty-desc { font-size: 13px; color: var(--slate-400); }

.cards-grid { display: grid; align-content: start; }

.card {
  display: flex; flex-direction: column;
  aspect-ratio: 1 / 1;
  width: 100%;
  box-shadow: 0 1px 2px rgba(0,0,0,0.06), 0 4px 16px rgba(0,0,0,0.08);
  border-radius: 2px;
  transition: box-shadow 0.2s;
}
.card.has-base {
  aspect-ratio: 2 / 3;
}
.card:hover { box-shadow: 0 1px 2px rgba(0,0,0,0.08), 0 8px 24px rgba(0,0,0,0.12); }

.half {
  flex: 1;
  display: flex; align-items: center; justify-content: center;
  overflow: hidden; padding: 3mm;
  position: relative;
  container-type: size;
}

.half-top { transform: rotate(180deg); }

.half-base {
  container-type: unset;
}

.divider {
  height: 0; flex-shrink: 0;
  margin: 0 8%;
  border-top: 1px dashed #d0d0d0;
}

.name {
  white-space: pre; line-height: 1; text-align: center;
  font-size: calc(1cqh * var(--font-pct, 70));
  -webkit-text-stroke-color: var(--stroke-color, transparent);
  -webkit-text-stroke-width: calc(1cqh * var(--stroke-pct, 0));
  paint-order: stroke fill;
}

/* 分屏预览布局 */
.preview-split {
  flex-direction: row;
  align-items: stretch;
}

.preview-left {
  flex: 1;
  overflow: auto;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  background: radial-gradient(circle, #d4d4d8 1px, transparent 1px);
  background-size: 20px 20px;
  background-color: #e8eaed;
}

.preview-right {
  flex: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: radial-gradient(circle, #d4d4d8 1px, transparent 1px);
  background-size: 20px 20px;
  background-color: #e8eaed;
  border-left: 1px solid var(--slate-200);
  cursor: grab;
  user-select: none;
  -webkit-user-select: none;
  position: relative;
}
.preview-right:active { cursor: grabbing; }

.preview-3d-full {
  flex: 1;
  align-self: stretch;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: radial-gradient(circle, #d4d4d8 1px, transparent 1px);
  background-size: 20px 20px;
  background-color: #e8eaed;
  cursor: grab;
  user-select: none;
  -webkit-user-select: none;
  position: relative;
}
.preview-3d-full:active { cursor: grabbing; }

/* 3D 立牌 */

/* 3D 场景与舞台 */
.standee-scene {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  perspective: 900px;
  perspective-origin: 50% 40%;
}

.standee-cam {
  transform-style: preserve-3d;
  transition: transform 0.3s ease-out;
}

.standee-stage {
  transform-style: preserve-3d;
  transition: transform 0.3s ease-out;
  will-change: transform;
}
.standee-stage.dragging {
  transition: none;
}

/* 模型容器 */
.standee {
  position: relative;
  transform-style: preserve-3d;
  width: var(--leaf-w);
  height: var(--tri-h);
}

/* 叶片（三角形的两个斜面） */
.leaf {
  position: absolute;
  left: 0; top: 0;
  width: var(--leaf-w);
  height: var(--leaf-h);
  transform-style: preserve-3d;
}

/* 前叶片：向观察者倾斜 30° */
.leaf-front {
  transform-origin: 50% 0%;
  transform: rotateX(30deg);
}

/* 后叶片：远离观察者倾斜 -30°，两叶片夹角 30° + 30° = 60° = 等边三角形内角 */
.leaf-back {
  transform-origin: 50% 0%;
  transform: rotateX(-30deg);
}

/* 底板：水平连接前后叶片底部 */
.leaf-base {
  transform-origin: 50% 0%;
  transform: translateY(var(--tri-h)) translateZ(var(--half-z)) rotateX(-90deg);
}

/* 叶片正反面 */
.leaf-face {
  position: absolute;
  border: #eee 1px solid;
  inset: 0;
  backface-visibility: hidden;
  border-radius: 2px;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

/* 反面：rotateY(180deg) 翻转到另一侧 */
.leaf-face-back {
  transform: rotateY(180deg);
}

/* 内侧为纯白 */
.leaf-face-white {
  background: #fff;
}

/* 光照渐变（伪元素模拟立体光影） */
.leaf-front .leaf-face-front::after {
  content: '';
  position: absolute; inset: 0;
  background: linear-gradient(180deg, rgba(255,255,255,0.18) 0%, transparent 55%);
  pointer-events: none;
}

.leaf-back .leaf-face-back::after {
  content: '';
  position: absolute; inset: 0;
  background: linear-gradient(180deg, rgba(0,0,0,0.08) 0%, transparent 65%);
  pointer-events: none;
}

.leaf-base .leaf-face-front::after {
  content: '';
  position: absolute; inset: 0;
  background: linear-gradient(0deg, rgba(0,0,0,0.04) 0%, transparent 40%);
  pointer-events: none;
}

/* 名字文字 */
.standee-name {
  white-space: pre;
  line-height: 1;
  text-align: center;
  pointer-events: none;
  paint-order: stroke fill;
}

/* 地面阴影 */
.standee-shadow {
  width: 70%;
  max-width: 260px;
  height: 24px;
  margin-top: 12px;
  background: radial-gradient(ellipse at center, rgba(0,0,0,0.18) 0%, transparent 70%);
  border-radius: 50%;
  pointer-events: none;
}

/* 操作提示文字 */
.standee-hint {
  margin-top: 10px;
  font-size: 12px;
  color: var(--slate-400);
  pointer-events: none;
}

/* 回正按钮 */
.reset-3d-btn {
  position: absolute;
  top: 10px; right: 10px;
  display: flex; align-items: center; gap: 4px;
  padding: 5px 10px;
  border: 1px solid var(--slate-200);
  border-radius: 6px;
  background: #fff;
  color: var(--slate-600);
  font-size: 12px; font-family: inherit;
  cursor: pointer; z-index: 5;
  transition: all 0.15s;
}
.reset-3d-btn:hover {
  background: var(--slate-50);
  border-color: var(--slate-300);
  color: var(--slate-900);
}
.reset-3d-btn svg { flex-shrink: 0; }

/* 打印样式 */
@media print {
  @page { size: A4; margin: 0; }

  html, body, #app {
    height: auto !important; background: #fff !important;
    -webkit-print-color-adjust: exact; print-color-adjust: exact;
    overflow: visible !important; margin: 0 !important; padding: 0 !important;
  }

  .app { display: block !important; height: auto !important; overflow: visible !important; }

  .right-panel { display: contents !important; }

  .no-print { display: none !important; }
  .divider { display: none !important; }
  .preview-right, .preview-3d-full, .standee-hint { display: none !important; }

  .preview {
    overflow: visible !important; padding: 0 !important; margin: 0 !important;
    background: #fff !important; display: block !important; width: 100% !important;
  }

  .cards-grid {
    display: block !important; gap: 0 !important;
    padding: 0 !important; margin: 0 !important; max-width: none !important;
  }

  .card {
    display: flex !important; flex-direction: column !important;
    aspect-ratio: auto !important;
    width: var(--print-w) !important; height: var(--print-h) !important;
    box-shadow: none !important; border-radius: 0 !important;
    margin: 4mm auto !important;
    break-before: page !important; page-break-before: always !important;
    break-inside: avoid !important; page-break-inside: avoid !important;
  }
  .card:first-child { break-before: avoid !important; page-break-before: avoid !important; }
  .card:last-child { break-after: avoid !important; page-break-after: avoid !important; }

  .half { container-type: unset !important; }
  .half-base { container-type: unset !important; }
  .name { font-size: var(--print-fs) !important; -webkit-text-stroke-width: var(--print-sw) !important; }
}
</style>
