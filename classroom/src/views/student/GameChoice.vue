<script setup lang="ts">
import type { GameQuestion } from '@/types'
const props = defineProps<{
  question: GameQuestion
  isCorrect: boolean | null
}>()
const emit = defineEmits<{
  'update:user-answer': [val: string]
}>()
const select = (id:string)=>{
  emit('update:user-answer', id)
}
</script>
<template>
  <div class="flex gap-4">
    <button
      v-for="opt in question.choices"
      :key="opt.id"
      @click="select(opt.id)"
      class="px-6 py-3 rounded-lg bg-white/20 hover:bg-white/40 border"
      :class="{
        'border-green-400': isCorrect === true && opt.id === question.answer,
        'border-red-400': isCorrect === false && opt.id === question.answer
      }"
    >
      {{ opt.text }}
    </button>
  </div>
</template>
