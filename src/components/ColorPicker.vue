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
      // 输入不合法，回退到当前有效值
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
  <div class="color-picker">
    <input type="color" :value="modelValue" @input="onColorInput" class="cp-swatch" />
    <input
      type="text"
      :value="hexText"
      @input="onHexInput"
      class="cp-hex"
      maxlength="7"
      placeholder="#000000"
      spellcheck="false"
    />
  </div>
</template>

<style scoped>
.color-picker {
  display: flex;
  align-items: center;
  gap: 6px;
  background: #fff;
  border: 1px solid var(--slate-200);
  border-radius: 5px;
  padding: 3px 6px;
  height: 30px;
  box-sizing: border-box;
}

.cp-swatch {
  width: 22px;
  height: 22px;
  border: none;
  border-radius: 3px;
  cursor: pointer;
  padding: 0;
  background: none;
  flex-shrink: 0;
}

.cp-swatch::-webkit-color-swatch-wrapper {
  padding: 0;
}

.cp-swatch::-webkit-color-swatch {
  border-radius: 3px;
  border: 1px solid rgba(0, 0, 0, 0.1);
}

.cp-hex {
  width: 58px;
  flex-shrink: 0;
  border: none;
  outline: none;
  font-family: "SF Mono", "Consolas", "Menlo", monospace;
  font-size: 11px;
  color: var(--slate-700);
  background: transparent;
  padding: 0;
  line-height: 22px;
}

.cp-hex::placeholder {
  color: var(--slate-300);
  font-family: "SF Mono", "Consolas", "Menlo", monospace;
}

.cp-hex:focus {
  color: var(--slate-900);
}
</style>
