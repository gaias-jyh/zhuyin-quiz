<script setup>
import { ref, computed, onMounted, onUnmounted, inject } from 'vue'
import ZhuyinKeyboard from './ZhuyinKeyboard.vue'
import dictionaryRaw from '../assets/dictionary.csv?raw'

const props = defineProps({
  questionsCount: {
    type: Number,
    default: 20
  },
  timeLimit: {
    type: Number,
    default: 600
  },
  pointsPerQuestion: {
    type: Number,
    default: 5
  }
})

const emit = defineEmits(['finish'])

// ============ 全域深色模式注入 ============
const isDarkMode = inject('isDarkMode', ref(true)) // 從 App.vue 注入深色模式狀態

const questions = ref([])
const currentIndex = ref(0)
const userAnswers = ref({}) // Map of index -> answer string
const checkedAnswers = ref({}) // Map of index -> boolean (true if checked)
const timeLeft = ref(props.timeLimit)
const timerInterval = ref(null)
const isPaused = ref(false)
const startTime = ref(null) // 測驗開始時間
const totalTimeSpent = ref(0) // 總作答時間（秒）

// ============ 修正模式狀態 ============
const correctionMode = ref({}) // Map of index -> boolean (是否處於修正模式)
const correctSyllables = ref({}) // Map of index -> array (正確的音節陣列)
const wrongSyllableIndex = ref({}) // Map of index -> number (需要修正的音節索引)
const correctionInput = ref({}) // Map of index -> string (修正輸入的內容)
const originalWrongAnswer = ref({}) // Map of index -> string (原始錯誤答案)
const initialAnswers = ref({}) // Map of index -> string (第一次提交時的答案，用於計分)

// 處理輕聲符號：將輕聲符號（˙）從音節開頭移到末尾
const normalizeZhuyin = (zhuyin) => {
  // 按空格分割音節
  const syllables = zhuyin.split(/\s+/).filter(s => s.length > 0)
  
  // 處理每個音節：如果輕聲符號在開頭，移到末尾
  const normalizedSyllables = syllables.map(syllable => {
    // 如果音節以輕聲符號（˙）開頭
    if (syllable.startsWith('˙')) {
      // 移除開頭的輕聲符號，然後加到末尾
      return syllable.substring(1) + '˙'
    }
    return syllable
  })
  
  // 重新組合
  return normalizedSyllables.join(' ')
}

// Parse CSV and initialize quiz
const initQuiz = () => {
  const lines = dictionaryRaw.trim().split('\n')
  const allQuestions = lines.map(line => {
    const firstComma = line.indexOf(',')
    const secondComma = line.indexOf(',', firstComma + 1)

    if (firstComma === -1 || secondComma === -1) return null

    const word = line.substring(0, firstComma).trim()
    let zhuyin = line.substring(firstComma + 1, secondComma).trim()
    const explanation = line.substring(secondComma + 1).trim()

    // 處理輕聲符號：將輕聲符號從音節開頭移到末尾
    zhuyin = normalizeZhuyin(zhuyin)

    return { word, zhuyin, explanation }
  }).filter(q => q !== null)

  // 檢查題目數量是否足夠
  const availableQuestions = allQuestions.length
  const requestedQuestions = props.questionsCount

  if (availableQuestions === 0) {
    console.error('字典中沒有可用題目')
    alert('錯誤：字典中沒有可用題目，請檢查 dictionary.csv 文件')
    return
  }

  // 如果可用題目少於請求的題目數量，使用可用題目數量
  const actualQuestionsCount = Math.min(availableQuestions, requestedQuestions)

  // Shuffle and pick questions
  // 如果題目數量不足，允許重複（先打亂，然後循環使用）
  let selectedQuestions = []
  if (availableQuestions >= requestedQuestions) {
    // 題目足夠，隨機選擇
    selectedQuestions = allQuestions
      .sort(() => Math.random() - 0.5)
      .slice(0, requestedQuestions)
  } else {
    // 題目不足，先打亂，然後循環使用直到達到請求數量
    const shuffled = allQuestions.sort(() => Math.random() - 0.5)
    for (let i = 0; i < requestedQuestions; i++) {
      selectedQuestions.push(shuffled[i % availableQuestions])
    }
  }

  questions.value = selectedQuestions

  // 記錄測驗開始時間
  startTime.value = Date.now()
  totalTimeSpent.value = 0

  startTimer()
}

