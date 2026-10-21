<script setup lang="ts">
import { ref } from 'vue'
import type { GameQuestion } from '@/types'
const props = defineProps<{
  question: GameQuestion
  isCorrect: boolean | null
}>()
const emit = defineEmits<{
  'update:user-answer': [val: string]
}>()
const inputVal = ref('')
const handleInput = ()=>{
  emit('update:user-answer', inputVal.value.trim())
}
</script>
<template>
  <div>
    <input
      v-model="inputVal"
      @input="handleInput"
      class="px-4 py-2 rounded text-black"
      placeholder="在这里填写答案"
      :class="{
        'ring-2 ring-green-400': isCorrect === true,
        'ring-2 ring-red-400': isCorrect === false
      }"
    />
  </div>
</template>
