<script setup>
import { ref, computed, watch, inject, onMounted, onUnmounted } from 'vue'

// ============ Props 定義 ============
const props = defineProps({
  disabled: Boolean, // 是否禁用鍵盤
  currentInput: {    // 當前輸入的內容（用於智能切換鍵盤狀態）
    type: String,
    default: ''
  },
  isCorrectionMode: { // 是否處於修正模式
    type: Boolean,
    default: false
  }
})

// ============ Events 定義 ============
const emit = defineEmits(['input', 'backspace', 'clear'])

// ============ 全域深色模式注入 ============
const isDarkMode = inject('isDarkMode', ref(true)) // 從 App.vue 注入深色模式狀態

// ============ 鍵盤佈局數據 ============

// iOS 動態佈局 - 子音（聲母）佈局 - 21個符號
const consonants = [
  ['ㄅ', 'ㄆ', 'ㄇ', 'ㄈ', 'ㄉ', 'ㄊ', 'ㄋ', 'ㄌ'],
  ['ㄍ', 'ㄎ', 'ㄏ', 'ㄐ', 'ㄑ', 'ㄒ', 'ㄓ', 'ㄔ'],
  ['ㄕ', 'ㄖ', 'ㄗ', 'ㄘ', 'ㄙ']
]

// iOS 動態佈局 - 韻母、介音、聲調佈局
const finals = [
  ['ㄚ', 'ㄛ', 'ㄜ', 'ㄝ', 'ㄞ', 'ㄟ', 'ㄠ', 'ㄡ'],  // 第一排：韻母
  ['ㄢ', 'ㄣ', 'ㄤ', 'ㄥ', 'ㄦ', 'ㄧ', 'ㄨ', 'ㄩ'],  // 第二排：韻母 + 介音
  ['ˊ', 'ˇ', 'ˋ', '˙']                          // 第三排：聲調（不包含一聲，一聲用空白表示）
]

// 傳統全注音鍵盤佈局（顯示所有符號）
const traditionalLayout = [
  ['ㄅ', 'ㄉ', 'ˇ', 'ˋ', 'ㄓ', 'ˊ', '˙', 'ㄚ', 'ㄞ', 'ㄢ', 'ㄦ'],
  ['ㄆ', 'ㄊ', 'ㄍ', 'ㄐ', 'ㄔ', 'ㄗ', 'ㄧ', 'ㄛ', 'ㄟ', 'ㄣ'],
  ['ㄇ', 'ㄋ', 'ㄎ', 'ㄑ', 'ㄕ', 'ㄘ', 'ㄨ', 'ㄜ', 'ㄠ', 'ㄤ'],
  ['ㄈ', 'ㄌ', 'ㄏ', 'ㄒ', 'ㄖ', 'ㄙ', 'ㄩ', 'ㄝ', 'ㄡ', 'ㄥ']
]

// ============ 符號分類集合 ============
// 所有子音（聲母）
const allConsonants = new Set(['ㄅ', 'ㄆ', 'ㄇ', 'ㄈ', 'ㄉ', 'ㄊ', 'ㄋ', 'ㄌ', 'ㄍ', 'ㄎ', 'ㄏ', 'ㄐ', 'ㄑ', 'ㄒ', 'ㄓ', 'ㄔ', 'ㄕ', 'ㄖ', 'ㄗ', 'ㄘ', 'ㄙ'])
// 所有介音
const allMedials = new Set(['ㄧ', 'ㄨ', 'ㄩ'])
// 所有韻母
const allRhymes = new Set(['ㄚ', 'ㄛ', 'ㄜ', 'ㄝ', 'ㄞ', 'ㄟ', 'ㄠ', 'ㄡ', 'ㄢ', 'ㄣ', 'ㄤ', 'ㄥ', 'ㄦ'])
// 所有聲調（包含空白作為一聲）
const allTones = new Set(['ˊ', 'ˇ', 'ˋ', '˙', ' '])

// ============ 狀態管理 ============
const showConsonants = ref(true)  // 是否顯示子音鍵盤（true=子音, false=韻母/介音/聲調）
const isShiftActive = ref(false)  // Shift 鍵是否啟用（用於直接輸入韻母/介音，例如「吳」）
const useDynamicLayout = ref(true) // 是否使用動態 iOS 佈局（true=動態iOS, false=傳統全注音）

// ============ 計算屬性 ============
// 當前顯示的鍵盤行（根據狀態決定顯示子音或韻母）
const currentRows = computed(() => {
  // 如果使用傳統全注音佈局，直接返回
  if (!useDynamicLayout.value) {
    return traditionalLayout
  }

  // iOS 動態佈局：根據狀態切換
  // 如果 Shift 啟用或不在子音模式，顯示韻母/介音/聲調
  if (isShiftActive.value || !showConsonants.value) {
    return finals
  }
  return consonants
})

