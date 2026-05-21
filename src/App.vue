<template>
  <div class="app">
    <!-- 首页 -->
    <div v-if="page === 'home'" class="page home">
      <div class="logo">📝</div>
      <h1>智能做题</h1>
      <p class="subtitle">AI解析 · 错题本 · 即时反馈</p>
      <div class="menu">
        <button class="btn primary" @click="startExam('all')">开始做题</button>
        <button class="btn secondary" @click="page = 'wrong'">📚 错题本 ({{ wrongQuestions.length }})</button>
        <button class="btn secondary" @click="showSettings = true">⚙️ 设置</button>
      </div>
    </div>

    <!-- 设置页 -->
    <div v-if="showSettings" class="page settings">
      <div class="header">
        <button class="back" @click="showSettings = false">← 返回</button>
        <h2>设置</h2>
      </div>
      <div class="form-group">
        <label>AI API 地址</label>
        <input v-model="settings.apiBase" placeholder="http://localhost:11434/v1" />
      </div>
      <div class="form-group">
        <label>API Key</label>
        <input v-model="settings.apiKey" placeholder="ollama 或 OpenAI Key" type="password" />
      </div>
      <div class="form-group">
        <label>模型</label>
        <input v-model="settings.model" placeholder="llama3 / gpt-4o" />
      </div>
      <div class="form-group">
        <label>题型</label>
        <label><input type="checkbox" v-model="settings.types.cloze" /> 选词填空</label>
        <label><input type="checkbox" v-model="settings.types.choice" /> 单选题</label>
        <label><input type="checkbox" v-model="settings.types.judge" /> 判断题</label>
      </div>
      <button class="btn primary" @click="saveSettings">保存</button>
    </div>

    <!-- 做题页 -->
    <div v-if="page === 'exam'" class="page exam">
      <div class="exam-header">
        <button class="back" @click="quitExam">← 退出</button>
        <span class="progress">{{ currentIndex + 1 }} / {{ examQuestions.length }}</span>
        <button class="star" @click="toggleFavorite">★</button>
      </div>
      <div class="progress-bar">
        <div class="bar" :style="{ width: ((currentIndex + 1) / examQuestions.length * 100) + '%' }"></div>
      </div>

      <!-- 选词填空 -->
      <div v-if="current.type === 'cloze'" class="question cloze">
        <div class="q-type">📝 选词填空</div>
        <p class="q-text">{{ current.english }}</p>
        <div class="options">
          <button
            v-for="opt in current.options"
            :key="opt"
            class="option"
            :class="{
              selected: selectedAnswer === opt,
              correct: submitted && opt === current.answer,
              wrong: submitted && selectedAnswer === opt && opt !== current.answer
            }"
            :disabled="submitted"
            @click="selectAnswer(opt)"
          >
            {{ opt }}
          </button>
        </div>
      </div>

      <!-- 单选题 -->
      <div v-if="current.type === 'choice'" class="question choice">
        <div class="q-type">🔘 单选题</div>
        <p class="q-text">{{ current.question }}</p>
        <div class="options">
          <button
            v-for="(opt, i) in current.options"
            :key="i"
            class="option"
            :class="{
              selected: selectedAnswer === i,
              correct: submitted && i === current.answer,
              wrong: submitted && selectedAnswer === i && i !== current.answer
            }"
            :disabled="submitted"
            @click="selectAnswer(i)"
          >
            {{ opt }}
          </button>
        </div>
      </div>

      <!-- 判断题 -->
      <div v-if="current.type === 'judge'" class="question judge">
        <div class="q-type">✓ 判断题</div>
        <p class="q-text">{{ current.question }}</p>
        <div class="options judge-options">
          <button
            class="option"
            :class="{
              selected: selectedAnswer === true,
              correct: submitted && current.answer === true,
              wrong: submitted && selectedAnswer === true && current.answer !== true
            }"
            :disabled="submitted"
            @click="selectAnswer(true)"
          >✓ 对</button>
          <button
            class="option"
            :class="{
              selected: selectedAnswer === false,
              correct: submitted && current.answer === false,
              wrong: submitted && selectedAnswer === false && current.answer !== false
            }"
            :disabled="submitted"
            @click="selectAnswer(false)"
          >✗ 错</button>
        </div>
      </div>

      <!-- 提交/下一题 -->
      <div class="actions">
        <button v-if="!submitted" class="btn primary" :disabled="selectedAnswer === null" @click="submitAnswer">
          提交答案
        </button>
        <div v-if="submitted" class="feedback">
          <div class="result" :class="{ correct: isCorrect, wrong: !isCorrect }">
            {{ isCorrect ? '✓ 正确' : '✗ 错误' }}
          </div>
          <div class="explanation">
            <strong>答案解析：</strong>{{ current.explanation }}
          </div>
          <button v-if="current.aiAnalysis" class="btn ai-btn" @click="showAiAnalysis = !showAiAnalysis">
            🤖 AI深度解析 {{ showAiAnalysis ? '▼' : '▶' }}
          </button>
          <div v-if="showAiAnalysis && current.aiAnalysis" class="ai-content">{{ current.aiAnalysis }}</div>
          <div v-if="showAiAnalysis && !current.aiAnalysis" class="ai-loading">正在调用AI分析...</div>
          <button class="btn secondary" @click="nextQuestion">
            {{ currentIndex + 1 < examQuestions.length ? '下一题 →' : '查看结果' }}
          </button>
        </div>
      </div>
    </div>

    <!-- 结果页 -->
    <div v-if="page === 'result'" class="page result">
      <div class="score-circle">
        <span class="score">{{ correctCount }}</span>
        <span class="total">/ {{ examQuestions.length }}</span>
      </div>
      <p class="score-text">正确率 {{ Math.round(correctCount / examQuestions.length * 100) }}%</p>
      <div class="stat-row">
        <span>✅ 正确: {{ correctCount }}</span>
        <span>❌ 错误: {{ examQuestions.length - correctCount }}</span>
      </div>
      <button class="btn primary" @click="startExam('all')">再来一轮</button>
      <button class="btn secondary" @click="page = 'wrong'" v-if="wrongQuestions.length > 0">
        📚 复习错题
      </button>
      <button class="btn secondary" @click="page = 'home'">返回首页</button>
    </div>

    <!-- 错题本 -->
    <div v-if="page === 'wrong'" class="page wrong-book">
      <div class="header">
        <button class="back" @click="page = 'home'">← 返回</button>
        <h2>错题本</h2>
      </div>
      <div v-if="wrongQuestions.length === 0" class="empty">
        <p>🎉 暂无错题，保持得好！</p>
      </div>
      <div v-else class="wrong-list">
        <div v-for="(q, i) in wrongQuestions" :key="q.id" class="wrong-item" @click="reviewQuestion(i)">
          <div class="wrong-type">{{ q.type === 'cloze' ? '📝' : q.type === 'choice' ? '🔘' : '✓' }}</div>
          <div class="wrong-content">
            <p>{{ q.type === 'cloze' ? q.english : q.question }}</p>
            <span class="wrong-answer">你的答案: {{ formatAnswer(q, q.userAnswer) }}</span>
          </div>
        </div>
      </div>
      <button v-if="wrongQuestions.length > 0" class="btn primary" @click="reviewWrong">重新练习错题</button>
    </div>

    <!-- AI加载中 -->
    <div v-if="aiLoading" class="ai-loading-overlay">
      <div class="spinner"></div>
      <p>🤖 AI分析中...</p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import { Haptics, ImpactStyle } from '@capacitor/haptics'

