<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { useAuthStore } from '@/stores/auth'
import { Button } from '@/components/ui/Button'
import { Input } from '@/components/ui/Input'
import MarkdownEditor from '@/components/ui/MarkdownEditor.vue'
import MarkdownRenderer from '@/components/ui/MarkdownRenderer.vue'
import { Card, CardContent } from '@/components/ui/Card'
import { Badge } from '@/components/ui/Badge'
import { Label } from '@/components/ui/Label'
import { LoadErrorBanner } from '@/components/ui/LoadErrorBanner'
import {
  Dialog,
  DialogHeader,
  DialogTitle,
} from '@/components/ui/Dialog'
import {
  ArrowLeft,
  BookOpen,
  LogOut,
  Plus,
  Users,
  Clock,
  CalendarDays,
  ExternalLink,
  Loader2,
  Trash2,
  Eye,
  Pencil,
  MessageSquare,
} from 'lucide-vue-next'
import {
  findCourseById,
  updateCourse,
  getAssignmentsByCourse,
  getMembersByCourse,
  findProfileById,
  saveAssignment,
  updateAssignment,
  deleteAssignment,
  getDiscussionCountsByAssignments,
} from '@/lib/db'
import { supabase } from '@/lib/supabase'
import type { Course, Assignment, SubmitType, CourseMember } from '@/types'

const router = useRouter()
const route = useRoute()
const authStore = useAuthStore()

const course = ref<Course | null>(null)
const assignments = ref<Assignment[]>([])
const members = ref<CourseMember[]>([])
const memberNameMap = ref<Record<string, string>>({})
const isCreateDialogOpen = ref(false)
const isCreating = ref(false)
const discussionCountMap = ref<Record<string, number>>({})
const isLoading = ref(true)
const errorMessage = ref('')
const isEditingMaterial = ref(false)
const isSavingMaterial = ref(false)
const saveMaterialError = ref('')
const materialLinksInput = ref([
  { title: '', url: '' },
  { title: '', url: '' },
  { title: '', url: '' },
])

const activeMaterialLinks = computed(() => course.value?.materialLinks?.filter(l => l.url) ?? [])

// Preview state（以学生视角查看作业内容）
const isPreviewDialogOpen = ref(false)
const previewAssignment = ref<Assignment | null>(null)

// Edit state
const isEditDialogOpen = ref(false)
const editingAssignment = ref<Assignment | null>(null)
const isEditing = ref(false)
const editForm = ref({
  title: '',
  description: '',
  submitType: 'game' as SubmitType,
  releaseDate: '',
  dueDate: '',
  showcaseEnabled: true,
  showcaseRequireApproval: true,
})

const newAssignment = ref({
  title: '',
  description: '',
  submitType: 'game' as SubmitType,
  releaseDate: '',
  dueDate: '',
  showcaseEnabled: true,
  showcaseRequireApproval: true,
})

const courseId = computed(() => route.params.id as string)

// ========== 闯关作业 - 关卡编辑器【移到这里！！在所有函数外面，顶层】 ==========
type QuestionType = 'single' | 'multiple' | 'blank' | 'sort'
interface LevelItem {
  id: string
  questionType: QuestionType
  title: string
  description: string
  options: Array<{ label: string; isAnswer: boolean }>
  blankAnswer: string
  sortItems: string[]
}
// 新增作业的关卡列表
const levelList = ref<LevelItem[]>([])
// 新增空白关卡
const addNewLevel = () => {
  levelList.value.push({
    id: crypto.randomUUID(),
    questionType: 'single' as QuestionType,
    title: `关卡 ${levelList.value.length + 1}`,
    description: '',
    options: [{ label: '', isAnswer: false }, { label: '', isAnswer: false }],
    blankAnswer: '',
    sortItems: []
  })
}

// 删除关卡
const removeLevel = (levelId:string) => {
  levelList.value = levelList.value.filter(l => l.id !== levelId)
}

// 切换题型，清空无关字段
const changeQuestionType = (level: LevelItem) => {
  if(level.questionType === 'single' || level.questionType === 'multiple'){
    if(!Array.isArray(level.options) || level.options.length <2) {
      level.options = [{ label: '', isAnswer: false }, { label: '', isAnswer: false }]
    }
    level.blankAnswer = ''
    level.sortItems = []
  }else if(level.questionType === 'blank'){
    level.options = []
    level.sortItems = []
  }else if(level.questionType === 'sort'){
    level.options = []
    level.blankAnswer = ''
  }
}

