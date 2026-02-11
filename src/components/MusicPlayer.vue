<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'

// === 音乐列表配置 ===
// 你只需要在这里添加或修改歌曲
const playlist = ref([
  {
    title: 'Annie'+'s' +'Wonderland',
    artist: 'Bandari',
    src: '/music/wonderLand.flac',
    cover: ''
  }
])

const currentTrackIndex = ref(0)
const isPlaying = ref(false)
const audio = ref(null)
const currentTime = ref(0)
const duration = ref(0)
const isMinimized = ref(false) // 是否最小化
const showPlaylist = ref(false) // 是否显示歌单列表
// 新增：记录当前歌曲是否播放失败，防止无限重试
const playErrorCount = ref(0)

const currentTrack = computed(() => playlist.value[currentTrackIndex.value])

// 初始化音频
// onMounted 已经在下方被重写，删除这里的旧 onMounted
// onUnmounted 也已经在下方被重写，删除这里的旧 onUnmounted

// 处理播放错误
const handleAudioError = (e) => {
  console.error('Audio Error:', e)
  isPlaying.value = false // 停止播放状态，不进行自动切歌
}

// 播放控制
const togglePlay = () => {
  if (!audio.value.src) return

  if (isPlaying.value) {
    audio.value.pause()
  } else {
    // 重置错误计数，允许再次尝试
    playErrorCount.value = 0
    audio.value.play().catch(e => console.error("播放失败，可能是浏览器策略限制", e))
  }
  isPlaying.value = !isPlaying.value
}

const playTrack = (index) => {
  if (currentTrackIndex.value === index) return // 点击同一首不重载
  currentTrackIndex.value = index
  loadTrack()
  showPlaylist.value = false // 选中后关闭列表
}

const nextTrack = () => {
  currentTrackIndex.value = (currentTrackIndex.value + 1) % playlist.value.length
  loadTrack()
}

const prevTrack = () => {
  currentTrackIndex.value = (currentTrackIndex.value - 1 + playlist.value.length) % playlist.value.length
  loadTrack()
}

const loadTrack = () => {
  if (!audio.value) return

  // 切歌时重置错误计数
  playErrorCount.value = 0

  const wasPlaying = isPlaying.value
  audio.value.src = currentTrack.value.src
  audio.value.load()

  // 只有当之前正在播放时，才继续播放（取消了强制自动播放）
  if (wasPlaying) {
    audio.value.play().catch(e => {
      console.error("Auto play failed:", e)
      isPlaying.value = false
    })
  } else {
    // 如果之前没在播放，切歌后保持暂停状态
    isPlaying.value = false
  }
}

const updateTime = () => {
  currentTime.value = audio.value.currentTime
}

const updateDuration = () => {
  duration.value = audio.value.duration
}

const seek = (e) => {
  const percent = e.target.value / 100
  audio.value.currentTime = percent * duration.value
}
// ... formatTime 和 progressPercent 保持不变
const formatTime = (seconds) => {
  if (!seconds) return '0:00'
  const m = Math.floor(seconds / 60)
  const s = Math.floor(seconds % 60)
  return `${m}:${s.toString().padStart(2, '0')}`
}

// 进度条百分比
const progressPercent = computed(() => {
  if (!duration.value) return 0
  return (currentTime.value / duration.value) * 100
})

// === 持久化逻辑 ===
const STORAGE_KEY = 'youth-blog-player-state'

// 保存状态到 localStorage
const saveState = () => {
  const state = {
    currentTrackIndex: currentTrackIndex.value,
    currentTime: currentTime.value,
    isMinimized: isMinimized.value,
    // 不保存 isPlaying，因为自动播放通常会被浏览器阻止，且用户下次进来可能不想被打扰
  }
  localStorage.setItem(STORAGE_KEY, JSON.stringify(state))
}

// 恢复状态
const restoreState = () => {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved) {
    try {
      const state = JSON.parse(saved)
      // 恢复歌单索引 (确保索引有效)
      if (typeof state.currentTrackIndex === 'number' && state.currentTrackIndex < playlist.value.length) {
        currentTrackIndex.value = state.currentTrackIndex
      }
      // 恢复播放进度
      if (typeof state.currentTime === 'number') {
        // 设置一个标志，等待 metadata 加载完成后再设置 currentTime
        // 注意：audio.value.currentTime 必须在 src 设置后才能设置，且最好在 loadedmetadata 事件后
        const savedTime = state.currentTime
        const restoreTimeHandler = () => {
          audio.value.currentTime = savedTime
          audio.value.removeEventListener('loadedmetadata', restoreTimeHandler)
        }
        audio.value.addEventListener('loadedmetadata', restoreTimeHandler)
      }
      // 恢复最小化状态
      if (typeof state.isMinimized === 'boolean') {
        isMinimized.value = state.isMinimized
      }
    } catch (e) {
      console.error('Failed to restore player state:', e)
    }
  }
}