// ===== 题库 =====
const questionBank = {
  cloze: [
    {
      id: 'c1', type: 'cloze',
      english: 'The quick brown fox ___ over the lazy dog.',
      options: ['jumps', 'jumped', 'jumping', 'jump'],
      answer: 'jumps',
      explanation: '主谓一致：The quick brown fox是第三人称单数，用jumps。',
      difficulty: 1
    },
    {
      id: 'c2', type: 'cloze',
      english: 'She ___ to the market yesterday.',
      options: ['go', 'goes', 'went', 'going'],
      answer: 'went',
      explanation: '时间状语yesterday表示过去，用过去式went。',
      difficulty: 1
    },
    {
      id: 'c3', type: 'cloze',
      english: 'If I ___ rich, I would travel the world.',
      options: ['am', 'was', 'were', 'be'],
      answer: 'were',
      explanation: '虚拟语气：对现在的假设，用were。',
      difficulty: 2
    },
    {
      id: 'c4', type: 'cloze',
      english: 'He ___ basketball every weekend.',
      options: ['play', 'plays', 'played', 'playing'],
      answer: 'plays',
      explanation: '一般现在时：主语he为第三人称单数，动词加s。',
      difficulty: 1
    },
    {
      id: 'c5', type: 'cloze',
      english: 'The sun ___ in the east.',
      options: ['rise', 'rises', 'rose', 'rising'],
      answer: 'rises',
      explanation: '一般现在时：自然规律用一般现在时，sun是第三人称单数。',
      difficulty: 1
    }
  ],
  choice: [
    {
      id: 'm1', type: 'choice',
      question: 'Python是什么类型的语言？',
      options: ['编译型', '解释型', '汇编型', '机器语言'],
      answer: 1,
      explanation: 'Python是解释型语言，逐行读取并执行代码。',
      difficulty: 1
    },
    {
      id: 'm2', type: 'choice',
      question: '以下哪个是Python的列表方法？',
      options: ['add()', 'append()', 'insert()', 'push()'],
      answer: 1,
      explanation: 'append()用于在列表末尾添加元素。',
      difficulty: 1
    },
    {
      id: 'm3', type: 'choice',
      question: 'JavaScript中，=== 表示什么？',
      options: ['赋值', '相等比较（值）', '严格相等（值+类型）', '不等比较'],
      answer: 2,
      explanation: '=== 是严格相等运算符，同时比较值和类型。',
      difficulty: 1
    }
  ],
  judge: [
    {
      id: 'j1', type: 'judge',
      question: 'Python的索引从1开始。',
      answer: false,
      explanation: 'Python索引从0开始，第一个元素索引为0。',
      difficulty: 1
    },
    {
      id: 'j2', type: 'judge',
      question: 'JavaScript是一种弱类型语言。',
      answer: true,
      explanation: 'JavaScript是弱类型语言，变量类型自动转换。',
      difficulty: 1
    },
    {
      id: 'j3', type: 'judge',
      question: 'HTML是一种编程语言。',
      answer: false,
      explanation: 'HTML是标记语言，不是编程语言。',
      difficulty: 1
    }
  ]
}