const startTimer = () => {
  if (timerInterval.value) clearInterval(timerInterval.value)
  timerInterval.value = setInterval(() => {
    if (!isPaused.value) {
      if (timeLeft.value > 0) {
        timeLeft.value--
      } else {
        submitQuiz()
      }
    }
  }, 1000)
}

const formattedTime = computed(() => {
  const minutes = Math.floor(timeLeft.value / 60)
  const seconds = timeLeft.value % 60
  return `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`
})

const currentQuestion = computed(() => questions.value[currentIndex.value])
const currentAnswer = computed({
  get: () => userAnswers.value[currentIndex.value] || '',
  set: (val) => { userAnswers.value[currentIndex.value] = val }
})

const isCurrentChecked = computed(() => checkedAnswers.value[currentIndex.value])
const isCurrentCorrect = computed(() => {
  if (!currentQuestion.value) return false
  const userAnswer = (userAnswers.value[currentIndex.value] || '').replace(/\s/g, '')
  const correct = currentQuestion.value.zhuyin.replace(/\s/g, '')
  return userAnswer === correct
})

// 是否處於修正模式
const isInCorrectionMode = computed(() => correctionMode.value[currentIndex.value] === true)

// 當前題目的修正模式相關數據
const currentCorrectSyllables = computed(() => correctSyllables.value[currentIndex.value] || [])
const currentWrongSyllableIndex = computed(() => wrongSyllableIndex.value[currentIndex.value] ?? -1)
const currentCorrectionInput = computed({
  get: () => correctionInput.value[currentIndex.value] || '',
  set: (val) => { correctionInput.value[currentIndex.value] = val }
})

// 原始答案的音節陣列（用於顯示顏色區分）
const originalAnswerSyllables = computed(() => {
  if (!isInCorrectionMode.value) return []
  const originalAnswer = originalWrongAnswer.value[currentIndex.value] || ''
  return originalAnswer.trim().split(/\s+/).filter(s => s.length > 0)
})

// 判斷原始答案中每個音節是否正確
const isSyllableCorrect = (syllableIndex) => {
  if (!isInCorrectionMode.value) return false
  const correctSyllablesArray = currentCorrectSyllables.value
  const originalSyllablesArray = originalAnswerSyllables.value
  
  if (syllableIndex >= correctSyllablesArray.length || syllableIndex >= originalSyllablesArray.length) {
    return false
  }
  
  const correctSyllable = correctSyllablesArray[syllableIndex].replace(/\s/g, '')
  const originalSyllable = originalSyllablesArray[syllableIndex].replace(/\s/g, '')
  
  return correctSyllable === originalSyllable
}

// ============ 答案比較函數 ============
// 比較正確答案和用戶答案，找出錯誤的音節索引
const compareAnswers = (correct, user) => {
  // 按空格分割音節
  const correctSyllables = correct.trim().split(/\s+/).filter(s => s.length > 0)
  const userSyllables = user.trim().split(/\s+/).filter(s => s.length > 0)
  
  // 找出第一個不同的音節索引
  for (let i = 0; i < Math.max(correctSyllables.length, userSyllables.length); i++) {
    const correctSyllable = correctSyllables[i] || ''
    const userSyllable = userSyllables[i] || ''
    
    // 比較音節（移除空格後比較）
    if (correctSyllable.replace(/\s/g, '') !== userSyllable.replace(/\s/g, '')) {
      return {
        wrongIndex: i,
        correctSyllables: correctSyllables,
        userSyllables: userSyllables
      }
    }
  }
  
  // 如果所有音節都匹配，返回 null
  return null
}

