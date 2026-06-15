<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import PomodoroTimer from './components/PomodoroTimer.vue'
import PixelTreeCanvas from './components/PixelTreeCanvas.vue'
import AmbientPlayer from './components/AmbientPlayer.vue'
import SettingsModal from './components/SettingsModal.vue'

const theme = ref(localStorage.getItem('theme') || 'dark')
const showSettings = ref(false)

const defaultSettings = { workMinutes: 25, breakMinutes: 5, sessionsToFullTree: 8 }
const settings = ref((() => {
  try { return JSON.parse(localStorage.getItem('settings')) || defaultSettings }
  catch { return defaultSettings }
})())

const completedSessions = ref(Math.max(0, parseInt(localStorage.getItem('completedSessions')) || 0))
const rawSeed = parseInt(localStorage.getItem('treeSeed'))
const treeSeed = ref(Number.isFinite(rawSeed) ? rawSeed : Math.floor(Math.random() * 99999))

const treeProgress = computed(() => Math.min(completedSessions.value / settings.value.sessionsToFullTree, 1))

function onSessionComplete() {
  completedSessions.value++
  localStorage.setItem('completedSessions', completedSessions.value)
}

function onSettingsSave(newSettings) {
  settings.value = newSettings
  localStorage.setItem('settings', JSON.stringify(newSettings))
  showSettings.value = false
}

function resetTree() {
  completedSessions.value = 0
  localStorage.setItem('completedSessions', '0')
  treeSeed.value = Math.floor(Math.random() * 99999)
  localStorage.setItem('treeSeed', treeSeed.value)
}

function toggleTheme() {
  theme.value = theme.value === 'dark' ? 'light' : 'dark'
  localStorage.setItem('theme', theme.value)
}

watch(theme, (val) => {
  document.documentElement.setAttribute('data-theme', val)
}, { immediate: true })
</script>

<template>
  <div class="app" :class="theme">
    <header class="app-header">
      <div class="logo">🌳 FocusTree</div>
      <div class="header-actions">
        <button class="icon-btn" @click="toggleTheme" :title="theme === 'dark' ? 'Light mode' : 'Dark mode'">
          {{ theme === 'dark' ? '☀️' : '🌙' }}
        </button>
        <button class="icon-btn" @click="showSettings = true" title="Settings">⚙️</button>
      </div>
    </header>

    <main class="app-main">
      <div class="left-panel">
        <PixelTreeCanvas :progress="treeProgress" :theme="theme" :tree-seed="treeSeed" />
        <div class="tree-info">
          <span class="sessions-count">{{ completedSessions }} / {{ settings.sessionsToFullTree }} sessions</span>
          <button class="reset-btn" @click="resetTree">Reset tree</button>
        </div>
      </div>

      <div class="right-panel">
        <PomodoroTimer
          :work-minutes="settings.workMinutes"
          :break-minutes="settings.breakMinutes"
          @session-complete="onSessionComplete"
        />
        <AmbientPlayer />
      </div>
    </main>

    <SettingsModal
      v-if="showSettings"
      :settings="settings"
      @save="onSettingsSave"
      @close="showSettings = false"
    />
  </div>
</template>

<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
body { font-family: 'Inter', system-ui, sans-serif; transition: background 0.3s, color 0.3s; }

.dark {
  --bg: #0f1117;
  --bg2: #1a1d27;
  --bg3: #222638;
  --text: #e8eaf0;
  --text2: #8b8fa8;
  --accent: #7c6af7;
  --accent2: #f97316;
  --green: #22c55e;
  --border: rgba(255,255,255,0.08);
  --shadow: 0 8px 32px rgba(0,0,0,0.4);
}

.light {
  --bg: #f0f2f8;
  --bg2: #ffffff;
  --bg3: #e8eaf2;
  --text: #1a1d27;
  --text2: #6b6f85;
  --accent: #7c6af7;
  --accent2: #f97316;
  --green: #16a34a;
  --border: rgba(0,0,0,0.08);
  --shadow: 0 8px 32px rgba(0,0,0,0.1);
}
</style>

<style scoped>
.app {
  min-height: 100vh;
  background: var(--bg);
  color: var(--text);
}

.app-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 32px;
  border-bottom: 1px solid var(--border);
}

.logo {
  font-size: 20px;
  font-weight: 700;
  letter-spacing: -0.5px;
}

.header-actions { display: flex; gap: 8px; }

.icon-btn {
  background: var(--bg3);
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 8px 12px;
  cursor: pointer;
  font-size: 16px;
  transition: background 0.2s;
}
.icon-btn:hover { background: var(--bg2); }

.app-main {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 32px;
  padding: 32px;
  max-width: 1100px;
  margin: 0 auto;
}

.left-panel, .right-panel {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.tree-info {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  background: var(--bg2);
  border-radius: 12px;
  border: 1px solid var(--border);
}

.sessions-count {
  font-size: 14px;
  color: var(--text2);
  font-weight: 500;
}

.reset-btn {
  background: none;
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 6px 12px;
  font-size: 13px;
  color: var(--text2);
  cursor: pointer;
  transition: all 0.2s;
}
.reset-btn:hover { border-color: var(--accent2); color: var(--accent2); }

@media (max-width: 768px) {
  .app-main { grid-template-columns: 1fr; padding: 16px; }
  .app-header { padding: 12px 16px; }
}
</style>
