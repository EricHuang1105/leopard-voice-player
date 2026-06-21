<template>
  <div class="player-container">
    <!-- BGM 背景音樂獨立開關 (右上角) -->
    
    <div class="content-wrapper">
    
    <button class="bgm-toggle-btn" @click="toggleBgm" title="切換背景音樂">
      <!-- 播放中圖示 -->
      <svg v-if="isBgmPlaying" viewBox="0 0 24 24" class="bgm-icon">
        <path fill="currentColor" d="M12 3v10.55c-.59-.34-1.27-.55-2-.55-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4V7h4V3h-6z"/>
      </svg>
      <!-- 靜音/暫停圖示 (帶斜線) -->
      <svg v-else viewBox="0 0 24 24" class="bgm-icon">
        <path fill="currentColor" d="M4.27 3L3 4.27l9 9v.28c-.59-.34-1.27-.55-2-.55-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4v-1.73l4.73 4.73L21 19.73 4.27 3zM14 7h4V3h-6v5.18l2 2z"/>
      </svg>
    </button>

    <button class="leave-btn" @click="handleLeave" title="關閉">
      <svg viewBox="0 0 24 24" class="leave-icon">
        <path fill="currentColor" d="M19 6.41L17.59 5 12 10.59 6.41 5 5 6.41 10.59 12 5 17.59 6.41 19 12 13.41 17.59 19 19 17.59 13.41 12z"/>
      </svg>
    </button>
    
    <!-- 精靈封面區 -->
    <div class="cover-wrapper" :class="{ 'is-playing': isPlaying }">
      
        <div class="cover-logo">
          <img src="/logo.png" alt="LOGO" /> </div>

        <!-- 精靈封面圖片 -->
        <div class="cover-image">
        <img src="/leopard.webp" alt="精靈封面" @error="handleImageError" />
      </div>
    </div>

   <!-- 頂部/背景裝飾區 -->
    <div class="header">
      <h1 class="title">石虎精靈</h1>
    </div>

    <!-- 進度條與時間軸 -->
    <div class="progress-section">
      <input 
        ref="sliderRef" 
        type="range" 
        class="progress-slider" 
        :max="duration" 
        step="any"          :value="currentTime" 
        @input="onSliderDrag" 
        @change="onSliderChange"
      />
    </div>

    <!-- 控制列 (動態音波 + 播放按鈕) -->
    <div class="controls-section">
      <!-- 左側動態音波 -->
      <canvas ref="canvasLeftRef" class="visualizer-canvas" width="80" height="40"></canvas>
      
      <!-- 主語音 播放/暫停按鈕 -->
      <button class="play-btn" @click="togglePlay">
        <svg v-if="!isPlaying" viewBox="0 0 24 24" class="icon">
          <path fill="currentColor" d="M8 5v14l11-7z" />
        </svg>
        <svg v-else viewBox="0 0 24 24" class="icon">
          <path fill="currentColor" d="M6 19h4V5H6v14zm8-14v14h4V5h-4z" />
        </svg>
      </button>

      <!-- 右側動態音波 -->
      <canvas ref="canvasRightRef" class="visualizer-canvas" width="80" height="40"></canvas>
    </div> </div>

    <!-- 隱藏的主語音頻元素 (有音波動效) -->
    <audio 
      ref="audioRef" 
      src="/leopard-voice.wav" 
      crossorigin="anonymous"
      @timeupdate="onTimeUpdate"
      @loadedmetadata="onLoadedMetadata"
      @ended="onEnded"
    ></audio>

    <!-- 隱藏的背景音樂 BGM 元素 (無限循環) -->
    <audio 
      ref="bgmRef" 
      src="/bgm.mp3" 
      loop 
      crossorigin="anonymous"
    ></audio>
  </div> </template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

// --- 狀態與 Refs ---
const isPlaying = ref(false)
const isBgmPlaying = ref(false) // BGM 狀態
const currentTime = ref(0)
const duration = ref(0)
const isDragging = ref(false)

const sliderRef = ref(null)
const audioRef = ref(null) // 主語音
const bgmRef = ref(null)   // 背景音樂
const canvasLeftRef = ref(null)
const canvasRightRef = ref(null)

// 請將這段程式碼加在 <script setup> 裡面，例如放在定義完 refs 變數的下方

onMounted(() => {
  // 將背景音樂音量設定為 20% (數值範圍是 0.0 到 1.0)
  // 如果覺得太大聲或太小聲，可以調整 0.2 這個數字
  if (bgmRef.value) {
    bgmRef.value.volume = 0.1;
  }
document.addEventListener('visibilitychange', handleVisibilityChange)
})