const handleInput = (char) => {
  const index = currentIndex.value
  
  // 如果處於修正模式
  if (isInCorrectionMode.value) {
    // 只允許輸入錯誤音節的修正
    // 在修正模式下，不應該自動加空白，直接添加字符
    correctionInput.value[index] = (correctionInput.value[index] || '') + char
    
    // 檢查修正是否完成（與正確音節匹配）
    const wrongIndex = wrongSyllableIndex.value[index]
    const correctSyllable = correctSyllables.value[index][wrongIndex]
    const currentCorrection = correctionInput.value[index] || ''
    
    // 比較修正輸入和正確音節（移除空格後比較）
    if (currentCorrection.replace(/\s/g, '') === correctSyllable.replace(/\s/g, '')) {
      // 修正完成，合併答案
      completeCorrection()
    }
  } else if (!isCurrentChecked.value) {
    // 正常輸入模式
    currentAnswer.value += char
  }
}

const handleBackspace = () => {
  const index = currentIndex.value
  
  // 如果處於修正模式
  if (isInCorrectionMode.value) {
    const currentCorrection = correctionInput.value[index] || ''
    if (currentCorrection.length > 0) {
      correctionInput.value[index] = currentCorrection.slice(0, -1)
    }
  } else if (!isCurrentChecked.value) {
    // 正常輸入模式
    currentAnswer.value = currentAnswer.value.slice(0, -1)
  }
}

const handleClear = () => {
  const index = currentIndex.value
  
  // 如果處於修正模式
  if (isInCorrectionMode.value) {
    correctionInput.value[index] = ''
  } else if (!isCurrentChecked.value) {
    // 正常輸入模式
    currentAnswer.value = ''
  }
}

// 完成修正，合併正確答案
const completeCorrection = () => {
  const index = currentIndex.value
  const wrongIndex = wrongSyllableIndex.value[index]
  const correctSyllablesArray = correctSyllables.value[index]
  const originalAnswer = originalWrongAnswer.value[index]
  const originalSyllables = originalAnswer.trim().split(/\s+/).filter(s => s.length > 0)
  const correctedSyllable = correctSyllablesArray[wrongIndex]
  
  // 替換錯誤的音節
  originalSyllables[wrongIndex] = correctedSyllable
  
  // 合併成完整答案
  const correctedAnswer = originalSyllables.join(' ')
  userAnswers.value[index] = correctedAnswer
  
  // 立即清空修正輸入，觸發鍵盤切回子音
  correctionInput.value[index] = ''
  
  // 檢查是否還有其他錯誤
  const finalComparison = compareAnswers(currentQuestion.value.zhuyin, correctedAnswer)
  
  if (finalComparison) {
    // 還有其他錯誤，繼續修正下一個
    wrongSyllableIndex.value[index] = finalComparison.wrongIndex
    originalWrongAnswer.value[index] = correctedAnswer
    // 使用 nextTick 確保鍵盤狀態更新後再繼續修正
    nextTick(() => {
      // 此時 correctionInput 已經是空字符串，會觸發鍵盤切回子音
      // 然後準備修正下一個音節
    })
  } else {
    // 所有音節都正確，退出修正模式
    correctionMode.value[index] = false
    checkedAnswers.value[index] = true
    // 使用 nextTick 確保鍵盤狀態更新
    nextTick(() => {
      // 此時 isCurrentChecked 為 true，currentInput 會是空字符串
      // 這會觸發鍵盤的 watcher，讓鍵盤切回子音
    })
  }
}