// ===== 状态 =====
const page = ref('home')
const showSettings = ref(false)
const showAiAnalysis = ref(false)
const aiLoading = ref(false)
const settings = ref({
  apiBase: 'http://localhost:11434/v1',
  apiKey: 'ollama',
  model: 'llama3',
  types: { cloze: true, choice: true, judge: true }
})
const examQuestions = ref([])
const currentIndex = ref(0)
const selectedAnswer = ref(null)
const submitted = ref(false)
const correctCount = ref(0)
const wrongQuestions = ref([])
const favorites = ref([])

onMounted(() => {
  const saved = localStorage.getItem('wrongQuestions')
  if (saved) wrongQuestions.value = JSON.parse(saved)
  const savedSettings = localStorage.getItem('examSettings')
  if (savedSettings) settings.value = JSON.parse(savedSettings)
})

// ===== 计算属性 =====
const current = computed(() => examQuestions.value[currentIndex.value] || {})
const isCorrect = computed(() => {
  if (!current.value) return false
  if (current.value.type === 'cloze') return selectedAnswer.value === current.value.answer
  if (current.value.type === 'choice') return selectedAnswer.value === current.value.answer
  if (current.value.type === 'judge') return selectedAnswer.value === current.value.answer
  return false
})

// ===== 方法 =====
function saveSettings() {
  localStorage.setItem('examSettings', JSON.stringify(settings.value))
  showSettings.value = false
}

function startExam(mode) {
  let pool = []
  if (settings.value.types.cloze) pool = pool.concat(questionBank.cloze)
  if (settings.value.types.choice) pool = pool.concat(questionBank.choice)
  if (settings.value.types.judge) pool = pool.concat(questionBank.judge)
  examQuestions.value = shuffle(pool)
  currentIndex.value = 0
  selectedAnswer.value = null
  submitted.value = false
  correctCount.value = 0
  page.value = 'exam'
}

