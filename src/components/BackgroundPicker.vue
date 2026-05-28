<script setup lang="ts">
import { ref, computed } from 'vue'
import ColorPicker from './ColorPicker.vue'

const props = defineProps<{
  color: string
  image: string | null
  hint?: string
}>()

const emit = defineEmits<{
  'update:color': [value: string]
  'update:image': [value: string | null]
}>()

const fileInput = ref<HTMLInputElement | null>(null)

const colorModel = computed({
  get: () => props.color,
  set: (v) => emit('update:color', v),
})

function triggerUpload() {
  fileInput.value?.click()
}

function handleUpload(e: Event) {
  const input = e.target as HTMLInputElement | null
  const file = input?.files?.[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = (ev) => {
    const img = new Image()
    img.onload = () => {
      emit('update:image', cropTo21(img))
    }
    img.src = ev.target?.result as string
  }
  reader.readAsDataURL(file)
  input.value = ''
}

function cropTo21(img: HTMLImageElement): string {
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
  return canvas.toDataURL('image/jpeg', 0.9)
}

function removeImage() {
  emit('update:image', null)
}
</script>

<template>
  <div>
    <div class="bg-row">
      <div v-if="image" class="img-select" @click="removeImage">
        <img :src="image" class="img-thumb-lg" />
        <div class="img-delete-overlay">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="3 6 5 6 21 6"/><path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"/></svg>
          <span>删除</span>
        </div>
      </div>
      <ColorPicker v-else v-model="colorModel" style="flex: 1" />
      <button class="upload-btn" @click="triggerUpload" :class="{ 'has-image': image }">
        <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8.5" cy="8.5" r="1.5"/><polyline points="21 15 16 10 5 21"/></svg>
        {{ image ? '更换图片' : '上传图片' }}
      </button>
      <input ref="fileInput" type="file" accept="image/*" class="hidden-input" @change="handleUpload" />
    </div>
    <div v-if="hint && image" class="hint">{{ hint }}</div>
  </div>
</template>

<style scoped>
.bg-row {
  display: flex;
  gap: 8px;
  align-items: center;
}

.upload-btn {
  display: flex;
  align-items: center;
  gap: 5px;
  padding: 7px 10px;
  border: 1px dashed var(--slate-300);
  border-radius: 6px;
  background: #fff;
  color: var(--slate-500);
  font-size: 12.5px;
  cursor: pointer;
  white-space: nowrap;
  transition: all 0.15s;
  font-family: inherit;
}

.upload-btn:hover {
  border-color: var(--blue-500);
  color: var(--blue-600);
  background: var(--blue-50);
}

.upload-btn.has-image {
  border-style: solid;
  border-color: var(--slate-300);
}

.hidden-input {
  display: none;
}

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
</style>