const checkCurrentAnswer = () => {
  const index = currentIndex.value
  const correctAnswer = currentQuestion.value.zhuyin
  const userAnswer = userAnswers.value[index] || ''
  
  // 記錄第一次提交時的答案（用於計分）
  if (!initialAnswers.value.hasOwnProperty(index)) {
    initialAnswers.value[index] = userAnswer
  }
  
  // 檢查答案是否正確
  const isCorrect = userAnswer.replace(/\s/g, '') === correctAnswer.replace(/\s/g, '')
  
  if (isCorrect) {
    // 答案正確，直接標記為已檢查
    checkedAnswers.value[index] = true
    // 清除修正模式（如果有的話）
    correctionMode.value[index] = false
  } else {
    // 答案錯誤，進入修正模式
    const comparison = compareAnswers(correctAnswer, userAnswer)
    
    if (comparison) {
      // 進入修正模式
      correctionMode.value[index] = true
      correctSyllables.value[index] = comparison.correctSyllables
      wrongSyllableIndex.value[index] = comparison.wrongIndex
      correctionInput.value[index] = ''
      originalWrongAnswer.value[index] = userAnswer
      // 暫時不標記為已檢查，等修正完成後再標記
    } else {
      // 如果無法比較（例如格式問題），直接標記為已檢查
      checkedAnswers.value[index] = true
    }
  }
}

const nextQuestion = () => {
  if (currentIndex.value < props.questionsCount - 1) {
    currentIndex.value++
    // 清除修正模式狀態（如果有的話）
    if (correctionMode.value[currentIndex.value]) {
      correctionMode.value[currentIndex.value] = false
    }
  } else {
    submitQuiz()
  }
}

const prevQuestion = () => {
  if (currentIndex.value > 0) {
    currentIndex.value--
    // 清除修正模式狀態（如果有的話）
    if (correctionMode.value[currentIndex.value]) {
      correctionMode.value[currentIndex.value] = false
    }
  }
}

const submitQuiz = () => {
  clearInterval(timerInterval.value)

  // 計算總作答時間
  if (startTime.value) {
    const endTime = Date.now()
    totalTimeSpent.value = Math.floor((endTime - startTime.value) / 1000) // 轉換為秒
  }

  // 扣分制：從 100 分開始，答錯扣分
  // 使用第一次提交時的答案來計分（initialAnswers），而不是修正後的答案
  let score = 100
  questions.value.forEach((q, index) => {
    // 如果有記錄初始答案，使用初始答案計分；否則使用當前答案
    const answerToCheck = initialAnswers.value.hasOwnProperty(index) 
      ? initialAnswers.value[index] 
      : (userAnswers.value[index] || '')
    
    if (answerToCheck.replace(/\s/g, '') !== q.zhuyin.replace(/\s/g, '')) {
      score -= props.pointsPerQuestion
    }
  })

  // 確保分數不低於 0
  score = Math.max(0, score)

  // 計算平均每題作答時間
  const averageTimePerQuestion = questions.value.length > 0 
    ? Math.floor(totalTimeSpent.value / questions.value.length) 
    : 0

  emit('finish', score, totalTimeSpent.value, averageTimePerQuestion)
}

const togglePause = () => {
  isPaused.value = !isPaused.value
}

const confirmExit = () => {
  isPaused.value = true
  if (confirm('確定要離開考試嗎？進度將不會保存。')) {
    emit('finish', 0)
  } else {
    isPaused.value = false
  }
}

onMounted(() => {
  initQuiz()
})

onUnmounted(() => {
  if (timerInterval.value) clearInterval(timerInterval.value)
})
</script>