function shuffle(arr) {
  const a = [...arr]
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]]
  }
  return a
}

function selectAnswer(ans) {
  selectedAnswer.value = ans
}

async function submitAnswer() {
  submitted.value = true
  showAiAnalysis.value = false

  if (isCorrect.value) {
    correctCount.value++
    try {
      await Haptics.impact({ style: ImpactStyle.Light })
    } catch(e) {}
  } else {
    try {
      await Haptics.impact({ style: ImpactStyle.Heavy })
    } catch(e) {}
    const q = { ...current.value, userAnswer: selectedAnswer.value }
    const exists = wrongQuestions.value.find(w => w.id === q.id)
    if (!exists) {
      wrongQuestions.value.push(q)
      localStorage.setItem('wrongQuestions', JSON.stringify(wrongQuestions.value))
    }
  }

  // 调用AI解析
  if (settings.value.apiKey) {
    aiLoading.value = true
    const analysis = await callAI(current.value)
    if (analysis) {
      examQuestions.value[currentIndex.value].aiAnalysis = analysis
    }
    aiLoading.value = false
    showAiAnalysis.value = true
  }
}

async function callAI(question) {
  const prompt = `你是一个英语/知识题目的AI家教。请为这道题提供深入浅出的讲解。

题目类型：${question.type === 'cloze' ? '选词填空' : question.type === 'choice' ? '单选题' : '判断题'}
${question.type === 'cloze' ? '原题：' + question.english : '题目：' + question.question}
答案：${question.type === 'choice' ? question.options[question.answer] : question.answer}

请用友好的语气解释：
1. 为什么这个答案是正确的
2. 常见的理解误区
3. 记忆技巧或关联知识

请用中文回答，控制在150字以内。`

  try {
    const res = await fetch(settings.value.apiBase + '/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer ' + settings.value.apiKey
      },
      body: JSON.stringify({
        model: settings.value.model,
        messages: [{ role: 'user', content: prompt }],
        max_tokens: 300
      })
    })
    if (!res.ok) return null
    const data = await res.json()
    return data.choices?.[0]?.message?.content || null
  } catch {
    return null
  }
}

function nextQuestion() {
  if (currentIndex.value + 1 < examQuestions.value.length) {
    currentIndex.value++
    selectedAnswer.value = null
    submitted.value = false
    showAiAnalysis.value = false
  } else {
    page.value = 'result'
  }
}

function quitExam() {
  page.value = 'home'
}

function toggleFavorite() {
  const id = current.value.id
  if (favorites.value.includes(id)) {
    favorites.value = favorites.value.filter(f => f !== id)
  } else {
    favorites.value.push(id)
  }
}

function formatAnswer(q, ans) {
  if (q.type === 'cloze') return ans
  if (q.type === 'choice') return q.options[ans]
  return ans ? '对' : '错'
}

function reviewQuestion(i) {
  // 暂时直接进入答题
  currentIndex.value = examQuestions.value.findIndex(q => q.id === wrongQuestions.value[i].id)
  if (currentIndex.value === -1) currentIndex.value = 0
  page.value = 'exam'
}

function reviewWrong() {
  if (wrongQuestions.value.length === 0) return
  examQuestions.value = shuffle([...wrongQuestions.value])
  currentIndex.value = 0
  selectedAnswer.value = null
  submitted.value = false
  page.value = 'exam'
}
</script>

<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; background: #f5f5f7; }

