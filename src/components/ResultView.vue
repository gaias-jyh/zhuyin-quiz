<script setup>
import { inject, ref } from 'vue'

defineProps({
  score: Number,
  totalQuestions: Number,
  totalTimeSpent: {
    type: Number,
    default: 0
  },
  averageTimePerQuestion: {
    type: Number,
    default: 0
  }
})

defineEmits(['restart'])

// ============ 全域深色模式注入 ============
const isDarkMode = inject('isDarkMode', ref(true))

// 格式化時間顯示（秒轉換為分:秒）
const formatTime = (seconds) => {
  const mins = Math.floor(seconds / 60)
  const secs = seconds % 60
  if (mins > 0) {
    return `${mins} 分 ${secs} 秒`
  }
  return `${secs} 秒`
}
</script>

<template>
  <div class="result-view" :class="{ 'dark-mode': isDarkMode }">
    <h2>考試結束</h2>
    <div class="score-card">
      <div class="score-label">你的得分</div>
      <div class="score-value">{{ score }}</div>
      <div class="total-label">滿分 100 分</div>
    </div>

    <div class="message">
      <p v-if="score === 100">太厲害了！滿分！ 🎉</p>
      <p v-else-if="score >= 80">很棒的成績！繼續保持！ 👍</p>
      <p v-else-if="score >= 60">及格了！再接再厲！ 💪</p>
      <p v-else>加油！多練習一定會進步的！ 📚</p>
    </div>

    <div class="time-stats">
      <div class="time-stat-item">
        <span class="time-label">總作答時間：</span>
        <span class="time-value">{{ formatTime(totalTimeSpent) }}</span>
      </div>
      <div class="time-stat-item">
        <span class="time-label">平均每題：</span>
        <span class="time-value">{{ formatTime(averageTimePerQuestion) }}</span>
      </div>
    </div>

    <button @click="$emit('restart')" class="restart-btn">重新開始</button>
  </div>
</template>

<style scoped>
.result-view {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  padding: 2rem;
  text-align: center;
  transition: background-color 0.3s ease, color 0.3s ease;
}

/* 深色模式 */
.result-view.dark-mode h2 {
  color: #fff;
}

.result-view.dark-mode .score-card {
  background: #2a2a2a;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
}

.result-view.dark-mode .score-label {
  color: #aaa;
}

.result-view.dark-mode .total-label {
  color: #777;
}

.result-view.dark-mode .message {
  color: #ccc;
}

h2 {
  font-size: 2.5rem;
  color: #2c3e50;
  margin-bottom: 2rem;
}

.score-card {
  background: white;
  padding: 3rem;
  border-radius: 20px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
  margin-bottom: 2rem;
  min-width: 300px;
}

.score-label {
  font-size: 1.2rem;
  color: #666;
  margin-bottom: 1rem;
}

.score-value {
  font-size: 6rem;
  font-weight: bold;
  color: #42b883;
  line-height: 1;
  margin-bottom: 1rem;
}

.total-label {
  font-size: 1rem;
  color: #999;
}

.message {
  font-size: 1.5rem;
  color: #2c3e50;
  margin-bottom: 2rem;
  min-height: 2em;
}

.time-stats {
  background: #f8f9fa;
  padding: 1.5rem;
  border-radius: 12px;
  margin-bottom: 2rem;
  min-width: 300px;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.time-stat-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 1.1rem;
}

.time-label {
  color: #666;
}

.time-value {
  color: #42b883;
  font-weight: bold;
}

.dark-mode .time-stats {
  background: #2a2a2a;
}

.dark-mode .time-label {
  color: #aaa;
}

.dark-mode .time-value {
  color: #42b883;
}

.restart-btn {
  font-size: 1.2rem;
  padding: 1rem 2.5rem;
  background-color: #34495e;
  color: white;
  border: none;
  border-radius: 50px;
  cursor: pointer;
  transition: all 0.2s;
}

.restart-btn:hover {
  background-color: #2c3e50;
  transform: translateY(-2px);
}
</style>