// --- Web Audio API 變數 ---
let audioContext = null
let analyser = null
let dataArray = null
let source = null
let animationId = null

// --- 初始化音頻分析器 (針對主語音) ---
const initAudioVisualizer = () => {
  if (audioContext) return

  audioContext = new (window.AudioContext || window.webkitAudioContext)()
  analyser = audioContext.createAnalyser()
  analyser.fftSize = 64
  const bufferLength = analyser.frequencyBinCount
  dataArray = new Uint8Array(bufferLength)

  source = audioContext.createMediaElementSource(audioRef.value)
  source.connect(analyser)
  analyser.connect(audioContext.destination)

  drawVisualizer()
}

// --- 繪製音波動畫 ---
const drawVisualizer = () => {
  animationId = requestAnimationFrame(drawVisualizer)
  if (!analyser) return
  
  analyser.getByteFrequencyData(dataArray)
  drawCanvas(canvasLeftRef.value, 'left')
  drawCanvas(canvasRightRef.value, 'right')
}

const drawCanvas = (canvas, direction) => {
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  const width = canvas.width
  const height = canvas.height
  
  ctx.clearRect(0, 0, width, height)

  const barWidth = 3
  const gap = 3
  const barCount = 10

  for (let i = 0; i < barCount; i++) {
    const dataIndex = i * 2 
    const percent = dataArray[dataIndex] / 255
    const barHeight = Math.max(percent * height * 0.8, 2) 
    ctx.fillStyle = 'rgba(255, 255, 255, 0.85)'
    
    let x = 0
    if (direction === 'left') {
      x = width - ((i * (barWidth + gap)) + barWidth)
    } else {
      x = i * (barWidth + gap)
    }

    const y = (height - barHeight) / 2
    ctx.beginPath()
    ctx.roundRect(x, y, barWidth, barHeight, 2)
    ctx.fill()
  }
}

// --- 控制 BGM 背景音樂 ---
const toggleBgm = () => {
  if (!bgmRef.value) return
  
  if (isBgmPlaying.value) {
    bgmRef.value.pause()
  } else {
    bgmRef.value.play().catch(e => {
      console.warn('播放 BGM 失敗，可能被瀏覽器阻擋：', e)
    })
  }
  isBgmPlaying.value = !isBgmPlaying.value
}

const handleLeave = () => {
  // 首選：嘗試退回上一頁
  if (window.history.length > 1) {
    window.history.back()
  } else {
    // 備案：如果沒有上一頁 (例如是直接貼網址進來的)，嘗試關閉視窗
    window.close()
  }
}

// --- 監聽分頁切換 (Page Visibility) 邏輯 ---
const handleVisibilityChange = () => {
  // 1. 處理 BGM 背景音樂
  if (bgmRef.value) {
    if (document.hidden) {
      // 切換到手機背景或另一個分頁時：強制暫停音樂
      bgmRef.value.pause()
    } else {
      // 回到目前的語錄網頁時：如果原本是開啟狀態，就自動恢復播放
      if (isBgmPlaying.value) {
        bgmRef.value.play().catch(e => console.warn('自動恢復 BGM 失敗：', e))
      }
    }
  }

  // 2. 順便處理主語音：如果切出畫面時精靈正在講話，就自動幫他暫停
  if (document.hidden && isPlaying.value && audioRef.value) {
    audioRef.value.pause()
    isPlaying.value = false // 讓播放按鈕變回三角形
  }
}

// --- 控制主語音邏輯 ---
const togglePlay = () => {
  if (!audioRef.value) return
  initAudioVisualizer()

  if (audioContext.state === 'suspended') {
    audioContext.resume()
  }

  if (isPlaying.value) {
    audioRef.value.pause()
  } else {
    audioRef.value.play()
  }
  isPlaying.value = !isPlaying.value
}

const onTimeUpdate = (e) => {
  if (!isDragging.value && isPlaying.value) {
    currentTime.value = e.target.currentTime
  }
}

const onLoadedMetadata = (e) => {
  duration.value = e.target.duration
}

const onEnded = () => {
  isPlaying.value = false
  currentTime.value = 0

  if (audioRef.value) {
    audioRef.value.currentTime = 0
  }
  
  if (sliderRef.value) {
    sliderRef.value.value = 0
  }
}

const onSliderDrag = (e) => {
  isDragging.value = true
  currentTime.value = Number(e.target.value)
}