// 编辑作业的关卡列表
const editLevelList = ref<LevelItem[]>([])

// 打开新增作业弹窗的时候清空关卡
const openCreateDialog = () => {
  isCreateDialogOpen.value = true
  levelList.value = []
}

// 打开编辑作业弹窗时，读取已存在的关卡
const openEditDialog = async (assignment: Assignment) => {
  editingAssignment.value = assignment
  editForm.value = {
    title: assignment.title,
    description: assignment.description,
    submitType: assignment.submitType,
    releaseDate: toLocalDatetimeInput(assignment.releaseDate),
    dueDate: assignment.dueDate ? toLocalDatetimeInput(assignment.dueDate) : '',
    showcaseEnabled: assignment.showcaseEnabled,
    showcaseRequireApproval: assignment.showcaseRequireApproval,
  }
  // 读取这一份作业的所有关卡
  const { data } = await supabase
    .from('assignment_questions')
    .select('*')
    .eq('assignment_id', assignment.id)
    .order('order_index', { ascending: true })
  // ✅ 重点：把数据库返回的数据做类型转换，强制question_type转为QuestionType
  editLevelList.value = (data || []).map(item => ({
    id: item.id,
    questionType: item.question_type as QuestionType,
    title: item.title,
    description: item.description ?? '',
    options: item.options ?? [],
    blankAnswer: item.blank_answer ?? '',
    sortItems: item.sort_items ?? []
  }))
  isEditDialogOpen.value = true
}

// ========== handleCreateAssignment，保存关卡到数据库 ==========
async function handleCreateAssignment() {
  if (!newAssignment.value.title.trim() || !course.value) return
  isCreating.value = true
  try {
    const maxOrderIndex = assignments.value.length > 0
      ? Math.max(...assignments.value.map(a => a.orderIndex))
      : -1

    const created = await saveAssignment({
      courseId: course.value.id,
      title: newAssignment.value.title.trim(),
      description: newAssignment.value.description.trim(),
      orderIndex: maxOrderIndex + 1,
      submitType: 'game', // 全部作业强制为闯关模式
      releaseDate: newAssignment.value.releaseDate
        ? new Date(newAssignment.value.releaseDate).toISOString()
        : new Date().toISOString(),
      dueDate: newAssignment.value.dueDate
        ? new Date(newAssignment.value.dueDate).toISOString()
        : undefined,
      isActive: true,
      showcaseEnabled: newAssignment.value.showcaseEnabled,
      showcaseRequireApproval: newAssignment.value.showcaseRequireApproval,
    })

    // 批量插入关卡到 assignment_questions
    if(levelList.value.length > 0){
      const insertData = levelList.value.map((item, idx) => ({
        assignment_id: created.id,
        order_index: idx,
        question_type: item.questionType as QuestionType,
        title: item.title,
        description: item.description,
        options: item.options,
        blank_answer: item.blankAnswer,
        sort_items: item.sortItems
      }))
      await supabase.from('assignment_questions').insert(insertData)
    }

    // Notify enrolled students via email
    supabase.functions.invoke('send-email', {
      body: { assignmentId: created.id, type: 'assignment_released' }
    }).catch(console.error)

    newAssignment.value = {
      title: '',
      description: '',
      submitType: 'game',
      releaseDate: '',
      dueDate: '',
      showcaseEnabled: true,
      showcaseRequireApproval: true,
    }
    isCreateDialogOpen.value = false
    await loadData()
  } catch (e) {
    console.error('Failed to create assignment:', e)
  } finally {
    isCreating.value = false
  }
}

// ========== handleEditAssignment，更新关卡 ==========
async function handleEditAssignment() {
  if (!editForm.value.title.trim() || !editingAssignment.value) return
  isEditing.value = true
  try {
    await updateAssignment(editingAssignment.value.id, {
      title: editForm.value.title.trim(),
      description: editForm.value.description.trim(),
      submitType: 'game',
      releaseDate: editForm.value.releaseDate
        ? new Date(editForm.value.releaseDate).toISOString()
        : editingAssignment.value.releaseDate,
      dueDate: editForm.value.dueDate
        ? new Date(editForm.value.dueDate).toISOString()
        : undefined,
      showcaseEnabled: editForm.value.showcaseEnabled,
      showcaseRequireApproval: editForm.value.showcaseRequireApproval,
    })
    // 删除旧关卡，再写入新关卡
    if (!editingAssignment.value) return
    const assignmentId = editingAssignment.value.id
    await supabase.from('assignment_questions').delete().eq('assignment_id', assignmentId)
    if(editLevelList.value.length>0){
      const insertData = editLevelList.value.map((item, idx)=>({
        assignment_id: assignmentId,
        order_index: idx,
        question_type: item.questionType as QuestionType,
        title: item.title,
        description: item.description,
        options: item.options,
        blank_answer: item.blankAnswer,
        sort_items: item.sortItems
      }))
      await supabase.from('assignment_questions').insert(insertData)
    }
    isEditDialogOpen.value = false
    editingAssignment.value = null
    await loadData()
  } catch (e) {
    console.error('Failed to update assignment:', e)
  } finally {
    isEditing.value = false
  }
}