// 监听状态变化并保存 (使用 watch 监听关键数据)
import { watch } from 'vue'
watch([currentTrackIndex, isMinimized], () => {
  saveState()
})

// 定时保存进度 (每 5 秒保存一次，避免过于频繁写入 localStorage)
let saveInterval
onMounted(() => {
  audio.value = new Audio()
  audio.value.src = currentTrack.value.src

  // 恢复状态
  restoreState()

  // 监听事件
  audio.value.addEventListener('timeupdate', updateTime)
  audio.value.addEventListener('loadedmetadata', updateDuration)
  audio.value.addEventListener('ended', nextTrack)
  audio.value.addEventListener('error', handleAudioError)
  
  // 启动定时保存
  saveInterval = setInterval(() => {
    if (isPlaying.value) {
      saveState()
    }
  }, 5000)
})

onUnmounted(() => {
  if (saveInterval) clearInterval(saveInterval)
  // 离开时保存一次最终状态
  saveState()
  
  if (audio.value) {
    audio.value.pause()
    audio.value.removeEventListener('timeupdate', updateTime)
    audio.value.removeEventListener('loadedmetadata', updateDuration)
    audio.value.removeEventListener('ended', nextTrack)
    audio.value.removeEventListener('error', handleAudioError)
  }
})
</script>

<template>
  <div class="music-player-container">
    <!-- 歌单列表弹窗 -->
    <Transition name="slide-up">
      <div v-if="showPlaylist && !isMinimized" class="playlist-panel">
        <div class="panel-header">
          <h3>播放列表 ({{ playlist.length }})</h3>
          <button class="close-panel" @click="showPlaylist = false">×</button>
        </div>
        <ul class="track-list">
          <li
            v-for="(track, index) in playlist"
            :key="index"
            :class="{ active: currentTrackIndex === index }"
            @click="playTrack(index)"
          >
            <span class="track-index">{{ index + 1 }}</span>
            <div class="track-info-mini">
              <span class="t-title">{{ track.title }}</span>
              <span class="t-artist">{{ track.artist }}</span>
            </div>
            <span v-if="currentTrackIndex === index" class="playing-indicator">♫</span>
          </li>
        </ul>
      </div>
    </Transition>

    <div class="music-player" :class="{ minimized: isMinimized }">
      <!-- 最小化按钮 -->
      <button class="toggle-btn" @click="isMinimized = !isMinimized" :title="isMinimized ? '展开' : '收起'">
        <span v-if="isMinimized">🎵</span>
        <span v-else>✖</span>
      </button>

      <!-- 最小化状态 -->
      <div v-if="isMinimized" class="mini-view" @click="isMinimized = false">
        <div class="vinyl" :class="{ rotating: isPlaying }">
          <img :src="currentTrack.cover || '@/assets/img/head.png'" alt="cover">
        </div>
      </div>

      <!-- 完整面板 -->
      <div v-else class="full-view">
        <div class="cover-wrapper">
          <div class="vinyl" :class="{ rotating: isPlaying }">
             <!-- 如果没有封面，显示默认占位 -->
             <img v-if="currentTrack.cover" :src="currentTrack.cover" alt="cover">
             <div v-else class="default-cover">🎵</div>
          </div>
        </div>

        <div class="info">
          <div class="track-name">{{ currentTrack.title }}</div>
          <div class="artist">{{ currentTrack.artist }}</div>
        </div>

        <div class="controls">
          <button @click="prevTrack" class="ctrl-btn" title="上一首">⏮</button>
          <button @click="togglePlay" class="ctrl-btn play-btn" title="播放/暂停">
            {{ isPlaying ? '⏸' : '▶' }}
          </button>
          <button @click="nextTrack" class="ctrl-btn" title="下一首">⏭</button>
          <button
            @click="showPlaylist = !showPlaylist"
            class="ctrl-btn list-btn"
            :class="{ active: showPlaylist }"
            title="歌单列表"
          >
            📜
          </button>
        </div>

        <div class="progress-bar">
          <span class="time">{{ formatTime(currentTime) }}</span>
          <input
            type="range"
            min="0"
            max="100"
            :value="progressPercent"
            @input="seek"
          >
          <span class="time">{{ formatTime(duration) }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
.music-player-container {
  position: fixed;
  bottom: 20px;
  right: 20px;
  z-index: 1000;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
}

.music-player {
  background: rgba(var(--color-bg-primary), 0.85);
  backdrop-filter: blur(12px);
  border: 1px solid rgba(var(--color-border-primary), 0.2);
  border-radius: 20px;
  padding: 15px;
  width: 280px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.15);
  transition: all 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
  color: rgb(var(--color-text-primary));
  position: relative;

  &.minimized {
    width: 60px;
    height: 60px;
    padding: 0;
    border-radius: 50%;
    overflow: hidden;
    cursor: pointer;

    .toggle-btn {
      display: none; /* 最小化时不显示关闭按钮 */
    }
  }
}