<template>
  <div class="quiz-view" :class="{ 'dark-mode': isDarkMode }">
    <div class="header">
      <div class="controls-left">
        <!-- 深色模式切換按鈕 -->
        <button @click="isDarkMode = !isDarkMode" class="icon-btn" :title="isDarkMode ? '切換到淺色模式' : '切換到深色模式'">
          <svg v-if="isDarkMode" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor"
            stroke-width="2">
            <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"></path>
          </svg>
          <svg v-else width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <circle cx="12" cy="12" r="5"></circle>
            <line x1="12" y1="1" x2="12" y2="3"></line>
            <line x1="12" y1="21" x2="12" y2="23"></line>
            <line x1="4.22" y1="4.22" x2="5.64" y2="5.64"></line>
            <line x1="18.36" y1="18.36" x2="19.78" y2="19.78"></line>
            <line x1="1" y1="12" x2="3" y2="12"></line>
            <line x1="21" y1="12" x2="23" y2="12"></line>
            <line x1="4.22" y1="19.78" x2="5.64" y2="18.36"></line>
            <line x1="18.36" y1="5.64" x2="19.78" y2="4.22"></line>
          </svg>
        </button>

        <button @click="confirmExit" class="icon-btn" title="離開">
          <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"
            stroke-linecap="round" stroke-linejoin="round">
            <line x1="18" y1="6" x2="6" y2="18"></line>
            <line x1="6" y1="6" x2="18" y2="18"></line>
          </svg>
        </button>
        <button @click="togglePause" class="icon-btn" title="暫停">
          <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor">
            <rect x="6" y="4" width="4" height="16" rx="1"></rect>
            <rect x="14" y="4" width="4" height="16" rx="1"></rect>
          </svg>
        </button>
      </div>
      <div class="timer" :class="{ 'urgent': timeLeft < 60 }">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"
          stroke-linecap="round" stroke-linejoin="round">
          <circle cx="12" cy="13" r="8"></circle>
          <polyline points="12 9 12 13 15 15"></polyline>
          <path d="M16.51 3.5L15 2"></path>
          <path d="M8.51 3.5L10 2"></path>
        </svg>
        {{ formattedTime }}
      </div>
      <div class="progress">
        {{ currentIndex + 1 }} / {{ props.questionsCount }}
      </div>
    </div>

    <div v-if="isPaused" class="pause-overlay">
      <div class="pause-content">
        <h2>考試暫停</h2>
        <button @click="togglePause" class="resume-btn">繼續考試</button>
      </div>
    </div>

    <div class="question-card" v-if="currentQuestion">
      <div class="word">{{ currentQuestion.word }}</div>
      <div class="explanation">{{ currentQuestion.explanation }}</div>

      <div class="answer-section">
        <div class="answer-wrapper">
          <!-- 答題後顯示圖標 -->
          <div v-if="isCurrentChecked && !isInCorrectionMode" class="answer-icon">
            <svg v-if="isCurrentCorrect" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="#42b883"
              stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
              <polyline points="20 6 9 17 4 12"></polyline>
            </svg>
            <svg v-else width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="#e74c3c" stroke-width="2.5"
              stroke-linecap="round" stroke-linejoin="round">
              <line x1="18" y1="6" x2="6" y2="18"></line>
              <line x1="6" y1="6" x2="18" y2="18"></line>
            </svg>
          </div>

          <!-- 修正模式：兩行顯示 -->
          <div v-if="isInCorrectionMode" class="correction-mode-container">
            <!-- 第一行：顯示使用者原本的答案（答對的綠色，答錯的紅色） -->
            <div class="original-answer-row">
              <span class="original-answer-label">您的答案：</span>
              <span class="original-answer-text">
                <template v-for="(syllable, index) in originalAnswerSyllables" :key="index">
                  <span :class="{ 'syllable-correct-in-original': isSyllableCorrect(index), 'syllable-wrong-in-original': !isSyllableCorrect(index) }">
                    {{ syllable }}<span v-if="index < originalAnswerSyllables.length - 1">&nbsp;</span>
                  </span>
                </template>
              </span>
            </div>
            
            <!-- 第二行：顯示需要修正的音節 -->
            <div class="correction-display">
              <template v-for="(syllable, index) in currentCorrectSyllables" :key="index">
                <!-- 已經答對的音節：綠色顯示，不需要再輸入 -->
                <span v-if="index < currentWrongSyllableIndex" class="syllable-correct">
                  {{ syllable }}<span v-if="index < currentCorrectSyllables.length - 1" class="syllable-separator">&nbsp;</span>
                </span>
                <!-- 需要修正的音節：紅色顯示，可編輯 -->
                <span v-else-if="index === currentWrongSyllableIndex" class="syllable-wrong">
                  {{ currentCorrectionInput }}<span class="cursor">|</span>
                </span>
                <!-- 待修正的音節：灰色顯示，稍後需要修正 -->
                <span v-else class="syllable-pending">
                  {{ syllable }}<span v-if="index < currentCorrectSyllables.length - 1" class="syllable-separator">&nbsp;</span>
                </span>
              </template>
            </div>
          </div>

          <!-- 正常模式：顯示完整答案 -->
          <div v-else class="answer-box"
            :class="{ 'correct': isCurrentChecked && isCurrentCorrect, 'wrong': isCurrentChecked && !isCurrentCorrect }">
            {{ currentAnswer }}<span v-if="!isCurrentChecked && !isInCorrectionMode" class="cursor">|</span>
          </div>
        </div>

        <!-- 修正模式提示 -->
        <div v-if="isInCorrectionMode" class="correction-hint">
          請修正錯誤的音節：<span class="correct-syllable-hint">{{ currentCorrectSyllables[currentWrongSyllableIndex] }}</span>
        </div>

        <!-- 只在答錯且不在修正模式時顯示正確答案 -->
        <div v-if="isCurrentChecked && !isCurrentCorrect && !isInCorrectionMode" class="correct-answer-label">
          正確答案：<span class="correct-answer">{{ currentQuestion.zhuyin }}</span>
        </div>
      </div>
    </div>

    <ZhuyinKeyboard @input="handleInput" @backspace="handleBackspace" @clear="handleClear"
      :disabled="(isCurrentChecked && !isInCorrectionMode) || isPaused" 
      :current-input="isInCorrectionMode ? currentCorrectionInput : (isCurrentChecked ? '' : currentAnswer)"
      :is-correction-mode="isInCorrectionMode" />

    <div class="navigation">
      <button @click="prevQuestion" :disabled="currentIndex === 0 || isPaused" class="nav-btn secondary">
        上一題
      </button>

      <button v-if="!isCurrentChecked && !isInCorrectionMode" @click="checkCurrentAnswer" class="nav-btn submit" :disabled="isPaused">
        送出
      </button>

      <button v-else-if="isInCorrectionMode" class="nav-btn correction" :disabled="isPaused">
        修正中...
      </button>

      <button v-else @click="nextQuestion" class="nav-btn primary" :disabled="isPaused">
        {{ currentIndex < props.questionsCount - 1 ? '下一題' : '查看結果' }} </button>
    </div>
  </div>
