<script setup>
import { ref, provide } from 'vue'
import StartView from './components/StartView.vue'
import QuizView from './components/QuizView.vue'
import ResultView from './components/ResultView.vue'

// ============ 視圖狀態 ============
const currentView = ref('start') // 當前顯示的視圖（start/quiz/result）
const score = ref(0)             // 考試分數
const totalQuestions = 3         // 總題數
const timeLimit = 10 * 60        // 時間限制（秒）
const pointsPerQuestion = Math.ceil(100 / totalQuestions) // 每題分數（無條件進位）
const totalTimeSpent = ref(0)    // 總作答時間（秒）
const averageTimePerQuestion = ref(0) // 平均每題作答時間（秒）

// ============ 全域深色模式 ============
const isDarkMode = ref(true) // 深色模式開關（預設為深色）

// 提供給所有子組件使用
provide('isDarkMode', isDarkMode)

// 切換深色/淺色模式
const toggleDarkMode = () => {
  isDarkMode.value = !isDarkMode.value
}

// ============ 視圖切換函數 ============
const startQuiz = () => {
  currentView.value = 'quiz'
}

const finishQuiz = (finalScore, totalTime, averageTime) => {
  score.value = finalScore
  totalTimeSpent.value = totalTime || 0
  averageTimePerQuestion.value = averageTime || 0
  currentView.value = 'result'
}

const restartQuiz = () => {
  currentView.value = 'start'
  score.value = 0
  totalTimeSpent.value = 0
  averageTimePerQuestion.value = 0
}
</script>

<template>
  <div class="app-container" :class="{ 'dark-mode': isDarkMode }">
    <!-- 全域深色模式切換按鈕（右上角） -->
    <button @click="toggleDarkMode" class="theme-toggle-btn" :title="isDarkMode ? '切換到淺色模式' : '切換到深色模式'">
      <svg v-if="isDarkMode" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor"
        stroke-width="2">
        <!-- 月亮圖標（深色模式） -->
        <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"></path>
      </svg>
      <svg v-else width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <!-- 太陽圖標（淺色模式） -->
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

    <!-- 視圖內容 -->
    <StartView v-if="currentView === 'start'" :total-questions="totalQuestions" :time-limit="timeLimit"
      :points-per-question="pointsPerQuestion" @start="startQuiz" />
    <QuizView v-else-if="currentView === 'quiz'" :questions-count="totalQuestions" :time-limit="timeLimit"
      :points-per-question="pointsPerQuestion" @finish="finishQuiz" />
    <ResultView v-else-if="currentView === 'result'" :score="score" :total-questions="totalQuestions"
      :total-time-spent="totalTimeSpent" :average-time-per-question="averageTimePerQuestion"
      @restart="restartQuiz" />
  </div>
</template>

<style>
/* ============ 全域樣式 ============ */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  transition: background-color 0.3s ease, color 0.3s ease;
}
</style>

<style scoped>
/* ============ 應用容器 ============ */
.app-container {
  width: 100%;
  min-height: 100vh;
  background-color: #f0f2f5;
  overflow-y: auto;
  position: relative;
  transition: background-color 0.3s ease, color 0.3s ease;
}

/* 深色模式 */
.app-container.dark-mode {
  background-color: #000;
  color: #fff;
}

/* ============ 深色模式切換按鈕 ============ */
.theme-toggle-btn {
  position: fixed;
  top: 10px;
  left: 10px;
  width: 40px;
  height: 40px;
  border-radius: 8px;
  border: 1px solid rgba(0, 0, 0, 0.1);
  background: rgba(255, 255, 255, 0.95);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  cursor: pointer;
  display: none;
  /* 隱藏全域按鈕，改用 QuizView 中的按鈕 */
  align-items: center;
  justify-content: center;
  z-index: 1000;
  transition: all 0.2s ease;
  color: #555;
}

.dark-mode .theme-toggle-btn {
  background: rgba(58, 58, 60, 0.95);
  color: #ccc;
  border-color: rgba(255, 255, 255, 0.1);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}

.theme-toggle-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
  background: rgba(255, 255, 255, 1);
}

.dark-mode .theme-toggle-btn:hover {
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.4);
  background: rgba(68, 68, 70, 0.95);
}

.theme-toggle-btn:active {
  transform: translateY(0);
}
</style>