// ========== 下面是原本旧的函数（保持原样） ==========
onMounted(async () => {
  await loadData()
})

async function loadData() {
  isLoading.value = true
  errorMessage.value = ''

  try {
    if (!authStore.profile && authStore.user) {
      await authStore.refreshProfile()
    }

    if (!authStore.profile) {
      router.push('/login')
      return
    }

    const courseData = await findCourseById(courseId.value)
    if (!courseData || courseData.teacherId !== authStore.profile.id) {
      router.push('/teacher')
      return
    }

    course.value = courseData
    const [assignmentData, memberData] = await Promise.all([
      getAssignmentsByCourse(courseId.value),
      getMembersByCourse(courseId.value),
    ])
    assignments.value = assignmentData
    members.value = memberData
    discussionCountMap.value = await getDiscussionCountsByAssignments(assignments.value.map(a => a.id))
    await loadMemberNames()
  } catch (e) {
    console.error('Failed to load course detail:', e)
    errorMessage.value = '载入课程资料失败，请重新整理或稍后试。'
  } finally {
    isLoading.value = false
  }
}

async function loadMemberNames() {
  const entries = await Promise.all(
    members.value.map(async (m) => {
      const profile = await findProfileById(m.studentId)
      return [m.studentId, profile?.name ?? '未知学生'] as const
    })
  )
  memberNameMap.value = Object.fromEntries(entries)
}

function getStudentName(studentId: string): string {
  return memberNameMap.value[studentId] ?? '未知学生'
}

