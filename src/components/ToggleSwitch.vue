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
  <span
    class="inline-flex items-center gap-2 cursor-pointer select-none text-[13px] font-medium text-slate-700 whitespace-nowrap"
    @click.stop="toggle"
  >
    <span
      :class="[
        'w-9 h-5 rounded-full relative shrink-0 transition-colors duration-200',
        modelValue ? 'bg-blue-600' : 'bg-slate-300',
      ]"
    >
      <span
        :class="[
          'absolute top-0.5 w-4 h-4 bg-white rounded-full shadow-sm transition-transform duration-200',
          modelValue ? 'translate-x-4 left-0.5' : 'translate-x-0 left-0.5',
        ]"
      ></span>
    </span>
  </span>
</template>
