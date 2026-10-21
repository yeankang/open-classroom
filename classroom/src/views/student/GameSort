<script setup lang="ts">
import { ref } from 'vue'
import type { GameQuestion } from '@/types'
const props = defineProps<{
  question: GameQuestion
  isCorrect: boolean | null
}>()
const emit = defineEmits<{
  'update:user-answer': [val: string[]]
}>()
const list = ref<string[]>(['a','b'])
const update = ()=>{
  emit('update:user-answer', [...list.value])
}
</script>
<template>
  <div class="flex gap-3">
    <div
      v-for="item in list"
      :key="item"
      class="px-4 py-3 bg-white/20 rounded border"
    >
      {{ item }}
    </div>
  </div>
</template>
