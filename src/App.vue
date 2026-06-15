<template>
  <div class="player-container">
    <!-- BGM 背景音樂獨立開關 (右上角) -->
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

    <!-- 頂部/背景裝飾區 -->
    <div class="header">
      <h1 class="title">Echoes of Deepspace</h1>
    </div>

    <!-- 精靈封面區 -->
    <div class="cover-wrapper" :class="{ 'is-playing': isPlaying }">
      <div class="cover-image">
        <!-- 精靈封面圖片 -->
        <img src="/pangolin.webp" alt="精靈封面" @error="handleImageError" />
      </div>
    </div>

    <!-- 專屬微醺語錄 -->
    <div class="quote-section">
      <p class="quote-text">「在微醺的氣泡裡，藏著我想對你說的秘密。」</p>
    </div>

    <!-- 進度條與時間軸 -->
    <div class="progress-section">
      <span class="time">{{ formatTime(currentTime) }}</span>
      <input 
        type="range" 
        class="progress-slider" 
        :max="duration" 
        :value="currentTime" 
        @input="onSliderDrag" 
        @change="onSliderChange"
      />
      <span class="time">{{ formatTime(duration) }}</span>
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
    </div>

    <!-- 隱藏的主語音頻元素 (有音波動效) -->
    <audio 
      ref="audioRef" 
      src="/audio/civet.mp3" 
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
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

// --- 狀態與 Refs ---
const isPlaying = ref(false)
const isBgmPlaying = ref(false) // BGM 狀態
const currentTime = ref(0)
const duration = ref(0)
const isDragging = ref(false)

const audioRef = ref(null) // 主語音
const bgmRef = ref(null)   // 背景音樂
const canvasLeftRef = ref(null)
const canvasRightRef = ref(null)

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
  if (!isDragging.value) {
    currentTime.value = e.target.currentTime
  }
}

const onLoadedMetadata = (e) => {
  duration.value = e.target.duration
}

const onEnded = () => {
  isPlaying.value = false
  currentTime.value = 0
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
})
</script>

<style scoped>
@font-face {
  font-family: 'ChenYuluoyan';
  src: url('/fonts/ChenYuluoyan-2.0-Thin.ttf') format('truetype');
  font-weight: normal;
  font-style: normal;
}

.player-container {
  max-width: 400px;
  min-height: 100vh;
  margin: 0 auto;
  background: linear-gradient(145deg, #a8c0cf, #62829c, #2a4c6a);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 30px 20px;
  box-sizing: border-box;
  font-family: sans-serif;
  color: white;
  position: relative;
  overflow: hidden;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.2);
}

/* BGM 右上角開關按鈕樣式 */
.bgm-toggle-btn {
  position: absolute;
  top: 30px;
  right: 25px;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  border: 1px solid rgba(255, 255, 255, 0.5);
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(5px);
  color: white;
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

.bgm-icon {
  width: 18px;
  height: 18px;
}

.header {
  position: absolute;
  top: 40px;
  text-align: center;
  width: 100%;
}

.title {
  font-size: 14px;
  letter-spacing: 2px;
  opacity: 0.8;
  font-weight: 300;
}

.cover-wrapper {
  width: 260px;
  height: 260px;
  border-radius: 20px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  margin-bottom: 30px;
  margin-top: 50px; /* 稍微往下推，避免跟按鈕或標題擠在一起 */
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  background-color: rgba(255, 255, 255, 0.1);
}

.cover-wrapper.is-playing {
  box-shadow: 0 15px 40px rgba(255, 255, 255, 0.2);
  transform: scale(1.02);
}

.cover-image, .cover-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
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
  color: #ffffff;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
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
  background-color: rgba(255, 255, 255, 0.3);
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
}

.play-btn {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  border: 1px solid rgba(255, 255, 255, 0.5);
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(5px);
  color: white;
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
</style>