function stripMarkdown(text: string, maxLines = 5): string {
  return text
    .replace(/#{1,6}\s+/g, '')
    .replace(/\*\*(.+?)\*\*/g, '$1')
    .replace(/\*(.+?)\*/g, '$1')
    .replace(/`(.+?)`/g, '$1')
    .replace(/\[(.+?)\]\(.+?\)/g, '$1')
    .replace(/^[-*+>]\s+/gm, '')
    .split('\n')
    .map(l => l.trim())
    .filter(l => l.length > 0)
    .slice(0, maxLines)
    .join('\n')
}

function formatDate(iso: string) {
  return new Date(iso).toLocaleDateString('zh-TW', { year: 'numeric', month: 'long', day: 'numeric' })
}

function toLocalDatetimeInput(iso: string): string {
  const d = new Date(iso)
  const pad = (n: number) => String(n).padStart(2, '0')
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}T${pad(d.getHours())}:${pad(d.getMinutes())}`
}

function openPreviewDialog(assignment: Assignment) {
  previewAssignment.value = assignment
  isPreviewDialogOpen.value = true
}

async function handleDeleteAssignment(assignmentId: string) {
  if (confirm('确定要删除此作业吗？此操作无法复原。')) {
    await deleteAssignment(assignmentId)
    await loadData()
  }
}

function startEditMaterial() {
  const existing = course.value?.materialLinks ?? []
  materialLinksInput.value = [
    { title: existing[0]?.title ?? '', url: existing[0]?.url ?? '' },
    { title: existing[1]?.title ?? '', url: existing[1]?.url ?? '' },
    { title: existing[2]?.title ?? '', url: existing[2]?.url ?? '' },
  ]
  isEditingMaterial.value = true
}

async function handleSaveMaterialLinks() {
  if (!course.value) return

  isSavingMaterial.value = true
  saveMaterialError.value = ''

  try {
    const materialLinks = materialLinksInput.value
      .filter(l => l.url.trim())
      .map(l => ({ title: l.title.trim(), url: l.url.trim() }))
    await updateCourse(course.value.id, { materialLinks })
    course.value = { ...course.value, materialLinks: materialLinks.length > 0 ? materialLinks : undefined }
    isEditingMaterial.value = false
  } catch (e) {
    console.error('Failed to update material links:', e)
    saveMaterialError.value = e instanceof Error ? e.message : '储存失败，请稍后再试。'
  } finally {
    isSavingMaterial.value = false
  }
}

function handleLogout() {
  authStore.logout()
  router.push('/login')
}

// ========== 新增【添加学生】相关变量和函数 ==========
const showAddStudentModal = ref(false)
const searchStudentList = ref<Array<{id:string,name:string}>>([])
const searchKeyword = ref('')

// 打开弹窗
const openAddStudentModal = () => {
  showAddStudentModal.value = true
  searchKeyword.value = ''
  searchStudentList.value = []
}

// 搜索学生（只查role=student）
const searchStudents = async () => {
  const { data } = await supabase
    .from('profiles')
    .select('id, name')
    .eq('role','student')
    .ilike('name',`%${searchKeyword.value}%`)
  searchStudentList.value = data || []
}

// 加入课程，插入course_member
const addStudentToCourse = async (studentId: string) => {
  if (!courseId.value) return

  // 先查询该学生是否已经加入课程
  const { data: existMember } = await supabase
    .from('course_member')
    .select('id')
    .eq('course_id', courseId.value)
    .eq('student_id', studentId)
    .single()

  if (existMember) {
    alert('该学生已经在本课程内！')
    return
  }

  await supabase.from('course_member').insert({
    course_id: courseId.value,
    student_id: studentId
  })
  showAddStudentModal.value = false
  await loadData() // 刷新学生名单
}

function createNewLevel() {
  return {
    id: crypto.randomUUID(),
    questionType: 'single' as QuestionType,
    title: `关卡 ${editLevelList.value.length + 1}`,
    description: '',
    options: [{ label: '', isAnswer: false }, { label: '', isAnswer: false }],
    blankAnswer: '',
    sortItems: []
  }
}
</script>

<template>
  <div class="min-h-screen bg-slate-50">
    <!-- Header -->
    <header class="bg-white border-b border-slate-200 sticky top-0 z-10">
      <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 h-16 flex items-center justify-between">
        <div class="flex items-center gap-4">
          <Button variant="ghost" size="sm" @click="router.push('/teacher')">
            <ArrowLeft class="h-4 w-4 mr-2" />
            返回
          </Button>
          <div v-if="course" class="flex items-center gap-2">
            <BookOpen class="h-6 w-6 text-slate-900" />
            <span class="text-xl font-bold">{{ course.name }}</span>
          </div>
        </div>
        <Button variant="ghost" size="sm" @click="handleLogout">
          <LogOut class="h-4 w-4 mr-2" />
          登出
        </Button>
      </div>
    </header>

    <main v-if="isLoading" class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-16">
      <div class="flex items-center justify-center text-slate-500">
        <Loader2 class="mr-2 h-5 w-5 animate-spin" />
        载入课程作业中...
      </div>
    </main>

    <main v-else-if="errorMessage" class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-16">
      <LoadErrorBanner :message="errorMessage" :is-retrying="isLoading" @retry="loadData" />
    </main>

    <!-- Main Content -->
    <main v-else-if="course" class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
      <!-- Course Info -->
      <div class="mb-8">
        <p class="text-slate-600 mb-4">{{ course.description }}</p>
        <div class="flex flex-wrap items-center gap-4 text-sm text-slate-600">
          <span class="flex items-center gap-1">
            <Users class="h-4 w-4" />
            {{ members.length }} 位学生
          </span>
          <Badge variant="secondary">课程码：{{ course.courseCode }}</Badge>
        </div>
        <div class="mt-4 rounded-lg border border-slate-200 bg-white p-4">
          <div class="flex items-center justify-between mb-3">
            <h2 class="text-sm font-semibold text-slate-900">教材连接</h2>
            <Button v-if="!isEditingMaterial" variant="outline" size="sm" @click="startEditMaterial">
              {{ activeMaterialLinks.length > 0 ? '修改连接' : '新增连接' }}
            </Button>
          </div>

          <!-- 展示模式 -->
          <div v-if="!isEditingMaterial">
            <div v-if="activeMaterialLinks.length > 0" class="flex flex-wrap gap-2">
              <a
                v-for="link in activeMaterialLinks"
                :key="link.url"
                :href="link.url"
                target="_blank"
                rel="noopener noreferrer"
                class="inline-flex items-center gap-2 rounded-lg border border-slate-200 px-3 py-2.5 hover:border-slate-300 hover:shadow-sm transition-all group"
              >
                <ExternalLink class="h-4 w-4 text-slate-400 shrink-0 group-hover:text-blue-500 transition-colors" />
                <div>
                  <p class="text-sm font-medium text-slate-900 leading-tight">{{ link.title || link.url }}</p>
                  <p class="text-xs text-slate-400 leading-tight mt-0.5">教材连接</p>
                </div>
              </a>
            </div>
            <p v-else class="text-sm text-slate-500">尚未设定教材连接</p>
          </div>

          <!-- 编辑模式 -->
          <div v-else class="space-y-2">
            <div v-for="(link, i) in materialLinksInput" :key="i" class="flex gap-2">
              <div class="w-2/5">
                <Input v-model="link.title" placeholder="标题（选填）" />
              </div>
              <div class="flex-1">
                <Input v-model="link.url" type="url" placeholder="https://..." />
              </div>
            </div>
            <p v-if="saveMaterialError" class="text-sm text-red-600">{{ saveMaterialError }}</p>
            <div class="flex justify-end gap-2 pt-1">
              <Button variant="outline" :disabled="isSavingMaterial" @click="isEditingMaterial = false">取消</Button>
              <Button :disabled="isSavingMaterial" @click="handleSaveMaterialLinks">
                <Loader2 v-if="isSavingMaterial" class="mr-2 h-4 w-4 animate-spin" />
                保存
              </Button>
            </div>
          </div>
        </div>
      </div>

      <!-- Students List -->
      <Card class="mb-8">
        <CardContent class="!p-6">
          <h2 class="text-base font-semibold text-slate-900 mb-4">学生名单</h2>
          <!-- 新增：添加学生按钮 -->
      <Button @click="openAddStudentModal">
        + 添加学生
      </Button>
          <div v-if="members.length === 0" class="text-center py-4 text-slate-500 text-sm">
            尚未学生加入此课程
          </div>
          <div v-else class="flex flex-wrap gap-2">
            <Badge
              v-for="member in members"
              :key="member.id"
              variant="outline"
            >
              {{ getStudentName(member.studentId) }}
            </Badge>
          </div>
        </CardContent>
      </Card>

      <!-- Assignments -->
      <div class="space-y-4">
        <div class="flex items-center justify-between mb-2">
          <h2 class="text-xl font-semibold text-slate-900">课程作业</h2>
          <Button @click="openCreateDialog">
            <Plus class="h-4 w-4 mr-2" />
          新增作业
          </Button>
        </div>

        <div v-if="assignments.length === 0" class="text-center py-16 text-slate-500">
          <BookOpen class="h-10 w-10 mx-auto mb-3 text-slate-300" />
          <p class="font-medium">此课程暂无作业</p>
          <p class="text-sm mt-1">点击上方按钮新增作业</p>
        </div>

        <div v-else class="space-y-3">
          <Card
            v-for="assignment in assignments"
            :key="assignment.id"
            class="hover:shadow-md transition-shadow duration-200"
          >
            <CardContent class="!p-5 lg:!p-6">
              <!-- Title row -->
              <div class="flex flex-wrap items-center gap-2 mb-3">
                <h3 class="font-semibold text-slate-900 text-base leading-snug">
                  {{ assignment.title }}
                </h3>
              </div>

              <!-- Description preview -->
              <p
                v-if="assignment.description"
                class="text-sm text-slate-600 leading-relaxed whitespace-pre-line mb-4"
              >{{ stripMarkdown(assignment.description) }}</p>

              <!-- Dates -->
              <div class="flex flex-wrap items-center gap-4 text-xs text-slate-500 mb-4">
                <span class="flex items-center gap-1">
                  <CalendarDays class="h-3.5 w-3.5" />
                  发布：{{ formatDate(assignment.releaseDate) }}
                </span>
                <span v-if="assignment.dueDate" class="flex items-center gap-1">
                  <Clock class="h-3.5 w-3.5" />
                  截止：{{ formatDate(assignment.dueDate) }}
                </span>
              </div>

              <!-- Actions -->
              <div class="flex items-center justify-end gap-2 border-t border-slate-100 pt-4">
                <router-link :to="`/teacher/discussion/${assignment.id}`">
                  <Button variant="ghost" size="sm" class="cursor-pointer">
                    <MessageSquare class="h-4 w-4 mr-1.5" />
                    讨论区
                    <span v-if="discussionCountMap[assignment.id]" class="ml-1.5 bg-slate-100 text-slate-600 text-xs rounded-full px-1.5 py-0.5 font-medium leading-none">
                      {{ discussionCountMap[assignment.id] }}
                    </span>
                  </Button>
                </router-link>
                <router-link :to="`/teacher/submissions/${assignment.id}`">
                  <Button variant="outline" size="sm" class="cursor-pointer">
                    <Eye class="h-4 w-4 mr-1.5" />
                    查看提交
                  </Button>
                </router-link>
                <Button
                  variant="outline"
                  size="sm"
                  class="cursor-pointer"
                  @click="openPreviewDialog(assignment)"
                >
                  <BookOpen class="h-4 w-4 mr-1.5" />
                  预览
                </Button>
                <Button
                  variant="ghost"
                  size="sm"
                  class="cursor-pointer"
                  @click="openEditDialog(assignment)"
                >
                  <Pencil class="h-4 w-4" />
                </Button>
                <Button
                  variant="ghost"
                  size="sm"
                  class="text-red-600 hover:text-red-700 hover:bg-red-50 cursor-pointer"
                  @click="handleDeleteAssignment(assignment.id)"
                >
                  <Trash2 class="h-4 w-4" />
                </Button>
              </div>
            </CardContent>
          </Card>
        </div>
      </div>
    </main>

    <!-- Create Assignment Dialog -->
    <Dialog v-model:open="isCreateDialogOpen" class="max-w-5xl max-h-[90vh] overflow-y-auto">
      <div class="space-y-4">
        <DialogHeader>
          <DialogTitle>新增作业</DialogTitle>
        </DialogHeader>
        <div class="space-y-4">
          <div class="space-y-2">
            <Label for="title">作业标题</Label>
            <Input
              id="title"
              v-model="newAssignment.title"
              placeholder="输入作业标题"
            />
          </div>
          <div class="space-y-2">
            <Label>作业描述</Label>
            <MarkdownEditor
              v-model="newAssignment.description"
              placeholder="支援 Markdown 格式，例如 **粗体**、`程式码`、清单等"
              minHeight="320px"
            />
          </div>
         <!-- 闯关关卡编辑器 -->
<div class="space-y-4 border-t pt-4">
  <div class="flex justify-between items-center">
    <Label>🎮 关卡题目管理</Label>
    <Button variant="outline" size="sm" @click="addNewLevel">+ 添加一关</Button>
  </div>

  <div v-if="levelList.length === 0" class="text-sm text-slate-500 border rounded p-4 text-center">
    还没有关卡，请点击【添加一关】来创建第一题
  </div>

  <!-- 循环渲染每一关 -->
  <div v-for="(level, idx) in levelList" :key="level.id" class="border rounded-lg p-4 space-y-3">
    <div class="flex justify-between items-center">
      <span class="font-medium">第 {{ idx +1 }} 关</span>
      <Button variant="ghost" size="sm" class="text-red-500" @click="removeLevel(level.id)">删除本关</Button>
    </div>

    <!-- 题型下拉选择 -->
    <div>
      <Label>题型</Label>
      <select v-model="level.questionType" @change="changeQuestionType(level)" class="border rounded px-2 py-1 w-full mt-1">
        <option value="single">单选题</option>
        <option value="multiple">多选题</option>
        <option value="blank">填空题</option>
        <option value="sort">排序题</option>
      </select>
    </div>

    <div>
      <Label>关卡标题</Label>
      <Input v-model="level.title" placeholder="例如：认识动物 第1题" />
    </div>

    <div>
      <Label>题目描述</Label>
      <Input v-model="level.description" placeholder="请输入题目内容" />
    </div>

    <!-- 单选 / 多选题选项 -->
    <div v-if="level.questionType === 'single' || level.questionType === 'multiple'" class="space-y-2">
      <Label>选项设置（勾选为正确答案）</Label>
      <div v-for="(opt, optIdx) in level.options" :key="optIdx" class="flex gap-2 items-center">
        <input type="checkbox" v-model="opt.isAnswer" />
        <Input v-model="opt.label" placeholder="选项内容" />
        <Button size="sm" variant="ghost" @click="level.options.splice(optIdx,1)">-</Button>
      </div>
      <Button size="sm" variant="outline" @click="level.options.push({label:'', isAnswer:false})">+新增选项</Button>
    </div>

    <!-- 填空题 -->
    <div v-if="level.questionType === 'blank'">
      <Label>正确答案</Label>
      <Input v-model="level.blankAnswer" placeholder="填写答案" />
    </div>

    <!-- 排序题 -->
    <div v-if="level.questionType === 'sort'" class="space-y-2">
      <Label>正确顺序（从上到下为正确顺序）</Label>
      <div v-for="(_item, sortIdx) in level.sortItems" :key="sortIdx" class="flex gap-2 items-center">
        <span>{{sortIdx+1}}.</span>
        <Input v-model="level.sortItems[sortIdx]" placeholder="项目内容" />
        <Button size="sm" variant="ghost" @click="level.sortItems.splice(sortIdx,1)">-</Button>
      </div>
      <Button size="sm" variant="outline" @click="level.sortItems.push('')">+新增排序项目</Button>
    </div>
  </div>
</div>
          <div class="grid grid-cols-2 gap-4">
            <div class="space-y-2">
              <Label for="releaseDate">发布时间</Label>
              <Input
                id="releaseDate"
                v-model="newAssignment.releaseDate"
                type="datetime-local"
              />
            </div>
            <div class="space-y-2">
              <Label for="dueDate">截止时间（选填）</Label>
              <Input
                id="dueDate"
                v-model="newAssignment.dueDate"
                type="datetime-local"
              />
            </div>
          </div>
          <div class="space-y-2">
            <Label class="flex items-center gap-2">
              <input
                v-model="newAssignment.showcaseEnabled"
                type="checkbox"
                class="w-4 h-4"
              />
              启用作业展示
            </Label>
            <Label v-if="newAssignment.showcaseEnabled" class="flex items-center gap-2 ml-6">
              <input
                v-model="newAssignment.showcaseRequireApproval"
                type="checkbox"
                class="w-4 h-4"
              />
              需要老师审核
            </Label>
          </div>
          <div class="flex justify-end gap-2">
            <Button variant="outline" @click="isCreateDialogOpen = false">取消</Button>
            <Button
              :disabled="!newAssignment.title.trim() || isCreating"
              @click="handleCreateAssignment"
            >
              <Loader2 v-if="isCreating" class="mr-2 h-4 w-4 animate-spin" />
              {{ isCreating ? '建立中...' : '建立' }}
            </Button>
          </div>
        </div>
      </div>
    </Dialog>

    <!-- Preview Assignment Dialog（学生视角）-->
    <Dialog v-model:open="isPreviewDialogOpen" class="max-w-3xl max-h-[85vh] overflow-y-auto">
      <div class="space-y-4">
        <DialogHeader>
          <DialogTitle>{{ previewAssignment?.title }}</DialogTitle>
          <p class="text-sm text-slate-500 mt-1">这是学生会看到的作业内容</p>
        </DialogHeader>
        <div v-if="previewAssignment?.description" class="assignment-document">
          <MarkdownRenderer :content="previewAssignment.description" />
        </div>
        <div v-else class="rounded-2xl border border-dashed border-slate-300 bg-slate-50 p-8 text-center text-slate-500">
          这份作业尚未提供详细内容。
        </div>
      </div>
    </Dialog>

    <!-- Edit Assignment Dialog -->
    <Dialog v-model:open="isEditDialogOpen" class="max-w-5xl max-h-[90vh] overflow-y-auto">
      <div class="space-y-4">
        <DialogHeader>
          <DialogTitle>编辑作业</DialogTitle>
        </DialogHeader>
        <div class="space-y-4">
          <div class="space-y-2">
            <Label for="edit-title">作业标题</Label>
            <Input
              id="edit-title"
              v-model="editForm.title"
              placeholder="输入作业标题"
            />
          </div>
          <div class="space-y-2">
            <Label>作业描述</Label>
            <MarkdownEditor
              v-model="editForm.description"
              placeholder="支援 Markdown 格式，例如 **粗体**、`程式码`、清单等"
              minHeight="320px"
            />
          </div>
         <!-- 编辑作业 - 闯关关卡编辑器 -->
<div class="space-y-4 border-t pt-4">
  <div class="flex justify-between items-center">
    <Label>🎮 关卡题目管理</Label>
<Button variant="outline" size="sm" @click="editLevelList.push(createNewLevel())"> 添加一关</Button>
  </div>

  <div v-if="editLevelList.length === 0" class="text-sm text-slate-500 border rounded p-4 text-center">
    还没有关卡，请点击【添加一关】来创建第一题
  </div>

  <div v-for="(level, idx) in editLevelList" :key="level.id" class="border rounded-lg p-4 space-y-3">
    <div class="flex justify-between items-center">
      <span class="font-medium">第 {{ idx +1 }} 关</span>
      <Button variant="ghost" size="sm" class="text-red-500" @click="editLevelList = editLevelList.filter(l=>l.id!==level.id)">删除本关</Button>
    </div>

    <div>
      <Label>题型</Label>
      <select v-model="level.questionType" @change="changeQuestionType(level)" class="border rounded px-2 py-1 w-full mt-1">
        <option value="single">单选题</option>
        <option value="multiple">多选题</option>
        <option value="blank">填空题</option>
        <option value="sort">排序题</option>
      </select>
    </div>

    <div>
      <Label>关卡标题</Label>
      <Input v-model="level.title" placeholder="例如：认识动物 第1题" />
    </div>

    <div>
      <Label>题目描述</Label>
      <Input v-model="level.description" placeholder="请输入题目内容" />
    </div>

    <div v-if="level.questionType === 'single' || level.questionType === 'multiple'" class="space-y-2">
      <Label>选项设置（勾选为正确答案）</Label>
      <div v-for="(opt, optIdx) in level.options" :key="optIdx" class="flex gap-2 items-center">
        <input type="checkbox" v-model="opt.isAnswer" />
        <Input v-model="opt.label" placeholder="选项内容" />
        <Button size="sm" variant="ghost" @click="level.options.splice(optIdx,1)">-</Button>
      </div>
      <Button size="sm" variant="outline" @click="level.options.push({label:'', isAnswer:false})">+新增选项</Button>
    </div>

    <div v-if="level.questionType === 'blank'">
      <Label>正确答案</Label>
      <Input v-model="level.blankAnswer" placeholder="填写答案" />
    </div>

    <div v-if="level.questionType === 'sort'" class="space-y-2">
      <Label>正确顺序（从上到下为正确顺序）</Label>
      <div v-for="(_item, sortIdx) in level.sortItems" :key="sortIdx" class="flex gap-2 items-center">
        <span>{{sortIdx+1}}.</span>
        <Input v-model="level.sortItems[sortIdx]" placeholder="项目内容" />
        <Button size="sm" variant="ghost" @click="level.sortItems.splice(sortIdx,1)">-</Button>
      </div>
      <Button size="sm" variant="outline" @click="level.sortItems.push('')">+新增排序项目</Button>
    </div>
  </div>
</div>
          <div class="grid grid-cols-2 gap-4">
            <div class="space-y-2">
              <Label for="edit-releaseDate">发布时间</Label>
              <Input
                id="edit-releaseDate"
                v-model="editForm.releaseDate"
                type="datetime-local"
              />
            </div>
            <div class="space-y-2">
              <Label for="edit-dueDate">截止时间（选填）</Label>
              <Input
                id="edit-dueDate"
                v-model="editForm.dueDate"
                type="datetime-local"
              />
            </div>
          </div>
          <div class="space-y-2">
            <Label class="flex items-center gap-2">
              <input
                v-model="editForm.showcaseEnabled"
                type="checkbox"
                class="w-4 h-4"
              />
              启用作业展示
            </Label>
            <Label v-if="editForm.showcaseEnabled" class="flex items-center gap-2 ml-6">
              <input
                v-model="editForm.showcaseRequireApproval"
                type="checkbox"
                class="w-4 h-4"
              />
              需要老师审核
            </Label>
          </div>
          <div class="flex justify-end gap-2">
            <Button variant="outline" @click="isEditDialogOpen = false">取消</Button>
            <Button
              :disabled="!editForm.title.trim() || isEditing"
              @click="handleEditAssignment"
            >
              <Loader2 v-if="isEditing" class="mr-2 h-4 w-4 animate-spin" />
              {{ isEditing ? '储存中...' : '储存变更' }}
            </Button>
          </div>
        </div>
      </div>
    </Dialog>
  </div>
  <!-- 添加学生弹窗 -->
<Dialog v-model:open="showAddStudentModal">
  <DialogHeader>
    <DialogTitle>添加学生到课程</DialogTitle>
  </DialogHeader>
  <div class="space-y-4 py-2">
    <Input 
      v-model="searchKeyword" 
      placeholder="输入学生名字搜索..." 
      @input="searchStudents"
    />
    <div class="space-y-2 max-h-64 overflow-y-auto">
      <div v-for="stu in searchStudentList" :key="stu.id" class="flex justify-between items-center border p-2 rounded">
        <span>{{ stu.name }}</span>
        <Button @click="addStudentToCourse(stu.id)">加入课程</Button>
      </div>
      <p v-if="searchStudentList.length === 0 && searchKeyword" class="text-sm text-slate-500">找不到学生</p>
    </div>
  </div>
</Dialog>
</template>
