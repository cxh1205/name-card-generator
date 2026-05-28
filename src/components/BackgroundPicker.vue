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
    <div class="flex gap-2 items-center">
      <div
        v-if="image"
        class="group relative flex-1 rounded-md overflow-hidden"
        @click="removeImage"
      >
        <img
          :src="image"
          class="w-full h-[54px] object-cover block rounded-md border border-slate-200"
        />
        <div
          class="absolute inset-0 flex items-center justify-center gap-1.5 opacity-0 transition-opacity duration-200 rounded-md text-white text-[13px] font-semibold cursor-pointer pointer-events-none group-hover:opacity-100 group-hover:pointer-events-auto"
          style="background: rgba(239, 68, 68, 0.78)"
        >
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="3 6 5 6 21 6"/><path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"/></svg>
          <span>删除</span>
        </div>
      </div>
      <ColorPicker v-else v-model="colorModel" style="flex: 1" />
      <button
        :class="[
          'inline-flex items-center gap-1.5 px-2.5 py-[7px] border border-dashed rounded-md bg-white text-slate-500 text-[12.5px] cursor-pointer whitespace-nowrap transition-all duration-150 font-sans',
          image
            ? 'border-solid border-slate-300'
            : 'border-slate-300 hover:border-blue-500 hover:text-blue-600 hover:bg-blue-50',
        ]"
        @click="triggerUpload"
      >
        <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8.5" cy="8.5" r="1.5"/><polyline points="21 15 16 10 5 21"/></svg>
        {{ image ? '更换图片' : '上传图片' }}
      </button>
      <input ref="fileInput" type="file" accept="image/*" class="hidden" @change="handleUpload" />
    </div>
    <div v-if="hint && image" class="text-[11.5px] text-slate-400 leading-[1.4] mt-2">{{ hint }}</div>
  </div>
</template>
