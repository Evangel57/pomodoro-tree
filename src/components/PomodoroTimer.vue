<script setup>
import { ref, computed, watch, onUnmounted } from 'vue'

const props = defineProps({
  workMinutes: { type: Number, default: 25 },
  breakMinutes: { type: Number, default: 5 }
})

const emit = defineEmits(['session-complete'])

const mode = ref('work')
const secondsLeft = ref(props.workMinutes * 60)
const running = ref(false)
let interval = null

const totalSeconds = computed(() => mode.value === 'work' ? props.workMinutes * 60 : props.breakMinutes * 60)
const progress = computed(() => 1 - secondsLeft.value / totalSeconds.value)

const minutes = computed(() => String(Math.floor(secondsLeft.value / 60)).padStart(2, '0'))
const seconds = computed(() => String(secondsLeft.value % 60).padStart(2, '0'))

const circumference = 2 * Math.PI * 54

watch(() => [props.workMinutes, props.breakMinutes], () => {
  reset()
})

function tick() {
  if (secondsLeft.value <= 0) {
    clearInterval(interval)
    running.value = false
    if (mode.value === 'work') {
      emit('session-complete')
      mode.value = 'break'
      secondsLeft.value = props.breakMinutes * 60
    } else {
      mode.value = 'work'
      secondsLeft.value = props.workMinutes * 60
    }
    playNotification()
    return
  }
  secondsLeft.value--
}

function toggle() {
  if (running.value) {
    clearInterval(interval)
    running.value = false
  } else {
    interval = setInterval(tick, 1000)
    running.value = true
  }
}

function reset() {
  clearInterval(interval)
  running.value = false
  mode.value = 'work'
  secondsLeft.value = props.workMinutes * 60
}

function skipToBreak() {
  clearInterval(interval)
  running.value = false
  if (mode.value === 'work') {
    emit('session-complete')
    mode.value = 'break'
    secondsLeft.value = props.breakMinutes * 60
  } else {
    mode.value = 'work'
    secondsLeft.value = props.workMinutes * 60
  }
}

function playNotification() {
  try {
    const ctx = new AudioContext()
    const master = ctx.createGain()
    master.gain.value = 0.22
    master.connect(ctx.destination)

    const isWork = mode.value !== 'work'
    const notes = isWork
      ? [523, 659, 784]      // work done: C E G ascending
      : [784, 659, 523]      // break done: G E C descending

    notes.forEach((freq, i) => {
      const osc  = ctx.createOscillator()
      const gain = ctx.createGain()
      osc.type = 'sine'
      osc.frequency.value = freq
      osc.connect(gain)
      gain.connect(master)
      const t = ctx.currentTime + i * 0.18
      gain.gain.setValueAtTime(0, t)
      gain.gain.linearRampToValueAtTime(1, t + 0.02)
      gain.gain.exponentialRampToValueAtTime(0.001, t + 0.5)
      osc.start(t)
      osc.stop(t + 0.5)
    })
  } catch {}
}

onUnmounted(() => clearInterval(interval))
</script>

<template>
  <div class="timer-card">
    <div class="mode-tabs">
      <button :class="['tab', { active: mode === 'work' }]" @click="reset">Focus</button>
      <button :class="['tab', { active: mode === 'break' }]" @click="skipToBreak">Break</button>
    </div>

    <div class="timer-ring">
      <svg viewBox="0 0 120 120" class="ring-svg">
        <circle cx="60" cy="60" r="54" class="ring-bg" />
        <circle
          cx="60" cy="60" r="54"
          class="ring-progress"
          :class="mode"
          :style="{
            strokeDasharray: circumference,
            strokeDashoffset: circumference * (1 - progress)
          }"
        />
      </svg>
      <div class="timer-display">
        <div class="time">{{ minutes }}:{{ seconds }}</div>
        <div class="mode-label">{{ mode === 'work' ? 'Focus time' : 'Break time' }}</div>
      </div>
    </div>

    <div class="timer-actions">
      <button class="btn-secondary" @click="reset">↺</button>
      <button class="btn-primary" @click="toggle">
        {{ running ? '⏸ Pause' : '▶ Start' }}
      </button>
      <button class="btn-secondary" @click="skipToBreak">⏭</button>
    </div>
  </div>
</template>

<style scoped>
.timer-card {
  background: var(--bg2);
  border: 1px solid var(--border);
  border-radius: 20px;
  padding: 28px 24px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 24px;
  box-shadow: var(--shadow);
}

.mode-tabs {
  display: flex;
  gap: 8px;
  background: var(--bg3);
  border-radius: 10px;
  padding: 4px;
}

.tab {
  padding: 8px 20px;
  border: none;
  border-radius: 8px;
  background: none;
  color: var(--text2);
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}
.tab.active {
  background: var(--accent);
  color: white;
}

.timer-ring {
  position: relative;
  width: 200px;
  height: 200px;
}

.ring-svg {
  width: 100%;
  height: 100%;
  transform: rotate(-90deg);
}

.ring-bg {
  fill: none;
  stroke: var(--bg3);
  stroke-width: 8;
}

.ring-progress {
  fill: none;
  stroke-width: 8;
  stroke-linecap: round;
  transition: stroke-dashoffset 1s linear;
}
.ring-progress.work { stroke: var(--accent); }
.ring-progress.break { stroke: var(--green); }

.timer-display {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 4px;
}

.time {
  font-size: 44px;
  font-weight: 700;
  letter-spacing: -2px;
  color: var(--text);
  font-variant-numeric: tabular-nums;
}

.mode-label {
  font-size: 13px;
  color: var(--text2);
  text-transform: uppercase;
  letter-spacing: 1px;
}

.timer-actions {
  display: flex;
  align-items: center;
  gap: 12px;
}

.btn-primary {
  background: var(--accent);
  color: white;
  border: none;
  border-radius: 12px;
  padding: 14px 32px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: opacity 0.2s, transform 0.1s;
  min-width: 130px;
}
.btn-primary:hover { opacity: 0.9; }
.btn-primary:active { transform: scale(0.98); }

.btn-secondary {
  background: var(--bg3);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 12px 16px;
  font-size: 18px;
  cursor: pointer;
  transition: background 0.2s;
  color: var(--text);
}
.btn-secondary:hover { background: var(--bg); }
</style>
