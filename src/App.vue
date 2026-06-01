<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'
import ToggleSwitch from './components/ToggleSwitch.vue'
import BackgroundPicker from './components/BackgroundPicker.vue'
import ColorPicker from './components/ColorPicker.vue'

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
const cardMode = ref<'tent' | 'flat'>('tent')
const columns = ref(1)
const bgColor = ref('#ffffff')
const fontColor = ref('#1a1a1a')
const fontPct = ref(70)
const fontWeight = ref(700)
const strokeColor = ref('#000000')
const strokePct = ref(0)
const autoFill = ref(true)
const bgImage = ref<string | null>(null)

// 第三联（底部面板）
const showBase = ref(false)
const baseBgColor = ref('#ffffff')
const baseBgImage = ref<string | null>(null)

// 打印分界线（仅三角席卡）
const showPrintFoldLine = ref(false)
const foldLineType = ref<'solid' | 'dashed' | 'dotted'>('solid')
const foldLineColor = ref('#999999')

const foldLinePrintStyle = computed<StyleMap>(() => ({
  borderTopStyle: foldLineType.value,
  borderTopColor: foldLineColor.value,
  borderTopWidth: '1px',
}))

// 切换到姓名卡片时自动关闭第三联
watch(cardMode, (mode) => {
  if (mode === 'flat') showBase.value = false
})

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
  '--print-h': cardMode.value === 'flat'
    ? `${(cardSize.value / 2).toFixed(1)}mm`
    : showBase.value ? `${(cardSize.value * 1.5).toFixed(1)}mm` : `${cardSize.value}mm`,
  '--print-fs': printFontSize.value,
  '--print-sw': printStrokeWidth.value,
  // 屏幕预览：预计算比例因子。半卡高度恒 = 卡片宽度 × 50%
  // （正方形 W/2=0.5W；2:3 时 1.5W/3=0.5W；扁平 2:1 时 0.5W/1=0.5W，全部相等）
  '--font-scale': 50 * fontPctEffective.value / 100,
  '--stroke-scale': 50 * strokePct.value / 100,
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

function doPrint(): void { window.print() }

// 动态设置打印页面尺寸，跟随纸张边长和第三联开关变化
onMounted(() => {
  const styleEl = document.createElement('style')
  styleEl.id = 'print-page-size'
  document.head.appendChild(styleEl)

  const update = () => {
    const w = cardSize.value
    const h = cardMode.value === 'flat'
      ? cardSize.value / 2
      : showBase.value ? cardSize.value * 1.5 : cardSize.value
    styleEl.textContent = `@page { size: ${w}mm ${h}mm; margin: 0; }`
  }
  watch([cardSize, showBase, cardMode], update, { immediate: true })
})

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