</template>

<style scoped>
.quiz-view {
  display: flex;
  flex-direction: column;
  align-items: center;
  max-width: 800px;
  margin: 0 auto;
  padding: 10px;
  /* iOS 修正：避免內容被底部導航欄遮擋 */
  height: calc(100vh - 120px);
  /* 支援 iOS safe area */
  padding-bottom: calc(10px + env(safe-area-inset-bottom));
  position: relative;
  transition: background-color 0.3s ease, color 0.3s ease;
}

/* 深色模式 */
.quiz-view.dark-mode {
  color: #fff;
}

.quiz-view.dark-mode .header {
  color: #ccc;
}

.quiz-view.dark-mode .icon-btn {
  background: rgba(58, 58, 60, 0.95);
  color: #ccc;
  border-color: rgba(255, 255, 255, 0.1);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}

.quiz-view.dark-mode .icon-btn:hover {
  background: rgba(68, 68, 70, 0.95);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.4);
}

.quiz-view.dark-mode .question-card {
  background: #2a2a2a;
  color: #fff;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
}

.quiz-view.dark-mode .word {
  color: #fff;
}

.quiz-view.dark-mode .explanation {
  background: #1a1a1a;
  color: #aaa;
}

.quiz-view.dark-mode .answer-label {
  color: #ccc;
}

