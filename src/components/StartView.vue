<script setup>
import { inject, ref } from 'vue'

defineProps({
  totalQuestions: {
    type: Number,
    required: true
  },
  timeLimit: {
    type: Number,
    required: true
  },
  pointsPerQuestion: {
    type: Number,
    required: true
  }
})
const emit = defineEmits(['start'])

// ============ 全域深色模式注入 ============
const isDarkMode = inject('isDarkMode', ref(true))

// ============ 全螢幕 API ============
const requestFullscreen = () => {
  const element = document.documentElement
  
  // 嘗試不同的全螢幕 API（處理瀏覽器相容性）
  if (element.requestFullscreen) {
    element.requestFullscreen().catch(err => {
      console.log('無法進入全螢幕模式:', err)
    })
  } else if (element.webkitRequestFullscreen) {
    // Safari
    element.webkitRequestFullscreen()
  } else if (element.mozRequestFullScreen) {
    // Firefox
    element.mozRequestFullScreen()
  } else if (element.msRequestFullscreen) {
    // IE/Edge
    element.msRequestFullscreen()
  }
}

// 處理開始考試
const handleStart = () => {
  // 先請求全螢幕
  requestFullscreen()
  // 然後發送開始事件
  emit('start')
}
</script>

<template>
  <div class="start-view" :class="{ 'dark-mode': isDarkMode }">
    <h1>注音大會考</h1>
    <p>準備好挑戰你的注音能力了嗎？</p>
    <div class="info">
      <p>總共 {{ totalQuestions }} 題</p>
      <p>限時 {{ Math.floor(timeLimit / 60) }} 分鐘</p>
      <p>每題 {{ pointsPerQuestion }} 分（答錯扣分）</p>
    </div>
    <button @click="handleStart" class="start-btn">開始考試</button>
  </div>
</template>

<style scoped>
.start-view {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  text-align: center;
  padding: 2rem;
  transition: background-color 0.3s ease, color 0.3s ease;
}

/* 深色模式 */
.start-view.dark-mode h1 {
  color: #fff;
}

.start-view.dark-mode p {
  color: #ccc;
}

.start-view.dark-mode .info {
  background: #2a2a2a;
  color: #ccc;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}

h1 {
  font-size: 3rem;
  color: #2c3e50;
  margin-bottom: 1rem;
}

.info {
  margin: 2rem 0;
  font-size: 1.2rem;
  color: #666;
  background: #f8f9fa;
  padding: 1.5rem;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.info p {
  margin: 0.5rem 0;
}

.start-btn {
  font-size: 1.5rem;
  padding: 1rem 3rem;
  background-color: #42b883;
  color: white;
  border: none;
  border-radius: 50px;
  cursor: pointer;
  transition: transform 0.2s, background-color 0.2s;
  box-shadow: 0 4px 12px rgba(66, 184, 131, 0.3);
}

.start-btn:hover {
  background-color: #3aa876;
  transform: translateY(-2px);
}

.start-btn:active {
  transform: translateY(0);
}
</style>