/* 播放列表面板 */
.playlist-panel {
  width: 280px;
  max-height: 300px;
  background: rgba(var(--color-bg-primary), 0.95);
  backdrop-filter: blur(12px);
  border: 1px solid rgba(var(--color-border-primary), 0.2);
  border-radius: 16px;
  margin-bottom: 15px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.panel-header {
  padding: 12px 15px;
  border-bottom: 1px solid rgba(var(--color-border-primary), 0.1);
  display: flex;
  justify-content: space-between;
  align-items: center;

  h3 {
    font-size: 14px;
    margin: 0;
    color: rgb(var(--color-text-primary));
  }

  .close-panel {
    background: transparent;
    border: none;
    color: rgb(var(--color-text-primary));
    cursor: pointer;
    font-size: 18px;
    opacity: 0.6;
    &:hover { opacity: 1; }
  }
}

.track-list {
  list-style: none;
  padding: 0;
  margin: 0;
  overflow-y: auto;
  flex: 1;

  li {
    padding: 10px 15px;
    display: flex;
    align-items: center;
    gap: 10px;
    cursor: pointer;
    transition: background 0.2s;
    border-bottom: 1px solid rgba(var(--color-border-primary), 0.05);

    &:hover {
      background: rgba(var(--color-bg-secondary), 0.5);
    }

    &.active {
      background: rgba(var(--color-bg-secondary), 0.8);

      .t-title {
        color: #409eff;
        font-weight: bold;
      }
    }
  }

  .track-index {
    font-size: 12px;
    opacity: 0.5;
    width: 20px;
  }

  .track-info-mini {
    flex: 1;
    display: flex;
    flex-direction: column;
    overflow: hidden;

    .t-title {
      font-size: 13px;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
      color: rgb(var(--color-text-primary));
    }

    .t-artist {
      font-size: 11px;
      opacity: 0.6;
      color: rgb(var(--color-text-primary));
    }
  }

  .playing-indicator {
    color: #409eff;
    font-size: 12px;
  }
}

/* 动画 */
.slide-up-enter-active,
.slide-up-leave-active {
  transition: all 0.3s ease;
}

.slide-up-enter-from,
.slide-up-leave-to {
  opacity: 0;
  transform: translateY(20px);
}

/* ... (保留原有播放器内部样式) ... */
.toggle-btn {
  position: absolute;
  top: 8px;
  right: 8px;
  background: transparent;
  border: none;
  font-size: 12px;
  cursor: pointer;
  color: rgb(var(--color-text-primary));
  opacity: 0.6;
  z-index: 10;

  &:hover {
    opacity: 1;
  }
}

/* 最小化视图 */
.mini-view {
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;

  .vinyl {
    width: 100%;
    height: 100%;

    img {
      width: 100%;
      height: 100%;
      object-fit: cover;
    }
  }
}

/* 完整视图 */
.full-view {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
}

.cover-wrapper {
  margin-top: 10px;
}

.vinyl {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0,0,0,0.2);
  border: 4px solid #1a1a1a;
  position: relative;

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .default-cover {
    width: 100%;
    height: 100%;
    background: #333;
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 30px;
  }

  &.rotating {
    animation: rotate 10s linear infinite;
  }
}

@keyframes rotate {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.info {
  text-align: center;

  .track-name {
    font-weight: bold;
    font-size: 16px;
    margin-bottom: 4px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 240px;
  }

  .artist {
    font-size: 12px;
    opacity: 0.7;
  }
}

.controls {
  display: flex;
  align-items: center;
  gap: 15px;
  margin: 5px 0;
}

.ctrl-btn {
  background: transparent;
  border: none;
  font-size: 20px;
  cursor: pointer;
  color: rgb(var(--color-text-primary));
  transition: transform 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;

  &:hover {
    transform: scale(1.2);
    color: #409eff;
  }

  &.play-btn {
    font-size: 32px;
  }

  &.list-btn {
    font-size: 18px;

    &.active {
      color: #409eff;
    }
  }
}

.progress-bar {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 10px;
  opacity: 0.8;

  input[type="range"] {
    flex: 1;
    height: 4px;
    border-radius: 2px;
    background: rgba(var(--color-text-primary), 0.2);
    appearance: none;
    cursor: pointer;

    &::-webkit-slider-thumb {
      appearance: none;
      width: 10px;
      height: 10px;
      border-radius: 50%;
      background: #409eff;
    }
  }
}
</style>
