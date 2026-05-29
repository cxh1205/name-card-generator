<script setup lang="ts">
import { ref, watch } from 'vue'

const props = defineProps<{
  modelValue: string
}>()

const emit = defineEmits<{
  'update:modelValue': [value: string]
}>()

const hexText = ref(props.modelValue)
let debounceTimer: ReturnType<typeof setTimeout> | null = null

watch(() => props.modelValue, (val) => {
  hexText.value = val
})

function onColorInput(e: Event): void {
  const value = (e.target as HTMLInputElement).value
  hexText.value = value
  emit('update:modelValue', value)
}

function onHexInput(e: Event): void {
  const value = (e.target as HTMLInputElement).value
  hexText.value = value

  if (debounceTimer) clearTimeout(debounceTimer)
  debounceTimer = setTimeout(() => {
    const normalized = normalizeHex(value)
    if (normalized) {
      emit('update:modelValue', normalized)
      hexText.value = normalized
    } else {
      hexText.value = props.modelValue
    }
  }, 500)
}

// 校验并规范化 hex 值：支持 #abc / #aabbcc / abc / aabbcc 格式
function normalizeHex(input: string): string | null {
  let hex = input.trim()
  if (!hex) return null
  if (!hex.startsWith('#')) hex = '#' + hex

  if (/^#[0-9a-fA-F]{3}$/.test(hex)) {
    return '#' + hex[1] + hex[1] + hex[2] + hex[2] + hex[3] + hex[3]
  }
  if (/^#[0-9a-fA-F]{6}$/.test(hex)) {
    return hex.toLowerCase()
  }
  return null
}
</script>

<template>
  <div class="flex items-center gap-1.5 bg-white border border-slate-200 rounded-md px-1.5 h-[30px] box-border w-full">
    <input
      type="color"
      :value="modelValue"
      @input="onColorInput"
      class="cp-swatch w-[22px] h-[22px] border-none rounded-sm cursor-pointer p-0 bg-none shrink-0"
    />
    <input
      type="text"
      :value="hexText"
      @input="onHexInput"
      class="w-[58px] shrink-0 border-none outline-none font-mono text-[11px] text-slate-700 bg-transparent p-0 leading-[22px] placeholder:text-slate-300 focus:text-slate-900"
      maxlength="7"
      placeholder="#000000"
      spellcheck="false"
    />
  </div>
</template>

<style scoped>
.cp-swatch::-webkit-color-swatch-wrapper {
  padding: 0;
}

.cp-swatch::-webkit-color-swatch {
  border-radius: 3px;
  border: 1px solid rgba(0, 0, 0, 0.1);
}
</style>