// ============ 監聽輸入變化 ============
// 根據當前輸入內容智能切換鍵盤狀態（僅在動態佈局時有效）
watch(() => props.currentInput, (newVal) => {
  // 如果使用傳統佈局，不需要自動切換
  if (!useDynamicLayout.value) return

  // 如果輸入為空，顯示子音，Shift 鍵關閉（優先處理，不受 Shift 狀態影響）
  if (!newVal || newVal.length === 0) {
    showConsonants.value = true
    isShiftActive.value = false
    return
  }

  // 如果 Shift 啟用，不自動切換（但空值已經在上面處理了）
  if (isShiftActive.value) return

  const lastChar = newVal.slice(-1)

  if (allConsonants.has(lastChar)) {
    // 最後一個字符是子音 → 顯示韻母/介音/聲調，Shift 鍵自動啟用
    showConsonants.value = false
    isShiftActive.value = true
  } else if (allMedials.has(lastChar) || allRhymes.has(lastChar)) {
    // 最後一個字符是介音或韻母 → 繼續顯示韻母/介音/聲調（等待輸入聲調），Shift 鍵保持啟用
    showConsonants.value = false
    isShiftActive.value = true
  } else if (allTones.has(lastChar)) {
    // 最後一個字符是聲調 → 完成一個字符 → 切回子音，Shift 鍵關閉
    showConsonants.value = true
    isShiftActive.value = false
  } else {
    // 未知字符（例如換行）→ 預設顯示子音，Shift 鍵關閉
    showConsonants.value = true
    isShiftActive.value = false
  }
}, { immediate: true })

// ============ 事件處理函數 ============

// 處理按鍵輸入
const handleInput = (char) => {
  if (props.disabled) return
  
  // 如果按下的是聲調符號（第三排），且不在修正模式下，輸入聲調後自動加上空白，然後切回子音畫面
  if (useDynamicLayout.value && allTones.has(char) && char !== ' ' && !props.isCorrectionMode) {
    emit('input', char + ' ')
    showConsonants.value = true
    isShiftActive.value = false
    return
  }
  
  emit('input', char)
  // 狀態更新由 watcher 處理
}

// 處理退格鍵
const handleBackspace = () => {
  if (props.disabled) return
  
  // 檢查當前輸入的最後一個字符
  const currentInput = props.currentInput || ''
  const lastChar = currentInput.slice(-1)
  
  // 先發送 backspace 事件
  emit('backspace')
  
  // 如果使用動態佈局，根據刪除後的狀態決定鍵盤顯示
  if (useDynamicLayout.value) {
    // 如果最後一個字符是韻母或介音，刪除後應該保留在母音頁
    if (allMedials.has(lastChar) || allRhymes.has(lastChar)) {
      // 刪除後如果還有內容，檢查新的最後一個字符
      const newInput = currentInput.slice(0, -1)
      if (newInput.length > 0) {
        const newLastChar = newInput.slice(-1)
        // 如果新的最後一個字符是子音，應該切回子音頁
        if (allConsonants.has(newLastChar)) {
          showConsonants.value = true
          isShiftActive.value = false
        } else {
          // 否則保留在母音頁
          showConsonants.value = false
          isShiftActive.value = true
        }
      } else {
        // 如果刪除後沒有內容了，切回子音頁
        showConsonants.value = true
        isShiftActive.value = false
      }
    } else {
      // 如果最後一個字符不是韻母或介音（是子音或聲調），刪除後切回子音頁
      showConsonants.value = true
      isShiftActive.value = false
    }
  }
  // 如果使用傳統佈局，狀態更新由 watcher 處理
}

// 處理空白鍵（一聲）
const handleSpace = () => {
  if (props.disabled) return
  // 空白代表一聲，輸入空白後切回子音畫面
  emit('input', ' ')
  if (useDynamicLayout.value) {
    showConsonants.value = true
    isShiftActive.value = false
  }
}

// 切換 Shift 狀態（用於直接輸入韻母/介音，例如「吳」）
const toggleShift = () => {
  if (props.disabled) return
  // 只在動態佈局時有效
  if (!useDynamicLayout.value) return

  // 切換 Shift 狀態
  isShiftActive.value = !isShiftActive.value

  // 如果啟用 Shift，顯示韻母/介音/聲調
  if (isShiftActive.value) {
    showConsonants.value = false
  } else {
    // 如果取消 Shift，切回子音畫面
    showConsonants.value = true
  }
}