const onSliderChange = (e) => {
  isDragging.value = false
  if (audioRef.value) {
    audioRef.value.currentTime = Number(e.target.value)
  }
}

const formatTime = (time) => {
  if (isNaN(time)) return '0:00'
  const minutes = Math.floor(time / 60)
  const seconds = Math.floor(time % 60)
  return `${minutes}:${seconds.toString().padStart(2, '0')}`
}

const handleImageError = (e) => {
  e.target.src = 'https://via.placeholder.com/300x300?text=Cover+Image'
}

onBeforeUnmount(() => {
  if (animationId) cancelAnimationFrame(animationId)
  if (audioContext) audioContext.close()
document.removeEventListener('visibilitychange', handleVisibilityChange)
})
</script>

<style>

@font-face {
  font-family: 'ChenYuluoyan-2.0-Thin';
  src: url('/ChenYuluoyan-2.0-Thin.ttf') format('truetype');
  font-weight: normal;
  font-style: normal;
}

html, body {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  background-color: #6a704a !important; 
  overflow: hidden;
}
</style>

<style scoped>

.player-container {
  max-width: 400px;
  position: fixed;
  top: 0;
  left: 50%;
  transform: translateX(-50%);
  width: 100%;
  height: 100vh;
  
  background: linear-gradient(145deg, #f5f6f1 0%, #c8ceb3 50%, #6a704a 100%);
  
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  
  /* 保持先前完美的彈性安全間距 */
  padding-top: 40px;
  padding-bottom: 120px; 
  box-sizing: border-box;
  
  font-family: sans-serif;
  color: #2c3319;
  overflow: hidden;
}


.content-wrapper {
  position: relative; /* 讓內部的絕對定位以這個隱形箱子為基準 */
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
}

/* BGM 右上角開關按鈕樣式 */
.bgm-toggle-btn {
  position: absolute;
  top:-20px;
  right: 20px;

  width: 36px;
  height: 36px;
  border-radius: 50%;
  border: 1px solid rgba(44, 51, 25, 0.2);
  background: rgba(255, 255, 255, 0.5);
  backdrop-filter: blur(5px);
  color: #2c3319;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.3s ease;
  z-index: 10;
}

.bgm-toggle-btn:hover {
  background: rgba(255, 255, 255, 0.2);
  transform: scale(1.1);
}

/* 離開按鈕樣式 (左上角) */
.leave-btn {
  position: absolute;
  top: -20px; /* 與 BGM 按鈕保持相同高度 */
  left: 20px; /* 貼齊左邊 */

  width: 36px;
  height: 36px;
  border-radius: 50%;
  
  /* 🌟🌟🌟 請注意！一定要把下面這兩個顏色，改成與目前動物的 BGM 按鈕相同！ 🌟🌟🌟 */
  border: 1px solid rgba(44, 51, 25, 0.2); /* 👈 替換這裡的 rgba */
  color: #2c3319; /* 👈 替換這裡的 hex 色碼 */
  
  background: rgba(255, 255, 255, 0.5);
  backdrop-filter: blur(5px);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.3s ease;
  z-index: 10;
}

.leave-btn:hover {
  background: rgba(255, 255, 255, 0.2);
  transform: scale(1.1);
}

.leave-icon {
  width: 18px;
  height: 18px;
}

.bgm-icon {
  width: 18px;
  height: 18px;
}

/* 頂部裝飾 */
.header {
  text-align: center;
  width: 100%;
  margin-bottom: 50px;
}

.title {
  font-size: 38px;
  letter-spacing: 2px;
  opacity: 0.9;
  font-weight: bold;
  font-family: 'ChenYuluoyan-2.0-Thin', cursive; /* 🌟 這行就是讓字體變化的關鍵！ */
  margin: 0;
}


.cover-wrapper {
  width: 300px;  
  height: 300px; 
  border-radius: 24px; 
  overflow: visible;
  
  border: 2px solid rgba(255, 255, 255, 0.8);

  margin-top: 50px; 
  margin-bottom: 50px;
  background-color:  transparent;
  box-shadow: 0 12px 35px rgba(44, 51, 25, 0.25);
  transition: transform 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* 🌟 新增 LOGO 的絕對定位與尺寸設定 */
.cover-logo {
  position: absolute;
  /* 透過 top 和 right 調整 LOGO 在框框內的右上角位置 */
  top: 60px;    /* 距離框框頂部的距離 */
  right: 60px;  /* 距離框框右側的距離 */
  
  /* 控制 LOGO 圖片的大小，您可以根據實際圖片比例微調 */
  width: 70px;  
  height: auto;
  
  /* 確保 LOGO 永遠浮在最上層，且不參與任何呼吸動畫 */
  z-index: 5; 
  pointer-events: none; /* 讓滑鼠點擊可以穿過 LOGO，不影響操作 */
}

.cover-logo img {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.cover-wrapper.is-playing .cover-image {
  animation: imgBreathe 4s ease-in-out infinite; 
}

.cover-image {
  width: 230px;        /* 👈 縮小圖片寬度（原本是 100%），留下的空間就會變成框 */
  height: 230px;       /* 👈 縮小圖片高度 */
  border-radius: 16px; /* 幫圖片本身也加上一點圓角，看起來更精緻 */
  overflow: visible;    /* 確保圖片呼吸放大時不會超出它自己的邊界 */
}

.cover-image img {
  width: 100%;
  height: 100%;
  object-fit: contain;
  display: block;
}

.quote-section {
  margin-bottom: 40px;
  text-align: center;
  padding: 0 20px; 
}

.quote-text {
  font-family: 'ChenYuluoyan', cursive;
  font-size: 26px;
  line-height: 1.5;
  color: #5c4a3d;
  text-shadow: 0 1px 2px rgba(255, 255, 255, 0.8);
}

.progress-section {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 40px;
  padding: 0 10px;
  box-sizing: border-box;
}

.time {
  font-size: 12px;
  opacity: 0.7;
  width: 40px;
  text-align: center;
}

.progress-slider {
  flex-grow: 1;
  margin: 0 15px;
  -webkit-appearance: none;
  background: transparent;
  height: 4px;
  border-radius: 2px;
  background-color: rgba(44, 51, 25, 0.15);
  outline: none;
  cursor: pointer; 
}

.progress-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: white;
  box-shadow: 0 0 5px rgba(0,0,0,0.3);
  cursor: pointer;
  transition: transform 0.1s;
}

.progress-slider::-webkit-slider-thumb:hover {
  transform: scale(1.3);
}

.controls-section {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 20px;
  margin-bottom: 20px;
}

.play-btn {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  border: 1px solid rgba(44, 51, 25, 0.2);
  background: rgba(255, 255, 255, 0.5);
  backdrop-filter: blur(5px);
  color: #2c3319;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.3s ease;
}

.play-btn:hover {
  background: rgba(255, 255, 255, 0.2);
  transform: scale(1.05);
}

.icon {
  width: 30px;
  height: 30px;
}

@keyframes imgBreathe {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.08); /* 吸氣：讓裡面的圖片稍微放大 8% 塞滿外框，產生微凸感 */
  }
  100% {
    transform: scale(1); /* 吐氣：慢慢縮回原本大小 */
  }
}

