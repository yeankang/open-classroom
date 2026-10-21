<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { Button } from '@/components/ui/Button'
import type { GameAssignment, GameQuestion, GameQuestionType, GameScene } from '@/types'

// ✅ 【重点】现在先导入三个题型组件（后面你要新建这三个文件）
import GameChoice from './GameChoice.vue'
import GameFillBlank from './GameFillBlank.vue'
import GameSort from './GameSort.vue'

const route = useRoute()
const router = useRouter()
const assignmentId = route.params.assignmentId as string
const isPreview = computed(() => !!route.query.preview)

// ========== 游戏全局状态 ==========
const MAX_LIFE = 3
const lifeCount = ref(MAX_LIFE)
const timeLeft = ref(0)
let timer: ReturnType<typeof setInterval> | null = null
const currentQuestionIndex = ref(0)
const userAnswer = ref<string | string[] | null>(null)
const isCorrect = ref<boolean | null>(null)

// 场景配色映射
const sceneBgMap: Record<GameScene, string> = {
  forest: 'linear-gradient(180deg, #22543d, #276749)',
  castle: 'linear-gradient(180deg, #4a5568, #2d3748)',
  space: 'linear-gradient(180deg, #1a202c, #0b1120)',
  ocean: 'linear-gradient(180deg, #2b6cb0, #2c5282)',
}

// 模拟测试题目
const gameAssignment = ref<GameAssignment>({
  id: 'demo-001',
  title: '闯关练习 Demo',
  questions: [
    {
      id: 'q1',
      type: 'choice',
      scene: 'forest',
      questionText: '第一题，森林场景，选择题',
      choices: [
        { id: 'c1', text: '选项A' },
        { id: 'c2', text: '选项B' },
      ],
      answer: 'c1',
      timeLimit: 60,
    },
    {
      id: 'q2',
      type: 'sort',
      scene: 'castle',
      questionText: '第二题，城堡场景，排序题',
      answer: ['a','b'],
      timeLimit: 60,
    },
    {
      id: 'q3',
      type: 'fillblank',
      scene: 'space',
      questionText: '第三题，太空场景，填空题',
      answer: 'moon',
      timeLimit: 60,
    },
  ]
})

const currentQuestion = computed(() => {
  return gameAssignment.value.questions[currentQuestionIndex.value]
})
const questionTotal = computed(() => gameAssignment.value.questions.length)
const isLastQuestion = computed(() => currentQuestionIndex.value >= questionTotal.value - 1)

// ========== 计时器逻辑 ==========
function startQuestionTimer() {
  if(isPreview.value) return
  if(timer) clearInterval(timer)
  if (!currentQuestion.value) return
  timeLeft.value = currentQuestion.value.timeLimit

  timer = setInterval(() => {
    timeLeft.value -= 1
    if(timeLeft.value <= 0) {
      handleWrongAnswer()
      nextQuestion()
    }
  },1000)
}

function handleWrongAnswer() {
  isCorrect.value = false
  if(isPreview.value) return
  lifeCount.value -= 1
  if(lifeCount.value <= 0) {
    endGame()
  }
}
function handleCorrectAnswer() {
  isCorrect.value = true
}

function endGame() {
  if(timer) clearInterval(timer)
  router.push(`/student/game-result/${assignmentId}?fail=true`)
}

function nextQuestion() {
  isCorrect.value = null
  userAnswer.value = null
  if(timer) clearInterval(timer)

  if (currentQuestionIndex.value < gameAssignment.value.questions.length -1) {
    currentQuestionIndex.value ++
    startQuestionTimer()
  } else {
    if(timer) clearInterval(timer)
    router.push(`/student/game-result/${assignmentId}?success=true`)
  }
}

async function submitAnswer() {
  if (!currentQuestion.value || userAnswer.value === null) return
  const correctAns = currentQuestion.value.answer
  let ok = false
  if (Array.isArray(correctAns)) {
    ok = JSON.stringify(userAnswer.value) === JSON.stringify(correctAns)
  } else {
    ok = userAnswer.value === correctAns
  }

  if(ok){
    handleCorrectAnswer()
  }else{
    handleWrongAnswer()
  }
  if (!isPreview.value) {
    // await saveStudentAnswer(assignmentId, currentQuestion.value.id, userAnswer.value, ok)
  }
}

onUnmounted(()=>{
  if(timer) clearInterval(timer)
})

onMounted(()=>{
  startQuestionTimer()
})

// ✅ 修复：直接返回导入好的组件，不再返回字符串
const getQuestionComponent = (type:GameQuestionType) => {
  switch(type){
    case 'choice': return GameChoice
    case 'fillblank': return GameFillBlank
    case 'sort': return GameSort
    default: return null
  }
}
</script>

<template>
  <div class="min-h-screen bg-slate-900 text-white p-6">
    <!-- 顶部游戏HUD：格子进度、爱心、倒计时、预览标记 -->
    <div class="flex items-center justify-between mb-6">
      <!-- 格子进度 -->
      <div class="flex gap-1">
        <div
          v-for="(_, idx) in questionTotal"
          :key="idx"
          class="w-8 h-8 rounded border-2 transition-colors duration-300"
          :class="{
            'bg-emerald-400 border-emerald-200': idx < currentQuestionIndex,
            'bg-amber-400 border-amber-200': idx === currentQuestionIndex,
            'bg-slate-600 border-slate-500': idx > currentQuestionIndex
          }"
        ></div>
      </div>

      <!-- 爱心生命值 -->
      <div class="flex gap-1 text-2xl">
        <span v-for="i in MAX_LIFE" :key="i">
          {{ i <= lifeCount ? '❤️' : '🖤' }}
        </span>
      </div>

      <!-- 倒计时 -->
      <div class="font-mono text-xl px-3 py-1 rounded bg-slate-700">
        ⏱️ {{ timeLeft }}s
      </div>

      <!-- 教师预览标签 -->
      <div v-if="isPreview" class="bg-amber-400 text-black px-3 py-1 rounded text-sm font-medium">
        教师预览模式
      </div>
    </div>

    <!-- 场景区域 -->
    <transition name="scene-fade" mode="out-in">
      <div
        v-if="currentQuestion"
        :key="currentQuestionIndex"
        class="game-scene relative h-[420px] rounded-xl overflow-hidden"
        :style="{background: sceneBgMap[currentQuestion.scene]}"
      >
        <div class="absolute inset-0 bg-black/25 z-0"></div>
        <!-- 小人占位 -->
        <div class="absolute bottom-8 left-12 w-14 h-14 bg-blue-400 rounded-full z-10"></div>

        <!-- ✅ 动态组件 -->
        <component
          v-if="getQuestionComponent(currentQuestion.type)"
          :is="getQuestionComponent(currentQuestion.type)"
          :question="currentQuestion"
          v-model:user-answer="userAnswer"
          :is-correct="isCorrect"
          class="absolute inset-0 z-10 flex items-center justify-center"
        />
      </div>
    </transition>

    <!-- 按钮区域 -->
    <div class="mt-6">
      <p class="text-lg">{{ currentQuestion?.questionText }}</p>
      <div class="flex gap-3 mt-4">
        <Button @click="submitAnswer" :disabled="userAnswer === null">
          提交答案
        </Button>
        <Button v-if="isCorrect !== null" @click="nextQuestion">
          {{ isLastQuestion ? '完成闯关' : '下一关' }}
        </Button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.scene-fade-enter-active,
.scene-fade-leave-active {
  transition: opacity 0.4s ease;
}
.scene-fade-enter-from,
.scene-fade-leave-to {
  opacity: 0;
}
</style>