.quiz-view.dark-mode .answer-box {
  color: #fff;
  border-bottom-color: #555;
}

.quiz-view.dark-mode .answer-box.correct {
  border-bottom-color: #42b883;
  color: #42b883;
}

.quiz-view.dark-mode .answer-box.wrong {
  border-bottom-color: #e74c3c;
  color: #e74c3c;
}

.quiz-view.dark-mode .feedback-section {
  background: #1a1a1a;
}

.quiz-view.dark-mode .pause-overlay {
  background: rgba(0, 0, 0, 0.95);
}

.quiz-view.dark-mode .pause-content h2 {
  color: #fff;
}

.quiz-view.dark-mode .nav-btn {
  color: #fff;
}

.quiz-view.dark-mode .nav-btn:disabled {
  background: #444;
  color: #888;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  margin-bottom: 10px;
  font-size: 1.1rem;
  font-weight: bold;
  color: #555;
}

.controls-left {
  display: flex;
  gap: 10px;
}

.icon-btn {
  background: rgba(255, 255, 255, 0.95);
  border: 1px solid rgba(0, 0, 0, 0.1);
  cursor: pointer;
  padding: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #555;
  transition: all 0.2s;
  border-radius: 8px;
  width: 40px;
  height: 40px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.icon-btn:hover {
  background: rgba(255, 255, 255, 1);
  transform: translateY(-1px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
}

.icon-btn:active {
  transform: translateY(0);
}

.timer {
  display: flex;
  align-items: center;
  gap: 6px;
}


.timer.urgent {
  color: #e74c3c;
  animation: pulse 1s infinite;
}

.pause-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(255, 255, 255, 0.95);
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 15px;
}

.pause-content {
  text-align: center;
}

.resume-btn {
  font-size: 1.5rem;
  padding: 1rem 3rem;
  background-color: #42b883;
  color: white;
  border: none;
  border-radius: 50px;
  cursor: pointer;
}

.question-card {
  background: white;
  padding: 10px;
  border-radius: 15px;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.08);
  width: 100%;
  margin-bottom: 10px;
  text-align: center;
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  overflow-y: auto;
  max-height: calc(100vh - 300px);
}

.word {
  font-size: 2.5rem;
  font-weight: bold;
  color: #2c3e50;
  margin-bottom: 5px;
}

.explanation {
  font-size: 1rem;
  color: #7f8c8d;
  margin-bottom: 20px;
  line-height: 1.4;
  padding: 8px;
  background: #f8f9fa;
  border-radius: 8px;
}

.answer-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  margin-bottom: 15px;
}

.answer-wrapper {
  display: flex;
  align-items: center;
  gap: 12px;
}

.answer-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 24px;
}

.answer-box {
  font-size: 1.5rem;
  min-width: 200px;
  min-height: 30px;
  border-bottom: 3px solid #ccc;
  padding: 3px 8px;
  font-family: "BiauKai", "DFKai-SB", serif;
  color: #333;
  transition: all 0.3s;
}

.answer-box.correct {
  border-bottom-color: #42b883;
  color: #333;
}

.answer-box.wrong {
  border-bottom-color: #e74c3c;
  color: #333;
}

.correct-answer-label {
  font-size: 0.9rem;
  color: #666;
  margin-top: 5px;
}

.correct-answer {
  color: #42b883;
  font-size: 1.3rem;
  font-family: "BiauKai", "DFKai-SB", serif;
  font-weight: bold;
}

/* ============ 修正模式樣式 ============ */
.correction-mode-container {
  display: flex;
  flex-direction: column;
  gap: 12px;
  width: 100%;
  min-width: 200px;
}