// 姓名卡片 3D：单叶片直立，仅需宽高
const flatStandeeVars = computed<StyleMap>(() => ({
  '--leaf-w': leafW.value + 'px',
  '--leaf-h': leafH.value + 'px',
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
  <div class="app flex h-screen overflow-hidden">
    <!-- 左侧边栏 -->
    <aside class="sidebar no-print w-[312px] min-w-[312px] bg-white border-r border-slate-200 flex flex-col gap-1.5 overflow-y-auto px-4 z-10">
      <a class="sidebar-header sticky top-0 z-[2] flex items-center gap-2.5 -mx-4 px-5 py-3.5 bg-white border-b border-slate-200 no-underline cursor-pointer transition-colors duration-150 hover:bg-slate-50" href="https://github.com/cxh1205/name-card-generator" target="_blank" rel="noopener" title="查看开源代码">
        <svg class="text-[#24292f] shrink-0" width="22" height="22" viewBox="0 0 24 24" fill="currentColor" stroke="none"><path d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0 0 24 12c0-6.63-5.37-12-12-12z"/></svg>
        <span class="logo-text-wrap relative text-base font-bold tracking-[-0.2px] whitespace-nowrap text-slate-900">
          <span class="logo-text-default">名牌生成器</span>
          <span class="logo-text-hover">查看开源代码</span>
        </span>
      </a>
      
      <!-- 卡牌模式 -->
      <div class="panel bg-slate-50 border border-slate-200 rounded-lg p-3 flex flex-col relative gap-2">
        <div class="panel-label flex items-center gap-1.5 text-[12.5px] font-semibold text-slate-700 tracking-[0.3px]">
          <svg class="text-slate-400 shrink-0" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><path d="M12 8v8"/><path d="M8 12h8"/></svg>
          卡牌模式
        </div>
        <div class="btn-group flex gap-px bg-slate-200 rounded-md p-0.5">
          <button
            :class="{ 'bg-white text-slate-900 font-semibold shadow-[0_1px_3px_rgba(0,0,0,0.08)]': cardMode === 'tent' }"
            class="grow py-1.5 px-4 border-none bg-transparent rounded-[5px] text-[13px] font-medium cursor-pointer text-slate-500 transition-all duration-150 hover:text-slate-700 hover:bg-white/50 whitespace-nowrap"
            @click="cardMode = 'tent'"
          >三角席卡</button>
          <button
            :class="{ 'bg-white text-slate-900 font-semibold shadow-[0_1px_3px_rgba(0,0,0,0.08)]': cardMode === 'flat' }"
            class="grow py-1.5 px-4 border-none bg-transparent rounded-[5px] text-[13px] font-medium cursor-pointer text-slate-500 transition-all duration-150 hover:text-slate-700 hover:bg-white/50 whitespace-nowrap"
            @click="cardMode = 'flat'"
          >姓名卡片</button>
        </div>
      </div>

      <!-- 名字输入 -->
      <div class="panel bg-slate-50 border border-slate-200 rounded-lg p-3 flex flex-col relative gap-2">
        <div class="panel-label flex items-center gap-1.5 text-[12.5px] font-semibold text-slate-700 tracking-[0.3px]">
          <svg class="text-slate-400 shrink-0" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/><path d="M23 21v-2a4 4 0 0 0-3-3.87"/><path d="M16 3.13a4 4 0 0 1 0 7.75"/></svg>
          名字输入
          <span v-if="names.length" class="ml-auto bg-blue-600 text-white text-[11px] font-semibold px-[7px] py-px rounded-[10px] min-w-5 text-center">{{ names.length }}</span>
        </div>
        <textarea
          v-model="namesText"
          placeholder="每行一个名字，用空格控制字间距&#10;例如：张  三"
          rows="10"
          spellcheck="false"
          class="name-input w-full px-3 py-2.5 border border-slate-200 rounded-md text-[13.5px] font-sans leading-[1.6] resize-y bg-white whitespace-pre-wrap transition-[border-color,box-shadow] duration-150 placeholder:text-slate-300 max-h-[540px] overflow-y-auto"
        ></textarea>
        <div class="text-[11.5px] text-slate-400 leading-[1.4]">支持换行、逗号、顿号分隔</div>
      </div>

      <!-- 字体设置 -->
      <div class="panel bg-slate-50 border border-slate-200 rounded-lg p-3 flex flex-col relative gap-2">
        <div class="panel-label flex items-center gap-1.5 text-[12.5px] font-semibold text-slate-700 tracking-[0.3px]">
          <svg class="text-slate-400 shrink-0" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="4 7 4 4 20 4 20 7"/><line x1="9" y1="20" x2="15" y2="20"/><line x1="12" y1="4" x2="12" y2="20"/></svg>
          字体
        </div>
        <div class="flex gap-2 items-start">
          <div class="flex-1 flex flex-col gap-1 min-w-0">
            <span class="text-[11px] text-slate-400 font-medium">颜色</span>
            <ColorPicker v-model="fontColor" />
          </div>
          <div class="flex-1 flex flex-col gap-1 min-w-0">
            <span class="text-[11px] text-slate-400 font-medium">粗细</span>
            <select v-model.number="fontWeight" class="px-1.5 py-[5px] border border-slate-200 rounded-md text-xs font-sans text-slate-700 bg-white cursor-pointer h-[30px] box-border focus:outline-none focus:border-blue-500 w-full">
              <option :value="300">Light</option>
              <option :value="400">Regular</option>
              <option :value="600">Semi Bold</option>
              <option :value="700">Bold</option>
              <option :value="900">Black</option>
            </select>
          </div>
        </div>
        <div class="flex gap-2 items-start mt-2">
          <div class="flex-1 flex flex-col gap-1 min-w-0">
            <span class="text-[11px] text-slate-400 font-medium">描边颜色</span>
            <ColorPicker v-model="strokeColor" />
          </div>
          <div class="flex-1 flex flex-col gap-1 min-w-0">
            <span class="text-[11px] text-slate-400 font-medium">描边粗细 {{ strokePct }}%</span>
            <div class="flex items-center gap-2 h-[30px]">
              <span class="text-[11px] text-slate-400 font-medium shrink-0">0</span>
              <input type="range" v-model.number="strokePct" min="0" max="8" step="0.5" class="flex-1 accent-blue-600 h-1 min-w-0" />
              <span class="text-[11px] text-slate-400 font-medium shrink-0">8%</span>
            </div>
          </div>
        </div>
        <div class="flex items-center cursor-pointer select-none py-0.5" @click="autoFill = !autoFill">
          <span class="flex-1 text-[13px] text-slate-700 font-medium">文字自动撑满卡片</span>
          <ToggleSwitch v-model="autoFill" />
        </div>
        <Transition
          @before-enter="onSlideBeforeEnter"
          @enter="onSlideEnter"
          @before-leave="onSlideBeforeLeave"
          @leave="onSlideLeave"
        >
          <div v-if="!autoFill" class="expand-inner" style="padding-top:8px;display:flex;flex-direction:column;gap:4px">
            <div class="flex items-center gap-2">
              <span class="text-[11px] text-slate-400 font-medium shrink-0">10%</span>
              <input type="range" v-model.number="fontPct" min="10" max="95" step="1" class="flex-1 accent-blue-600 h-1" />
              <span class="text-[11px] text-slate-400 font-medium shrink-0">95%</span>
            </div>
          </div>
        </Transition>
        <span class="text-[11.5px] text-slate-500 mt-0.5">
          {{ autoFill ? '自动' : '手动' }}：字号占半卡高度的 <strong class="text-blue-600">{{ fontPctEffective }}%</strong>
        </span>
      </div>

      <!-- 背景设置 -->
      <div class="panel bg-slate-50 border border-slate-200 rounded-lg p-3 flex flex-col relative gap-2">
        <div class="panel-label flex items-center gap-1.5 text-[12.5px] font-semibold text-slate-700 tracking-[0.3px]">
          <svg class="text-slate-400 shrink-0" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 20h9"/><path d="M16.5 3.5a2.121 2.121 0 0 1 3 3L7 19l-4 1 1-4L16.5 3.5z"/></svg>
          背景
        </div>
        <BackgroundPicker v-model:color="bgColor" v-model:image="bgImage" hint="图片已自动裁剪为 2:1 填充半张卡" />
      </div>

      <!-- 纸张边长 -->
      <div class="panel bg-slate-50 border border-slate-200 rounded-lg p-3 flex flex-col relative gap-2">
        <div class="panel-label flex items-center gap-1.5 text-[12.5px] font-semibold text-slate-700 tracking-[0.3px]">
          <svg class="text-slate-400 shrink-0" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polygon points="12 2 22 8.5 22 15.5 12 22 2 15.5 2 8.5"/></svg>
          纸张边长
          <span class="font-medium text-blue-600 ml-auto text-xs">{{ cardSize }} mm</span>
        </div>
        <div class="text-[11.5px] text-slate-400 leading-[1.4] mb-1">屏幕预览与打印均使用此尺寸</div>
        <div class="flex items-center gap-2">
          <span class="text-[11px] text-slate-400 font-medium shrink-0">60</span>
          <input type="range" v-model.number="cardSize" min="60" max="200" step="5" class="flex-1 accent-blue-600 h-1" />
          <span class="text-[11px] text-slate-400 font-medium shrink-0">200</span>
        </div>
      </div>

      <!-- 第三联（底部面板） —— 仅三角席卡可用 -->
      <div v-if="cardMode === 'tent'" class="panel bg-slate-50 border border-slate-200 rounded-lg p-3 flex flex-col relative gap-2">
        <div class="panel-label flex items-center gap-1.5 text-[12.5px] font-semibold text-slate-700 tracking-[0.3px] cursor-pointer transition-colors duration-150 rounded-md px-1.5 py-[3px] -mx-1.5 -my-[3px] hover:bg-slate-200" @click="showBase = !showBase">
          <svg class="text-slate-400 shrink-0" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><line x1="3" y1="9" x2="21" y2="9"/><line x1="3" y1="15" x2="21" y2="15"/></svg>
          第三联（底部）
          <ToggleSwitch v-model="showBase" style="margin-left: auto" />
        </div>
        <Transition
          @before-enter="onSlideBeforeEnter"
          @enter="onSlideEnter"
          @before-leave="onSlideBeforeLeave"
          @leave="onSlideLeave"
        >
          <div v-if="showBase" class="expand-inner" style="padding-top:8px;display:flex;flex-direction:column;gap:8px">
            <BackgroundPicker v-model:color="baseBgColor" v-model:image="baseBgImage" hint="图片已自动裁剪为 2:1 填充底部" />
          </div>
        </Transition>
      </div>

      <!-- 打印分界线（仅三角席卡） -->
      <div v-if="cardMode === 'tent'" class="panel bg-slate-50 border border-slate-200 rounded-lg p-3 flex flex-col relative gap-2">
        <div class="panel-label flex items-center gap-1.5 text-[12.5px] font-semibold text-slate-700 tracking-[0.3px] cursor-pointer transition-colors duration-150 rounded-md px-1.5 py-[3px] -mx-1.5 -my-[3px] hover:bg-slate-200" @click="showPrintFoldLine = !showPrintFoldLine">
          <svg class="text-slate-400 shrink-0" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="4" y1="12" x2="20" y2="12"/><polyline points="16 6 20 12 16 18"/></svg>
          打印分界线
          <ToggleSwitch v-model="showPrintFoldLine" style="margin-left: auto" />
        </div>
        <Transition
          @before-enter="onSlideBeforeEnter"
          @enter="onSlideEnter"
          @before-leave="onSlideBeforeLeave"
          @leave="onSlideLeave"
        >
          <div v-if="showPrintFoldLine" class="expand-inner" style="padding-top:8px;display:flex;flex-direction:column;gap:8px">
            <div class="flex gap-2 items-start">
              <div class="flex-1 flex flex-col gap-1 min-w-0">
                <span class="text-[11px] text-slate-400 font-medium">线型</span>
                <div class="btn-group flex gap-px bg-slate-200 rounded-md p-0.5">
                  <button
                    v-for="item in ([['solid','实线'],['dashed','虚线'],['dotted','点线']] as const)"
                    :key="item[0]"
                    :class="{ 'bg-white text-slate-900 font-semibold shadow-[0_1px_3px_rgba(0,0,0,0.08)]': foldLineType === item[0] }"
                    class="grow py-1 px-2.5 border-none bg-transparent rounded-[5px] text-[12px] font-medium cursor-pointer text-slate-500 transition-all duration-150 hover:text-slate-700 hover:bg-white/50 whitespace-nowrap"
                    @click="foldLineType = item[0]"
                  >{{ item[1] }}</button>
                </div>
              </div>
              <div class="flex-1 flex flex-col gap-1 min-w-0">
                <span class="text-[11px] text-slate-400 font-medium">颜色</span>
                <ColorPicker v-model="foldLineColor" />
              </div>
            </div>
          </div>
        </Transition>
      </div>

      <!-- 打印按钮（吸底固定） -->
      <div class="print-sticky sticky bottom-0 bg-white mt-auto -mx-4 px-4 pt-3 pb-4 border-t border-slate-200">
        <button
          class="flex items-center justify-center gap-2 w-full p-3 bg-blue-600 text-white border-none rounded-lg text-[15px] font-semibold cursor-pointer transition-all duration-150 font-sans hover:not-disabled:bg-blue-700 hover:not-disabled:-translate-y-px hover:not-disabled:shadow-[0_4px_12px_rgba(37,99,235,0.35)] active:not-disabled:translate-y-0 disabled:bg-slate-200 disabled:text-slate-400 disabled:cursor-not-allowed disabled:shadow-none"
          @click="doPrint"
          :disabled="names.length === 0"
        >
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="6 9 6 2 18 2 18 9"/><path d="M6 12H4a2 2 0 0 0-2 2v5a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2v-5a2 2 0 0 0-2-2h-2"/><rect x="6" y="14" width="12" height="8"/></svg>
          打印 {{ names.length }} 张名牌
        </button>
      </div>
    </aside>

    <!-- 右侧面板 -->
    <div class="right-panel flex-1 flex flex-col overflow-hidden">
      <!-- 列数切换 & 3D 开关 -->
      <div class="col-bar no-print flex items-center gap-2.5 px-5 py-2 bg-white border-b border-slate-200 shrink-0">
        <span class="text-[13px] font-semibold text-slate-700 whitespace-nowrap">预览列数</span>
        <div class="btn-group flex gap-px bg-slate-200 rounded-md p-0.5">
          <button
            v-for="n in [1,2,3]"
            :key="n"
            :class="{ 'bg-white text-slate-900 font-semibold shadow-[0_1px_3px_rgba(0,0,0,0.08)]': columns === n }"
            class="grow py-1.5 px-4 border-none bg-transparent rounded-[5px] text-[13px] font-medium cursor-pointer text-slate-500 transition-all duration-150 hover:text-slate-700 hover:bg-white/50 whitespace-nowrap"
            @click="columns = n"
          >{{ n }} 列</button>
        </div>
        <div class="flex-1"></div>
        <span class="inline-flex items-center gap-2 cursor-pointer select-none text-[13px] font-medium text-slate-700 whitespace-nowrap" @click="show3D = !show3D">
          <ToggleSwitch v-model="show3D" />
          <span>3D 预览</span>
        </span>
      </div>

      <!-- 预览区 -->
      <main
        class="preview flex-1 overflow-auto flex items-start justify-center"
        :class="{ 'flex-row items-stretch': show3D && names.length > 0 }"
        style="background: radial-gradient(circle, #d4d4d8 1px, transparent 1px) 0 0 / 20px 20px, #e8eaed"
      >
        <!-- 无名字时的空状态提示 -->
        <div v-if="!show3D && names.length === 0" class="flex flex-col items-center justify-center mt-[18vh] gap-2.5 text-center">
          <div class="mb-2 opacity-60">
            <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="#cbd5e1" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><path d="M12 8v8"/><path d="M8 12h8"/></svg>
          </div>
          <p class="text-base font-semibold text-slate-500">在左侧输入名字开始生成名牌</p>
          <p class="text-[13px] text-slate-400">{{ cardMode === 'tent' ? '名字将自动排版为可折叠的三角形名牌' : '名字将自动排版为单面姓名卡片' }}</p>
        </div>

        <!-- 仅卡片预览（3D 关闭时） -->
        <div v-else-if="!show3D" class="cards-grid grid content-start" :style="gridStyle">
          <div
            v-for="(name, index) in names"
            :key="index"
            class="card flex flex-col w-full rounded-sm transition-shadow duration-200 shadow-[0_1px_2px_rgba(0,0,0,0.06),0_4px_16px_rgba(0,0,0,0.08)] hover:shadow-[0_1px_2px_rgba(0,0,0,0.08),0_8px_24px_rgba(0,0,0,0.12)]"
            :class="cardMode === 'flat' ? 'aspect-[2/1]' : (showBase ? 'aspect-[2/3]' : 'aspect-square')"
            :style="cardStyle"
          >
            <template v-if="cardMode === 'tent'">
              <div class="half flex-1 flex items-center justify-center overflow-hidden p-[3mm] relative half-top rotate-180" :style="halfStyle">
                <span class="name whitespace-pre leading-none text-center" v-html="displayName(name)"></span>
              </div>
              <div v-if="!showPrintFoldLine" class="divider h-0 shrink-0 mx-[8%] border-t border-dashed border-[#d0d0d0]"></div>
              <div v-if="showPrintFoldLine" class="print-fold-line h-0 shrink-0 border-t" :style="foldLinePrintStyle"></div>
              <div class="half flex-1 flex items-center justify-center overflow-hidden p-[3mm] relative" :style="halfStyle">
                <span class="name whitespace-pre leading-none text-center" v-html="displayName(name)"></span>
              </div>
              <template v-if="showBase">
                <div v-if="!showPrintFoldLine" class="divider h-0 shrink-0 mx-[8%] border-t border-dashed border-[#d0d0d0]"></div>
                <div v-if="showPrintFoldLine" class="print-fold-line h-0 shrink-0 border-t" :style="foldLinePrintStyle"></div>
                <div class="half flex-1 flex items-center justify-center overflow-hidden p-[3mm] relative" :style="baseStyle"></div>
              </template>
            </template>
            <template v-else>
              <div class="half flex-1 flex items-center justify-center overflow-hidden p-[3mm] relative" :style="halfStyle">
                <span class="name whitespace-pre leading-none text-center" v-html="displayName(name)"></span>
              </div>
            </template>
          </div>
        </div>

        <!-- 3D + 卡片分屏预览 -->
        <template v-else-if="names.length > 0">
          <div class="flex-1 overflow-auto flex items-start justify-center" style="background: radial-gradient(circle, #d4d4d8 1px, transparent 1px) 0 0 / 20px 20px, #e8eaed">
            <div class="cards-grid grid content-start" :style="gridStyle">
              <div
                v-for="(name, index) in names"
                :key="index"
                class="card flex flex-col w-full rounded-sm transition-shadow duration-200 shadow-[0_1px_2px_rgba(0,0,0,0.06),0_4px_16px_rgba(0,0,0,0.08)] hover:shadow-[0_1px_2px_rgba(0,0,0,0.08),0_8px_24px_rgba(0,0,0,0.12)]"
                :class="cardMode === 'flat' ? 'aspect-[2/1]' : (showBase ? 'aspect-[2/3]' : 'aspect-square')"
                :style="cardStyle"
              >
                <template v-if="cardMode === 'tent'">
                  <div class="half flex-1 flex items-center justify-center overflow-hidden p-[3mm] relative half-top rotate-180" :style="halfStyle">
                    <span class="name whitespace-pre leading-none text-center" v-html="displayName(name)"></span>
                  </div>
                  <div v-if="!showPrintFoldLine" class="divider h-0 shrink-0 mx-[8%] border-t border-dashed border-[#d0d0d0]"></div>
                  <div v-if="showPrintFoldLine" class="print-fold-line h-0 shrink-0 border-t" :style="foldLinePrintStyle"></div>
                  <div class="half flex-1 flex items-center justify-center overflow-hidden p-[3mm] relative" :style="halfStyle">
                    <span class="name whitespace-pre leading-none text-center" v-html="displayName(name)"></span>
                  </div>
                  <template v-if="showBase">
                    <div v-if="!showPrintFoldLine" class="divider h-0 shrink-0 mx-[8%] border-t border-dashed border-[#d0d0d0]"></div>
                    <div v-if="showPrintFoldLine" class="print-fold-line h-0 shrink-0 border-t" :style="foldLinePrintStyle"></div>
                    <div class="half flex-1 flex items-center justify-center overflow-hidden p-[3mm] relative" :style="baseStyle"></div>
                  </template>
                </template>
                <template v-else>
                  <div class="half flex-1 flex items-center justify-center overflow-hidden p-[3mm] relative" :style="halfStyle">
                    <span class="name whitespace-pre leading-none text-center" v-html="displayName(name)"></span>
                  </div>
                </template>
              </div>
            </div>
          </div>
          <div class="flex-1 overflow-hidden flex flex-col items-center justify-center border-l border-slate-200 cursor-grab select-none relative active:cursor-grabbing no-print"
               style="background: radial-gradient(circle, #d4d4d8 1px, transparent 1px) 0 0 / 20px 20px, #e8eaed"
               @mousedown="pointerDown3D"
               @mousemove="pointerMove3D"
               @mouseup="pointerUp3D"
               @mouseleave="pointerUp3D"
               @touchstart.prevent="pointerDown3D"
               @touchmove.prevent="pointerMove3D"
               @touchend="pointerUp3D"
               @wheel.prevent="onWheel3D">
            <button class="reset-3d-btn no-print absolute top-2.5 right-2.5 flex items-center gap-1 px-2.5 py-[5px] border border-slate-200 rounded-md bg-white text-slate-600 text-xs font-sans cursor-pointer z-[5] transition-all duration-150 hover:bg-slate-50 hover:border-slate-300 hover:text-slate-900" @click="reset3DView" title="回正视角">
              <svg class="shrink-0" width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="1 4 1 10 7 10"/><path d="M3.51 15a9 9 0 1 0 2.13-9.36L1 10"/></svg>
              回正
            </button>
            <div class="standee-scene flex flex-col items-center justify-center">
              <div class="standee-cam" :style="camStyle">
                <div class="standee-stage" :style="stage3DStyle" :class="{ dragging: isDragging3D }">
                  <!-- 三角席卡 3D：双叶片三角形 -->
                  <template v-if="cardMode === 'tent'">
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
                  </template>
                  <!-- 姓名卡片 3D：单叶片直立 -->
                  <template v-else>
                    <div class="standee standee--flat" :style="flatStandeeVars">
                      <div class="leaf leaf-flat">
                        <div class="leaf-face leaf-face-front" :style="leaf3DContent">
                          <span class="standee-name" v-html="displayName(displayName3D)"></span>
                        </div>
                        <div class="leaf-face leaf-face-back leaf-face-white"></div>
                      </div>
                    </div>
                  </template>
                </div>
              </div>
              <div class="standee-shadow"></div>
            </div>
            <p class="standee-hint mt-2.5 text-xs text-slate-400 pointer-events-none">拖拽旋转 / 滚轮缩放</p>
          </div>
        </template>

        <!-- 仅 3D 立牌全宽显示（无名字时） -->
        <div v-else class="flex-1 self-stretch overflow-hidden flex flex-col items-center justify-center cursor-grab select-none relative active:cursor-grabbing no-print"
             style="background: radial-gradient(circle, #d4d4d8 1px, transparent 1px) 0 0 / 20px 20px, #e8eaed"
             @mousedown="pointerDown3D"
             @mousemove="pointerMove3D"
             @mouseup="pointerUp3D"
             @mouseleave="pointerUp3D"
             @touchstart.prevent="pointerDown3D"
             @touchmove.prevent="pointerMove3D"
             @touchend="pointerUp3D"
             @wheel.prevent="onWheel3D">
          <button class="reset-3d-btn no-print absolute top-2.5 right-2.5 flex items-center gap-1 px-2.5 py-[5px] border border-slate-200 rounded-md bg-white text-slate-600 text-xs font-sans cursor-pointer z-[5] transition-all duration-150 hover:bg-slate-50 hover:border-slate-300 hover:text-slate-900" @click="reset3DView" title="回正视角">
            <svg class="shrink-0" width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="1 4 1 10 7 10"/><path d="M3.51 15a9 9 0 1 0 2.13-9.36L1 10"/></svg>
            回正
          </button>
          <div class="standee-scene flex flex-col items-center justify-center">
            <div class="standee-cam" :style="camStyle">
              <div class="standee-stage" :style="stage3DStyle" :class="{ dragging: isDragging3D }">
                <!-- 三角席卡 3D：双叶片三角形 -->
                <template v-if="cardMode === 'tent'">
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
                </template>
                <!-- 姓名卡片 3D：单叶片直立 -->
                <template v-else>
                  <div class="standee standee--flat" :style="flatStandeeVars">
                    <div class="leaf leaf-flat">
                      <div class="leaf-face leaf-face-front" :style="leaf3DContent">
                        <span class="standee-name" v-html="displayName(displayName3D)"></span>
                      </div>
                      <div class="leaf-face leaf-face-back leaf-face-white"></div>
                    </div>
                  </div>
                </template>
              </div>
            </div>
            <div class="standee-shadow"></div>
          </div>
          <p class="standee-hint mt-2.5 text-xs text-slate-400 pointer-events-none">拖拽旋转 / 滚轮缩放</p>
        </div>
      </main>
    </div>
  </div>
</template>

<style>
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

html, body, #app {
  height: 100%;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "PingFang SC", "Microsoft YaHei", sans-serif;
  -webkit-font-smoothing: antialiased;
  user-select: none;
  -webkit-user-select: none;
}

/* 输入框允许选中 */
input, textarea {
  user-select: text;
  -webkit-user-select: text;
}

/* 标题悬停动画 */
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

/* 名字输入框焦点样式 */
.name-input:focus {
  outline: none;
  border-color: var(--color-blue-500);
  box-shadow: 0 0 0 3px rgba(59,130,246,0.12);
}

/* 打印分界线：v-if 控制显隐，屏幕和打印均可见（贯穿卡片） */

/* 屏幕预览：纯百分比字号，比例因子由 JS 预计算，CSS 仅一次乘法，杜绝浮点误差 */
.half {
  container-type: inline-size;
}
.half-base {
  container-type: unset;
}
.name {
  font-size: calc(1cqw * var(--font-scale, 35));
  -webkit-text-stroke-color: var(--stroke-color, transparent);
  -webkit-text-stroke-width: calc(1cqw * var(--stroke-scale, 0));
  paint-order: stroke fill;
}

/* 3D 立牌 */
.standee-scene {
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
.standee {
  position: relative;
  transform-style: preserve-3d;
  width: var(--leaf-w);
  height: var(--tri-h);
}
.leaf {
  position: absolute;
  left: 0; top: 0;
  width: var(--leaf-w);
  height: var(--leaf-h);
  transform-style: preserve-3d;
}
.leaf-front {
  transform-origin: 50% 0%;
  transform: rotateX(30deg);
}
.leaf-back {
  transform-origin: 50% 0%;
  transform: rotateX(-30deg);
}
.leaf-base {
  transform-origin: 50% 0%;
  transform: translateY(var(--tri-h)) translateZ(var(--half-z)) rotateX(-90deg);
}
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
.leaf-face-back {
  transform: rotateY(180deg);
}
.leaf-face-white {
  background: #fff;
}

/* 光照渐变 */
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

/* 姓名卡片 3D：单叶片直立结构 */
.standee--flat {
  height: var(--leaf-h);
}
.leaf-flat {
  transform-origin: 50% 100%;
  transform: rotateX(-8deg);
}
.leaf-flat .leaf-face-front::after {
  content: '';
  position: absolute; inset: 0;
  background: linear-gradient(180deg, rgba(255,255,255,0.18) 0%, transparent 55%);
  pointer-events: none;
}

.standee-name {
  white-space: pre;
  line-height: 1;
  text-align: center;
  pointer-events: none;
  paint-order: stroke fill;
}
.standee-shadow {
  width: 70%;
  max-width: 260px;
  height: 24px;
  margin-top: 12px;
  background: radial-gradient(ellipse at center, rgba(0,0,0,0.18) 0%, transparent 70%);
  border-radius: 50%;
  pointer-events: none;
}

/* 打印样式 */
@media print {
  html, body, #app {
    height: auto !important; background: #fff !important;
    -webkit-print-color-adjust: exact; print-color-adjust: exact;
    overflow: visible !important; margin: 0 !important; padding: 0 !important;
  }

  .app { display: block !important; height: auto !important; overflow: visible !important; }

  .right-panel { display: contents !important; }

  .no-print { display: none !important; }
  .divider { display: none !important; }
  .print-fold-line { display: block !important; }

  #__vue-devtools-container__ { display: none !important; }

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
    margin: 0 auto !important;
    break-before: page !important; page-break-before: always !important;
    break-inside: avoid !important; page-break-inside: avoid !important;
  }
  .card:first-child { break-before: avoid !important; page-break-before: avoid !important; }
  .card:last-child { break-after: avoid !important; page-break-after: avoid !important; }

  .half { container-type: unset !important; }
  .half-base { container-type: unset !important; }
  .name { font-size: var(--print-fs) !important; -webkit-text-stroke-width: var(--print-sw) !important; }

  /* 3D 立牌不打印 */
  .standee-scene,
  .standee-shadow,
  .standee-hint { display: none !important; }

  /* 3D 分屏时左侧卡片包装器 —— 清除仅用于屏幕预览的样式 */
  .preview > .flex-1:not(.no-print) {
    overflow: visible !important; background: none !important;
  }
}
</style>