// 切換鍵盤佈局（動態 iOS / 傳統全注音）
const toggleLayout = () => {
  useDynamicLayout.value = !useDynamicLayout.value
  // 切換佈局時重置狀態
  showConsonants.value = true
  isShiftActive.value = false
}

</script>

<template>
  <div class="keyboard-wrapper" :class="{ 'dark-mode': isDarkMode }">
    <div class="keyboard">
      <!-- 鍵盤主體區域 -->
      <div class="keys-container">
        <div v-for="(row, rowIndex) in currentRows" :key="rowIndex" class="keyboard-row"
          :class="{ 'centered-row': useDynamicLayout && showConsonants && rowIndex === 2 }">
          <!-- Shift 鍵（在動態佈局時，子音畫面顯示在第三排左側，韻母畫面顯示在第三排左側） -->
          <button v-if="useDynamicLayout && rowIndex === currentRows.length - 1"
            @click="toggleShift" class="key-btn action-key shift-key" :class="{ 'shift-active': isShiftActive }"
            :disabled="disabled">
            <svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
              <path d="M12 4L4 12H8V20H16V12H20L12 4Z" :fill="isShiftActive ? 'currentColor' : 'none'"
                stroke="currentColor" stroke-width="2" />
            </svg>
          </button>

          <!-- 鍵盤按鍵 -->
          <button v-for="char in row" :key="char" @click="handleInput(char)" 
            class="key-btn" 
            :class="{ 'tone-key': useDynamicLayout && !showConsonants && rowIndex === 2 && allTones.has(char) }"
            :disabled="disabled">
            {{ char }}
          </button>

          <!-- 退格鍵（在最後一排的最右邊） -->
          <button v-if="rowIndex === currentRows.length - 1" @click="handleBackspace"
            class="key-btn action-key backspace-key" :disabled="disabled">
            <svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
              <path
                d="M22 3H7C6.31 3 5.77 3.35 5.6 3.6L0 12L5.6 20.4C5.77 20.65 6.31 21 7 21H22C23.1 21 24 20.1 24 19V5C24 3.9 23.1 3 22 3ZM19 15.59L17.59 17L14 13.41L10.41 17L9 15.59L12.59 12L9 8.41L10.41 7L14 10.59L17.59 7L19 8.41L15.41 12L19 15.59Z"
                fill="currentColor" />
            </svg>
          </button>
        </div>
      </div>

      <!-- 底部功能列 -->
      <div class="bottom-row">
        <!-- 地球按鈕（切換鍵盤佈局：動態 iOS / 傳統全注音） -->
        <button @click="toggleLayout" class="action-btn globe" :disabled="disabled"
          :title="useDynamicLayout ? '切換到傳統全注音' : '切換到動態 iOS 注音'">
          <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"
            stroke-linecap="round" stroke-linejoin="round">
            <circle cx="12" cy="12" r="10"></circle>
            <line x1="2" y1="12" x2="22" y2="12"></line>
            <path d="M12 2a15.3 15.3 0 0 1 4 10 15.3 15.3 0 0 1-4 10 15.3 15.3 0 0 1-4-10 15.3 15.3 0 0 1 4-10z"></path>
          </svg>
        </button>

        <!-- 空白鍵（一聲） -->
        <button @click="handleSpace" class="space-btn" :disabled="disabled">空白</button>

        <!-- 換行鍵 -->
        <button @click="emit('input', '\n')" class="action-btn return" :disabled="disabled">
          換行
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* ============ 鍵盤容器 ============ */
.keyboard-wrapper {
  width: 100%;
  background: #d1d5db;
  /* 淺色模式背景（iOS 灰） */
  padding-bottom: env(safe-area-inset-bottom);
  transition: background 0.3s ease;
  /* 禁用雙擊放大 */
  touch-action: manipulation;
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  user-select: none;
}

/* 深色模式背景 */
.keyboard-wrapper.dark-mode {
  background: #1a1a1a;
}

.keyboard {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 6px;
  width: 100%;
  max-width: 100%;
  box-sizing: border-box;
  user-select: none;
  /* 禁用雙擊放大 */
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
}

/* ============ 按鍵區域 ============ */
.keys-container {
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 5px 0;
  /* 禁用雙擊放大 */
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
}

.keyboard-row {
  display: flex;
  justify-content: center;
  gap: 6px;
}



/* ============ 按鍵樣式 ============ */
.key-btn {
  flex: 1;
  max-width: 42px;
  height: 46px;
  font-size: 1.5rem;
  border: none;
  border-radius: 5px;
  background: white;
  box-shadow: 0 1px 0px rgba(0, 0, 0, 0.3);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  color: #000;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  touch-action: manipulation;
  transition: background 0.1s ease;
  /* 禁用雙擊放大和點擊高亮 */
  -webkit-tap-highlight-color: transparent;
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  user-select: none;
}