/* 🌟 矮螢幕/工具列擠壓時：整體等比例縮小 */
@media (max-height: 740px) {
  /* 1. 稍微減少最外層的上下距離 */
  .player-container {
    padding-top: 50px;
    padding-bottom: calc(180px + env(safe-area-inset-bottom));
  }
  
  /* 2. 外框微微縮小到 280px (原本是 300px) */
  .cover-wrapper {
    width: 280px;
    height: 280px;
    margin-top: 40px;
    margin-bottom: 40px;
  }
  
  /* 3. 裡面的圖片也跟著微縮 */
  .cover-image {
    width: 210px;
    height: 210px;
  }
  
  /* 4. Logo 微縮並稍微調整位置，確保不出框 */
  .cover-logo {
    width: 60px;
    top: 50px;
    right: 60px;
  }
  
  /* 5. 縮小標題與下方的距離 */
  .header {
    margin-bottom: 20px;
  }

  /* 6. 縮小進度條區塊的下方距離 (原本是 40px) */
  .progress-section {
    margin-bottom: 40px;
  }

  /* 7. 縮小主播放按鈕與裡面的圖示 (原本是 60px / 30px) */
  .play-btn {
    width: 50px;
    height: 50px;
  }
  .icon {
    width: 24px;
    height: 24px;
  }

  /* 8. 縮小底部控制區的按鈕間距，讓畫面更緊湊 (原本是 20px) */
  .controls-section {
    gap: 15px;
    margin-bottom: 10px;
  }

  /* 9. 讓旁邊的動態音波也跟著稍微縮小一點比例 */
  .visualizer-canvas {
    width: 60px !important;
    height: 30px !important;
  }
}


</style>

