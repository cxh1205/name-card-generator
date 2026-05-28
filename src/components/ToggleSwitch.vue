<script setup lang="ts">
const props = defineProps<{
  modelValue: boolean
}>()

const emit = defineEmits<{
  'update:modelValue': [value: boolean]
}>()

function toggle() {
  emit('update:modelValue', !props.modelValue)
}
</script>

<template>
  <span class="toggle-sw" @click.stop="toggle">
    <span class="toggle-track" :class="{ active: modelValue }">
      <span class="toggle-thumb"></span>
    </span>
  </span>
</template>

<style scoped>
.toggle-sw {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  user-select: none;
  font-size: 13px;
  font-weight: 500;
  color: var(--slate-700);
  white-space: nowrap;
}

.toggle-track {
  width: 36px;
  height: 20px;
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
  top: 2px;
  left: 2px;
  width: 16px;
  height: 16px;
  background: #fff;
  border-radius: 50%;
  transition: transform 0.2s;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.15);
}

.toggle-track.active .toggle-thumb {
  transform: translateX(16px);
}
</style>
