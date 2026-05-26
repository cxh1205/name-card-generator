<script setup>
import { ref, computed } from 'vue'

// ---- state ----
const namesText = ref('张三\n李 四\n王老五')
const cardSize = ref(120)
const columns = ref(1)
const bgColor = ref('#ffffff')
const fontColor = ref('#1a1a1a')
const fontPct = ref(70)
const fontWeight = ref(700)
const autoFill = ref(true)
const bgImage = ref(null)
const imageInput = ref(null)

// ---- derived ----
const names = computed(() =>
  namesText.value
    .split(/[\n,，、]+/)
    .map(n => n.trim())
    .filter(Boolean)
)

function displayName(name) {
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

const cardStyle = computed(() => ({
  width: `${cardSize.value}mm`,
  height: `${cardSize.value}mm`,
  backgroundColor: bgColor.value,
  '--print-w': `${cardSize.value}mm`,
  '--print-h': `${cardSize.value}mm`,
  '--print-fs': printFontSize.value,
}))

const halfStyle = computed(() => {
  const s = {
    fontWeight: fontWeight.value,
    color: fontColor.value,
    fontFamily: '"KaiTi", "STKaiti", "楷体", "KaiTi SC", "AR PL UKai CN", serif',
    '--font-pct': fontPctEffective.value,
  }
  if (bgImage.value) {
    s.backgroundImage = `url(${bgImage.value})`
    s.backgroundSize = '100% 100%'
    s.backgroundRepeat = 'no-repeat'
  }
  return s
})

const gridStyle = computed(() => ({
  gridTemplateColumns: `repeat(${columns.value}, auto)`,
  gap: '24px',
  justifyContent: 'center',
  padding: '28px',
}))

// ---- image upload ----
function triggerUpload() { imageInput.value?.click() }

function handleImageUpload(e) {
  const file = e.target.files?.[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = (ev) => {
    const img = new Image()
    img.onload = () => cropTo21(img)
    img.src = ev.target.result
  }
  reader.readAsDataURL(file)
  e.target.value = ''
}

function cropTo21(img) {
  const canvas = document.createElement('canvas')
  canvas.width = 800
  canvas.height = 400
  const srcW = img.naturalWidth
  const srcH = img.naturalHeight
  const srcRatio = srcW / srcH
  let sx, sy, sw, sh
  if (srcRatio > 2) {
    sw = srcH * 2; sh = srcH; sx = (srcW - sw) / 2; sy = 0
  } else {
    sw = srcW; sh = srcW / 2; sx = 0; sy = (srcH - sh) / 2
  }
  canvas.getContext('2d').drawImage(img, sx, sy, sw, sh, 0, 0, 800, 400)
  bgImage.value = canvas.toDataURL('image/jpeg', 0.9)
}

function removeBgImage() { bgImage.value = null }
function doPrint() { window.print() }
</script>

<template>
  <div class="app">
    <!-- ==================== SIDEBAR ==================== -->
    <aside class="sidebar no-print">
      <div class="sidebar-header">
        <div class="logo">
          <svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><path d="M12 8v8"/><path d="M8 12h8"/></svg>
          名牌生成器
        </div>
      </div>

      <!-- Name Input -->
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

      <!-- Card size -->
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

      <!-- Background -->
      <div class="panel">
        <div class="panel-label">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 20h9"/><path d="M16.5 3.5a2.121 2.121 0 0 1 3 3L7 19l-4 1 1-4L16.5 3.5z"/></svg>
          背景
        </div>
        <div class="bg-row">
          <div class="color-swatch">
            <input type="color" v-model="bgColor" class="color-input" />
            <span class="color-hex">{{ bgColor }}</span>
          </div>
          <button class="upload-btn" @click="triggerUpload" :class="{ 'has-image': bgImage }">
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8.5" cy="8.5" r="1.5"/><polyline points="21 15 16 10 5 21"/></svg>
            {{ bgImage ? '更换图片' : '上传图片' }}
          </button>
          <input ref="imageInput" type="file" accept="image/*" class="hidden-input" @change="handleImageUpload" />
        </div>
        <div v-if="bgImage" class="img-preview-row">
          <img :src="bgImage" class="img-thumb" />
          <button class="remove-img" @click="removeBgImage" title="移除图片">
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/></svg>
          </button>
        </div>
        <div class="hint" v-if="bgImage">图片已自动裁剪为 2:1 填充半张卡</div>
      </div>

      <!-- Font -->
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
        <div class="font-size-row">
          <label class="auto-check">
            <input type="checkbox" v-model="autoFill" />
            <span>自动撑满卡片</span>
          </label>
          <div v-if="!autoFill" class="slider-row" style="margin-top:6px">
            <span class="slider-end">10%</span>
            <input type="range" v-model.number="fontPct" min="10" max="95" step="1" />
            <span class="slider-end">95%</span>
          </div>
          <span class="auto-badge">
            {{ autoFill ? '自动' : '手动' }}：字号占半卡高度的 <strong>{{ fontPctEffective }}%</strong>
          </span>
        </div>
      </div>

      <!-- Print (sticky) -->
      <div class="print-sticky">
        <button class="print-btn" @click="doPrint" :disabled="names.length === 0">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="6 9 6 2 18 2 18 9"/><path d="M6 12H4a2 2 0 0 0-2 2v5a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2v-5a2 2 0 0 0-2-2h-2"/><rect x="6" y="14" width="12" height="8"/></svg>
          打印 {{ names.length }} 张名牌
        </button>
      </div>
    </aside>

    <!-- ==================== RIGHT PANEL ==================== -->
    <div class="right-panel">
      <!-- column selector -->
      <div class="col-bar no-print">
        <span class="col-bar-label">预览列数</span>
        <div class="btn-group">
          <button v-for="n in [1,2,3]" :key="n" :class="{ active: columns === n }" @click="columns = n">{{ n }} 列</button>
        </div>
      </div>

      <!-- preview -->
      <main class="preview">
        <div v-if="names.length === 0" class="empty-state">
          <div class="empty-icon">
            <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="#cbd5e1" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><path d="M12 8v8"/><path d="M8 12h8"/></svg>
          </div>
          <p class="empty-title">在左侧输入名字开始生成名牌</p>
          <p class="empty-desc">名字将自动排版为可折叠的三角形名牌</p>
        </div>

        <div v-else class="cards-grid" :style="gridStyle">
          <div v-for="(name, index) in names" :key="index" class="card" :style="cardStyle">
            <div class="half half-top" :style="halfStyle">
              <span class="name" v-html="displayName(name)"></span>
            </div>
            <div class="divider"></div>
            <div class="half half-bottom" :style="halfStyle">
              <span class="name" v-html="displayName(name)"></span>
            </div>
          </div>
        </div>
      </main>
    </div>
  </div>
</template>

<style>
* { margin: 0; padding: 0; box-sizing: border-box; }

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

/* ============================================================
   LAYOUT
   ============================================================ */
.app { display: flex; height: 100vh; overflow: hidden; }

.right-panel {
  flex: 1;
  display: flex; flex-direction: column;
  overflow: hidden;
}

/* ============================================================
   SIDEBAR
   ============================================================ */
.sidebar {
  width: 312px; min-width: 312px;
  background: #fff;
  border-right: 1px solid var(--slate-200);
  display: flex; flex-direction: column; gap: 6px;
  overflow-y: auto;
  padding: 0 16px 16px;
  z-index: 10;
}

.sidebar-header {
  position: sticky; top: 0; background: #fff;
  padding: 18px 4px 12px; z-index: 2;
}

.logo {
  display: flex; align-items: center; gap: 10px;
  font-size: 17px; font-weight: 700; color: var(--slate-900);
  letter-spacing: -0.2px;
}
.logo svg { color: var(--blue-600); flex-shrink: 0; }

/* ---- Panels ---- */
.panel {
  background: var(--slate-50);
  border: 1px solid var(--slate-200);
  border-radius: var(--radius);
  padding: 12px 14px;
  display: flex; flex-direction: column; gap: 8px;
}

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

/* ---- Name Input ---- */
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

/* ---- Button Group ---- */
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

/* ---- Slider ---- */
.slider-row { display: flex; align-items: center; gap: 8px; }
.slider-end { font-size: 11px; color: var(--slate-400); font-weight: 500; flex-shrink: 0; }
input[type="range"] { flex: 1; accent-color: var(--blue-600); height: 4px; }

/* ---- Background ---- */
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

.img-preview-row { display: flex; align-items: center; gap: 8px; }
.img-thumb {
  width: 100%; height: 40px;
  object-fit: cover; border-radius: 4px; border: 1px solid var(--slate-200);
}
.remove-img {
  display: flex; align-items: center; justify-content: center;
  width: 28px; height: 28px;
  border: none; background: #fee2e2; color: #ef4444;
  border-radius: 6px; cursor: pointer; flex-shrink: 0;
  transition: background 0.15s;
}
.remove-img:hover { background: #fecaca; }

/* ---- Font ---- */
.font-grid { display: flex; gap: 8px; }
.font-item { flex: 1; display: flex; flex-direction: column; gap: 4px; }
.font-item-label { font-size: 11px; color: var(--slate-400); font-weight: 500; }

.select {
  padding: 5px 6px; border: 1px solid var(--slate-200);
  border-radius: 5px; font-size: 12px; font-family: inherit;
  color: var(--slate-700); background: #fff; cursor: pointer;
}
.select:focus { outline: none; border-color: var(--blue-500); }

.font-size-row { display: flex; flex-direction: column; }
.auto-check {
  display: flex; align-items: center; gap: 7px;
  font-size: 13px; color: var(--slate-700); cursor: pointer; user-select: none;
}
.auto-check input[type="checkbox"] {
  width: 16px; height: 16px; accent-color: var(--blue-600); cursor: pointer;
}
.auto-badge {
  font-size: 11.5px; color: var(--slate-500); margin-top: 2px;
}
.auto-badge strong { color: var(--blue-600); }

/* ---- Print Button ---- */
.print-sticky {
  position: sticky; bottom: 0; background: #fff;
  padding: 12px 0 4px; margin: 0 -16px;
  padding-left: 16px; padding-right: 16px;
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

/* ============================================================
   COLUMN BAR (top of right panel)
   ============================================================ */
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

/* ============================================================
   PREVIEW AREA
   ============================================================ */
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
  box-shadow: 0 1px 2px rgba(0,0,0,0.06), 0 4px 16px rgba(0,0,0,0.08);
  border-radius: 2px;
  transition: box-shadow 0.2s;
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

.divider {
  height: 0; flex-shrink: 0;
  margin: 0 8%;
  border-top: 1px dashed #d0d0d0;
}

.name {
  white-space: pre; line-height: 1; text-align: center;
  font-size: calc(1cqh * var(--font-pct, 70));
}

/* ============================================================
   PRINT
   ============================================================ */
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
  .name { font-size: var(--print-fs) !important; }
}
</style>