.app { min-height: 100vh; max-width: 480px; margin: 0 auto; background: #f5f5f7; }

.page { padding: 20px; min-height: 100vh; }
.header { display: flex; align-items: center; gap: 12px; margin-bottom: 20px; }
.back { background: none; border: none; font-size: 18px; cursor: pointer; }
h2 { font-size: 20px; }

/* 首页 */
.home { display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 24px; }
.logo { font-size: 80px; }
.home h1 { font-size: 32px; color: #1a1a1a; }
.subtitle { color: #888; font-size: 16px; }
.menu { display: flex; flex-direction: column; gap: 12px; width: 100%; max-width: 300px; }

/* 按钮 */
.btn { padding: 14px 24px; border: none; border-radius: 12px; font-size: 16px; font-weight: 600; cursor: pointer; width: 100%; transition: all 0.2s; }
.btn:disabled { opacity: 0.5; cursor: not-allowed; }
.btn.primary { background: #007AFF; color: white; }
.btn.primary:hover:not(:disabled) { background: #0056b3; }
.btn.secondary { background: white; color: #007AFF; border: 1.5px solid #007AFF; }
.btn.ai-btn { background: #AF52DE; color: white; margin-top: 12px; }

/* 设置 */
.settings { background: white; }
.form-group { margin-bottom: 20px; }
.form-group label { display: block; font-size: 14px; color: #666; margin-bottom: 6px; }
.form-group input { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 8px; font-size: 16px; }
.form-group label input { width: auto; margin-right: 8px; }

/* 做题页 */
.exam-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px; }
.progress { font-size: 16px; font-weight: 600; color: #007AFF; }
.star { background: none; border: none; font-size: 24px; cursor: pointer; color: #ddd; }
.star.active { color: #FFD700; }
.progress-bar { height: 4px; background: #e0e0e0; border-radius: 2px; margin-bottom: 24px; }
.progress-bar .bar { height: 100%; background: #007AFF; border-radius: 2px; transition: width 0.3s; }

.question { background: white; border-radius: 16px; padding: 24px; margin-bottom: 20px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); }
.q-type { font-size: 12px; color: #007AFF; margin-bottom: 12px; font-weight: 600; }
.q-text { font-size: 18px; line-height: 1.6; color: #1a1a1a; margin-bottom: 20px; }

.options { display: flex; flex-direction: column; gap: 10px; }
.option { padding: 14px 16px; border: 2px solid #e0e0e0; border-radius: 10px; background: white; font-size: 16px; text-align: left; cursor: pointer; transition: all 0.2s; }
.option:hover:not(:disabled) { border-color: #007AFF; }
.option.selected { border-color: #007AFF; background: #e8f0ff; }
.option.correct { border-color: #34C759; background: #e8f8ea; }
.option.wrong { border-color: #FF3B30; background: #fff0f0; }

.judge-options { flex-direction: row; }
.judge-options .option { flex: 1; text-align: center; }

/* 反馈 */
.feedback { display: flex; flex-direction: column; gap: 12px; }
.result { font-size: 24px; font-weight: 700; text-align: center; padding: 12px; border-radius: 12px; }
.result.correct { background: #e8f8ea; color: #34C759; }
.result.wrong { background: #fff0f0; color: #FF3B30; }
.explanation { background: white; padding: 16px; border-radius: 12px; font-size: 14px; line-height: 1.6; color: #333; }
.ai-content { background: #f0e8ff; padding: 16px; border-radius: 12px; font-size: 14px; line-height: 1.6; color: #333; }
.ai-loading { color: #888; text-align: center; font-size: 14px; }

/* 结果页 */
.result { display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 20px; }
.score-circle { width: 160px; height: 160px; border-radius: 50%; background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); display: flex; flex-direction: column; align-items: center; justify-content: center; color: white; }
.score { font-size: 56px; font-weight: 700; }
.total { font-size: 24px; opacity: 0.8; }
.score-text { font-size: 20px; color: #666; }
.stat-row { display: flex; gap: 24px; font-size: 16px; color: #333; }

/* 错题本 */
.wrong-book { background: white; }
.empty { text-align: center; padding: 60px 0; color: #888; font-size: 18px; }
.wrong-list { display: flex; flex-direction: column; gap: 12px; margin-bottom: 20px; }
.wrong-item { display: flex; gap: 12px; padding: 16px; background: #f9f9f9; border-radius: 12px; cursor: pointer; }
.wrong-type { font-size: 24px; }
.wrong-content p { font-size: 14px; color: #333; margin-bottom: 4px; }
.wrong-answer { font-size: 12px; color: #FF3B30; }

/* AI加载 */
.ai-loading-overlay { position: fixed; bottom: 20px; right: 20px; background: white; padding: 16px 20px; border-radius: 12px; box-shadow: 0 4px 16px rgba(0,0,0,0.15); display: flex; align-items: center; gap: 12px; z-index: 100; }
.spinner { width: 20px; height: 20px; border: 3px solid #e0e0e0; border-top-color: #007AFF; border-radius: 50%; animation: spin 0.8s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }
</style>