.original-answer-row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  background-color: rgba(231, 76, 60, 0.1);
  border-radius: 8px;
  border-left: 3px solid #e74c3c;
}

.original-answer-label {
  font-size: 0.9rem;
  color: #666;
  font-weight: 500;
  white-space: nowrap;
}

.original-answer-text {
  font-size: 1.3rem;
  font-family: "BiauKai", "DFKai-SB", serif;
  font-weight: bold;
  flex: 1;
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 2px;
}

.syllable-correct-in-original {
  color: #42b883;
}

.syllable-wrong-in-original {
  color: #e74c3c;
}

.correction-display {
  font-size: 1.5rem;
  min-width: 200px;
  min-height: 30px;
  padding: 3px 8px;
  font-family: "BiauKai", "DFKai-SB", serif;
  color: #333;
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 2px;
  line-height: 1.8;
}

/* 已經答對的音節：綠色，粗體，背景色區分 */
.syllable-correct {
  color: #42b883;
  font-weight: bold;
  background-color: rgba(66, 184, 131, 0.1);
  padding: 2px 6px;
  border-radius: 4px;
  border: 1px solid rgba(66, 184, 131, 0.3);
}

/* 需要修正的音節：紅色，有底線，可編輯 */
.syllable-wrong {
  color: #e74c3c;
  border-bottom: 3px solid #e74c3c;
  padding: 2px 4px;
  min-width: 20px;
  background-color: rgba(231, 76, 60, 0.1);
  border-radius: 4px;
}

/* 待修正的音節：灰色，半透明 */
.syllable-pending {
  color: #999;
  opacity: 0.5;
  padding: 2px 4px;
}

/* 音節分隔符 */
.syllable-separator {
  margin: 0 2px;
}

.correction-hint {
  font-size: 0.9rem;
  color: #666;
  margin-top: 5px;
  text-align: center;
}

.correct-syllable-hint {
  color: #42b883;
  font-size: 1.2rem;
  font-family: "BiauKai", "DFKai-SB", serif;
  font-weight: bold;
}

.nav-btn.correction {
  background-color: #f39c12;
  color: white;
}

.dark-mode .original-answer-row {
  background-color: rgba(231, 76, 60, 0.2);
  border-left-color: #e74c3c;
}

.dark-mode .original-answer-label {
  color: #aaa;
}

.dark-mode .syllable-correct-in-original {
  color: #42b883;
}

.dark-mode .syllable-wrong-in-original {
  color: #e74c3c;
}

.dark-mode .correction-display {
  color: #fff;
}

.dark-mode .syllable-correct {
  color: #42b883;
  background-color: rgba(66, 184, 131, 0.2);
  border-color: rgba(66, 184, 131, 0.5);
}

.dark-mode .syllable-wrong {
  color: #e74c3c;
  border-bottom-color: #e74c3c;
  background-color: rgba(231, 76, 60, 0.2);
}

.dark-mode .syllable-pending {
  color: #888;
  opacity: 0.4;
}

.dark-mode .correction-hint {
  color: #aaa;
}

.dark-mode .correct-syllable-hint {
  color: #42b883;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.cursor {
  animation: blink 1s infinite;
  color: #ccc;
  font-weight: 100;
}

@keyframes blink {

  0%,
  100% {
    opacity: 1;
  }

  50% {
    opacity: 0;
  }
}

.navigation {
  display: flex;
  gap: 15px;
  margin-top: 10px;
  width: 100%;
  justify-content: center;
}

.nav-btn {
  padding: 10px 20px;
  font-size: 1rem;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
  min-width: 100px;
}

.nav-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  background: #ccc;
}

.nav-btn.primary {
  background-color: #3498db;
  color: white;
}

.nav-btn.secondary {
  background-color: #95a5a6;
  color: white;
}

.nav-btn.submit {
  background-color: #e67e22;
  color: white;
}
</style>