/* 深色模式按鍵 */
.dark-mode .key-btn {
  background: #3a3a3c;
  color: #fff;
  box-shadow: 0 1px 0px rgba(255, 255, 255, 0.1);
}

.key-btn:active:not(:disabled) {
  background: #e5e5e5;
}

.dark-mode .key-btn:active:not(:disabled) {
  background: #5a5a5c;
}

/* 功能鍵（Shift、退格等） */
.action-key {
  background: #b4b9c2;
  color: #000;
  max-width: 50px;
  flex: 0 0 auto;
}

/* Shift 鍵固定寬度，保持靠左 */
.shift-key {
  flex: 0 0 50px;
  max-width: 50px;
}

.dark-mode .action-key {
  background: #2c2c2e;
  color: #fff;
}

.action-key:active:not(:disabled) {
  background: #9ca3ac;
}

.dark-mode .action-key:active:not(:disabled) {
  background: #3c3c3e;
}

/* Shift 鍵啟用狀態 */
.shift-key.shift-active {
  background: white;
  color: #000;
}

.dark-mode .shift-key.shift-active {
  background: #fff;
  color: #000;
}

/* 退格鍵固定寬度，保持靠右 */
.backspace-key {
  flex: 0 0 50px;
  max-width: 50px;
}

/* 退格鍵圖標 */
.backspace-key svg {
  width: 22px;
  height: 22px;
}

/* 母音頁第三排的聲調按鈕加寬 */
.tone-key {
  flex: 1.5;
  max-width: 60px;
  min-width: 50px;
}

/* ============ 底部功能列 ============ */
.bottom-row {
  display: flex;
  gap: 6px;
  padding: 0 4px;
  margin-top: 4px;
  /* 禁用雙擊放大 */
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
}

.action-btn {
  min-width: 42px;
  height: 46px;
  border: none;
  border-radius: 5px;
  background: #b4b9c2;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #000;
  font-size: 1rem;
  box-shadow: 0 1px 0px rgba(0, 0, 0, 0.3);
  transition: background 0.1s ease;
  /* 禁用雙擊放大和點擊高亮 */
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  user-select: none;
}

.dark-mode .action-btn {
  background: #2c2c2e;
  color: #fff;
  box-shadow: 0 1px 0px rgba(255, 255, 255, 0.1);
}

.action-btn:active:not(:disabled) {
  background: #9ca3ac;
}

.dark-mode .action-btn:active:not(:disabled) {
  background: #3c3c3e;
}

/* 地球按鈕 */
.action-btn.globe {
  width: 46px;
}

/* 換行鍵 */
.action-btn.return {
  width: 80px;
}

/* 空白鍵 */
.space-btn {
  flex: 1;
  height: 46px;
  font-size: 1rem;
  border: none;
  border-radius: 5px;
  background: white;
  box-shadow: 0 1px 0px rgba(0, 0, 0, 0.3);
  cursor: pointer;
  color: #000;
  transition: background 0.1s ease;
  /* 禁用雙擊放大和點擊高亮 */
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  user-select: none;
}

.dark-mode .space-btn {
  background: #3a3a3c;
  color: #fff;
  box-shadow: 0 1px 0px rgba(255, 255, 255, 0.1);
}

.space-btn:active:not(:disabled) {
  background: #e5e5e5;
}

.dark-mode .space-btn:active:not(:disabled) {
  background: #5a5a5c;
}

/* ============ 響應式調整 ============ */
/* 針對較多按鍵的行（例如韻母第一排有 8 個） */
.keyboard-row:has(> .key-btn:nth-child(8)) .key-btn {
  max-width: 40px;
  font-size: 1.4rem;
}

/* 針對更多按鍵的行（例如韻母第二排有 8 個 + 介音） */
.keyboard-row:has(> .key-btn:nth-child(9)) .key-btn {
  max-width: 38px;
  font-size: 1.35rem;
}

/* 傳統佈局的 11 鍵行 */
.keyboard-row:has(> .key-btn:nth-child(11)) .key-btn {
  max-width: 34px;
  font-size: 1.3rem;
}

/* 小螢幕適配 */
@media (max-width: 390px) {
  .key-btn {
    height: 42px;
    font-size: 1.4rem;
  }

  .keyboard-row:has(> .key-btn:nth-child(8)) .key-btn {
    max-width: 36px;
    font-size: 1.3rem;
  }

  .keyboard-row:has(> .key-btn:nth-child(9)) .key-btn {
    max-width: 34px;
    font-size: 1.25rem;
  }

  .keyboard-row:has(> .key-btn:nth-child(11)) .key-btn {
    max-width: 30px;
    font-size: 1.2rem;
  }
}
</style>
