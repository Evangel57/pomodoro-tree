<script setup>
import { ref, onUnmounted } from 'vue'

const scenes = [
  { name: 'Brown Noise', icon: '🎧', type: 'brown' },
  { name: 'Rain', icon: '🌧️', type: 'fire' },
  { name: 'White Noise', icon: '🌊', type: 'white' },
  { name: 'Deep Focus', icon: '🧠', type: 'deep' },
  { name: 'Thunderstorm', icon: '⛈️', type: 'thunder' },
  { name: 'Forest', icon: '🌲', type: 'forest' },
  { name: 'Fireplace', icon: '🔥', type: 'fireplace' },
  { name: 'Ocean Waves', icon: '🌊', type: 'ocean' },
  { name: 'Embers', icon: '✨', type: 'embers' },
  { name: 'Pixel', icon: '👾', type: 'pixel' },
  { name: 'Metal Drone', icon: '🔔', type: 'metal' },
  { name: 'Gravel', icon: '🪨', type: 'gravel' },
  { name: 'Underwater', icon: '🫧', type: 'underwater' },
]

const current = ref(0)
const playing = ref(false)
const volume = ref(0.5)

let ctx = null
let nodes = []
let timers = []

function createCtx() {
  if (!ctx) ctx = new (window.AudioContext || window.webkitAudioContext)()
  return ctx
}

function stopAll() {
  timers.forEach(id => { clearInterval(id); clearTimeout(id) })
  timers = []
  nodes.forEach(n => { try { n.stop?.(); n.disconnect?.() } catch {} })
  nodes = []
}

function makeNoise(type) {
  const c = createCtx()
  const bufSize = c.sampleRate * 4
  const buf = c.createBuffer(1, bufSize, c.sampleRate)
  const data = buf.getChannelData(0)

  if (type === 'brown') {
    let last = 0
    for (let i = 0; i < bufSize; i++) {
      const w = Math.random() * 2 - 1
      last = (last + 0.02 * w) / 1.02
      data[i] = last * 8
    }
  } else if (type === 'white' || type === 'rain') {
    for (let i = 0; i < bufSize; i++) data[i] = Math.random() * 2 - 1
  } else if (type === 'deep') {
    let b0 = 0, b1 = 0, b2 = 0, b3 = 0, b4 = 0, b5 = 0
    for (let i = 0; i < bufSize; i++) {
      const w = Math.random() * 2 - 1
      b0 = 0.99886 * b0 + w * 0.0555179
      b1 = 0.99332 * b1 + w * 0.0750759
      b2 = 0.96900 * b2 + w * 0.1538520
      b3 = 0.86650 * b3 + w * 0.3104856
      b4 = 0.55000 * b4 + w * 0.5329522
      b5 = -0.7616 * b5 - w * 0.0168980
      data[i] = (b0 + b1 + b2 + b3 + b4 + b5) * 0.11
    }
  }

  return buf
}

function playThunder() {
  const c = createCtx()
  const gain = c.createGain()
  gain.gain.setValueAtTime(0, c.currentTime)
  gain.gain.linearRampToValueAtTime(volume.value * 0.4, c.currentTime + 0.5)
  gain.connect(c.destination)

  const buf = makeNoise('white')

  const rainSrc = c.createBufferSource()
  rainSrc.buffer = buf
  rainSrc.loop = true
  const rainFilter = c.createBiquadFilter()
  rainFilter.type = 'bandpass'
  rainFilter.frequency.value = 600
  rainFilter.Q.value = 0.4
  rainSrc.connect(rainFilter)
  rainFilter.connect(gain)
  rainSrc.start()

  // ветер: white → highpass 400Hz, LFO на громкость (порывы)
  const windBuf = makeNoise('white')
  const windSrc = c.createBufferSource()
  windSrc.buffer = windBuf
  windSrc.loop = true
  const windF = c.createBiquadFilter()
  windF.type = 'highpass'
  windF.frequency.value = 400
  const windGain = c.createGain()
  windGain.gain.value = volume.value * 0.25
  const windLfo = c.createOscillator()
  windLfo.frequency.value = 0.12
  const windLfoG = c.createGain()
  windLfoG.gain.value = volume.value * 0.15
  windLfo.connect(windLfoG); windLfoG.connect(windGain.gain)
  windLfo.start()
  windSrc.connect(windF); windF.connect(windGain); windGain.connect(gain)
  windSrc.start()
  nodes.push(windSrc, windGain, windLfo, windLfoG)

  function scheduleThunder() {
    const delay = 4 + Math.random() * 10
    const tid = setTimeout(() => {
      if (!playing.value) return
      const tGain = c.createGain()
      tGain.connect(c.destination)
      const tBuf = makeNoise('brown')
      const tSrc = c.createBufferSource()
      tSrc.buffer = tBuf
      const tFilter = c.createBiquadFilter()
      tFilter.type = 'lowpass'
      tFilter.frequency.value = 120
      tSrc.connect(tFilter)
      tFilter.connect(tGain)
      tGain.gain.setValueAtTime(0, c.currentTime)
      tGain.gain.linearRampToValueAtTime(volume.value * 0.8, c.currentTime + 0.1)
      tGain.gain.exponentialRampToValueAtTime(0.001, c.currentTime + 2.5)
      tSrc.start()
      tSrc.stop(c.currentTime + 2.5)
      nodes.push(tSrc, tGain)
      scheduleThunder()
    }, delay * 1000)
    timers.push(tid)
  }
  scheduleThunder()
  nodes.push(rainSrc, gain)
  return gain
}

function playForest() {
  const c = createCtx()
  const masterGain = c.createGain()
  masterGain.gain.setValueAtTime(0, c.currentTime)
  masterGain.gain.linearRampToValueAtTime(volume.value * 0.35, c.currentTime + 0.5)
  masterGain.connect(c.destination)

  const buf = makeNoise('white')

  const windSrc = c.createBufferSource()
  windSrc.buffer = buf
  windSrc.loop = true
  const windFilter = c.createBiquadFilter()
  windFilter.type = 'bandpass'
  windFilter.frequency.value = 300
  windFilter.Q.value = 0.3
  windSrc.connect(windFilter)
  windFilter.connect(masterGain)
  windSrc.start()

  function chirp(freq, interval) {
    const id = setInterval(() => {
      if (!playing.value) { clearInterval(id); timers.splice(timers.indexOf(id), 1); return }
      const g = c.createGain()
      g.connect(c.destination)
      g.gain.setValueAtTime(0, c.currentTime)
      const osc1 = c.createOscillator()
      osc1.type = 'sine'
      osc1.frequency.setValueAtTime(freq, c.currentTime)
      osc1.frequency.linearRampToValueAtTime(freq * 1.3, c.currentTime + 0.08)
      osc1.connect(g)
      g.gain.linearRampToValueAtTime(volume.value * 0.15, c.currentTime + 0.02)
      g.gain.exponentialRampToValueAtTime(0.001, c.currentTime + 0.25)
      osc1.start()
      osc1.stop(c.currentTime + 0.3)
    }, interval + Math.random() * 1000)
    return id
  }

  timers.push(...[chirp(2200, 3000), chirp(3100, 4500), chirp(1800, 6000)])

  nodes.push(windSrc, masterGain)
  return masterGain
}

function playFire() {
  const c = createCtx()
  const master = c.createGain()
  master.gain.setValueAtTime(0, c.currentTime)
  master.gain.linearRampToValueAtTime(volume.value * 0.5, c.currentTime + 0.8)
  master.connect(c.destination)

  // === base flame hiss: white noise → bandpass 300-700Hz (airy whoosh) ===
  const flameBuf = makeNoise('white')
  const flameSrc = c.createBufferSource()
  flameSrc.buffer = flameBuf
  flameSrc.loop = true
  const flameFilter = c.createBiquadFilter()
  flameFilter.type = 'bandpass'
  flameFilter.frequency.value = 450
  flameFilter.Q.value = 0.6
  const flameGain = c.createGain()
  flameGain.gain.value = 0.6
  flameSrc.connect(flameFilter)
  flameFilter.connect(flameGain)
  flameGain.connect(master)
  flameSrc.start()

  // === deep rumble: brown noise → lowpass 200Hz ===
  const rumbleBuf = makeNoise('brown')
  const rumbleSrc = c.createBufferSource()
  rumbleSrc.buffer = rumbleBuf
  rumbleSrc.loop = true
  const rumbleFilter = c.createBiquadFilter()
  rumbleFilter.type = 'lowpass'
  rumbleFilter.frequency.value = 180
  const rumbleGain = c.createGain()
  rumbleGain.gain.value = 1.2
  rumbleSrc.connect(rumbleFilter)
  rumbleFilter.connect(rumbleGain)
  rumbleGain.connect(master)
  rumbleSrc.start()

  // === crackles: route through master so stopAll works ===
  function crackle() {
    const delay = 60 + Math.random() * 300
    const tid = setTimeout(() => {
      if (!playing.value) return
      const cGain = c.createGain()
      cGain.connect(master)           // ← через master, не destination
      const cBuf = makeNoise('white')
      const cSrc = c.createBufferSource()
      cSrc.buffer = cBuf
      const cf = c.createBiquadFilter()
      cf.type = 'bandpass'
      cf.frequency.value = 600 + Math.random() * 3000
      cf.Q.value = 3 + Math.random() * 6
      cSrc.connect(cf)
      cf.connect(cGain)
      const dur = 0.015 + Math.random() * 0.055
      cGain.gain.setValueAtTime(volume.value * (0.15 + Math.random() * 0.35), c.currentTime)
      cGain.gain.exponentialRampToValueAtTime(0.001, c.currentTime + dur)
      cSrc.start()
      cSrc.stop(c.currentTime + dur + 0.01)
      nodes.push(cSrc, cGain)
      crackle()
    }, delay)
    timers.push(tid)
  }
  crackle()
  crackle()
  crackle() // три потока для плотного треска

  // === fire pops: частые, менее глухие ===
  function pop() {
    const delay = 200 + Math.random() * 600
    const tid = setTimeout(() => {
      if (!playing.value) return
      const pGain = c.createGain()
      pGain.connect(gain)
      const pBuf = makeNoise('white')
      const pSrc = c.createBufferSource()
      pSrc.buffer = pBuf
      const pf = c.createBiquadFilter()
      pf.type = 'bandpass'
      pf.frequency.value = 800 + Math.random() * 1200
      pf.Q.value = 1.5
      pSrc.connect(pf)
      pf.connect(pGain)
      const peakVol = volume.value * (1.2 + Math.random() * 1.2)
      const dur = 0.03 + Math.random() * 0.06
      pGain.gain.setValueAtTime(peakVol, c.currentTime)
      pGain.gain.exponentialRampToValueAtTime(0.001, c.currentTime + dur)
      pSrc.start()
      pSrc.stop(c.currentTime + dur + 0.01)
      nodes.push(pSrc, pGain)
      pop()
    }, delay)
    timers.push(tid)
  }
  pop()
  pop()
  pop()

  nodes.push(flameSrc, flameGain, rumbleSrc, rumbleGain, master)
  return master
}

function playEmbers() {
  const c = createCtx()
  const master = c.createGain()
  master.gain.setValueAtTime(0, c.currentTime)
  master.gain.linearRampToValueAtTime(volume.value * 0.4, c.currentTime + 1.0)
  master.connect(c.destination)

  // тихий фоновый жар
  const baseBuf = makeNoise('brown')
  const baseSrc = c.createBufferSource()
  baseSrc.buffer = baseBuf
  baseSrc.loop = true
  const baseF = c.createBiquadFilter()
  baseF.type = 'lowpass'
  baseF.frequency.value = 200
  const baseGain = c.createGain()
  baseGain.gain.value = 0.5
  baseSrc.connect(baseF)
  baseF.connect(baseGain)
  baseGain.connect(master)
  baseSrc.start()

  // тёплые угасающие искры
  function ember() {
    const delay = 600 + Math.random() * 2500
    const tid = setTimeout(() => {
      if (!playing.value) return
      const eGain = c.createGain()
      eGain.connect(c.destination)
      const eBuf = makeNoise('brown')
      const eSrc = c.createBufferSource()
      eSrc.buffer = eBuf
      const ef = c.createBiquadFilter()
      ef.type = 'lowpass'
      ef.frequency.value = 400 + Math.random() * 500
      eSrc.connect(ef)
      ef.connect(eGain)
      const peakVol = volume.value * (2.0 + Math.random() * 2.0)
      const dur = 0.08 + Math.random() * 0.15
      eGain.gain.setValueAtTime(peakVol, c.currentTime)
      eGain.gain.exponentialRampToValueAtTime(0.001, c.currentTime + dur)
      eSrc.start()
      eSrc.stop(c.currentTime + dur + 0.01)
      nodes.push(eSrc, eGain)
      ember()
    }, delay)
    timers.push(tid)
  }
  ember()
  ember()

  nodes.push(baseSrc, baseGain, master)
  return master
}

function playOcean() {
  const c = createCtx()
  const gain = c.createGain()
  gain.gain.setValueAtTime(0, c.currentTime)
  gain.gain.linearRampToValueAtTime(volume.value * 0.4, c.currentTime + 0.5)
  gain.connect(c.destination)

  const buf = makeNoise('white')
  const src = c.createBufferSource()
  src.buffer = buf
  src.loop = true

  const filter = c.createBiquadFilter()
  filter.type = 'lowpass'
  filter.frequency.value = 500

  src.connect(filter)
  filter.connect(gain)
  src.start()

  let wavePhase = 0
  const waveInterval = setInterval(() => {
    if (!playing.value) { clearInterval(waveInterval); timers.splice(timers.indexOf(waveInterval), 1); return }
    wavePhase++
    const period = 4
    const t = c.currentTime
    const waveGain = c.createGain()
    waveGain.connect(c.destination)
    const waveBuf = makeNoise('white')
    const waveSrc = c.createBufferSource()
    waveSrc.buffer = waveBuf
    const wf = c.createBiquadFilter()
    wf.type = 'bandpass'
    wf.frequency.value = 400
    wf.Q.value = 0.5
    waveSrc.connect(wf)
    wf.connect(waveGain)
    waveGain.gain.setValueAtTime(0, t)
    waveGain.gain.linearRampToValueAtTime(volume.value * 0.3, t + period * 0.4)
    waveGain.gain.linearRampToValueAtTime(0, t + period)
    waveSrc.start(t)
    waveSrc.stop(t + period)
    nodes.push(waveSrc, waveGain)
  }, 3500)

  timers.push(waveInterval)
  nodes.push(src, gain)
  return gain
}

function playFireplace() {
  const c = createCtx()
  const master = c.createGain()
  master.gain.setValueAtTime(0, c.currentTime)
  master.gain.linearRampToValueAtTime(volume.value * 0.45, c.currentTime + 1.0)
  master.connect(c.destination)

  // === белый шум → lowpass 900Hz — основное шипение пламени ===
  const hissBuf = makeNoise('white')
  const hissSrc = c.createBufferSource()
  hissSrc.buffer = hissBuf
  hissSrc.loop = true
  const hissF = c.createBiquadFilter()
  hissF.type = 'lowpass'
  hissF.frequency.value = 500
  // LFO тонко качает cutoff: глубина 150Hz — едва заметное дыхание
  const lfo = c.createOscillator()
  lfo.frequency.value = 0.2
  const lfoGain = c.createGain()
  lfoGain.gain.value = 150
  lfo.connect(lfoGain)
  lfoGain.connect(hissF.frequency)
  lfo.start()
  const hissGain = c.createGain()
  hissGain.gain.value = 0.7
  hissSrc.connect(hissF); hissF.connect(hissGain); hissGain.connect(master)
  hissSrc.start()

  // === brown noise → lowpass 200Hz + peaking — глубокий жар ===
  const heatBuf = makeNoise('brown')
  const heatSrc = c.createBufferSource()
  heatSrc.buffer = heatBuf
  heatSrc.loop = true
  const heatF = c.createBiquadFilter()
  heatF.type = 'lowpass'
  heatF.frequency.value = 100
  const heatF2 = c.createBiquadFilter()
  heatF2.type = 'peaking'
  heatF2.frequency.value = 80
  heatF2.gain.value = 8
  const heatGain = c.createGain()
  heatGain.gain.value = 1.5
  heatSrc.connect(heatF); heatF.connect(heatF2); heatF2.connect(heatGain); heatGain.connect(master)
  heatSrc.start()

  // === треск: пачками — тишина, потом несколько щелчков подряд ===
  function snap() {
    const now = c.currentTime
    const cg = c.createGain()
    cg.connect(master)
    const cs = c.createBufferSource()
    cs.buffer = makeNoise('white')
    cs.connect(cg)
    const dur = 0.006 + Math.random() * 0.018
    cg.gain.setValueAtTime(volume.value * (0.25 + Math.random() * 0.65), now)
    cg.gain.exponentialRampToValueAtTime(0.0001, now + dur)
    cs.start(now); cs.stop(now + dur + 0.002)
    nodes.push(cs, cg)
  }

  function burst() {
    // пауза 300–2000мс перед следующей пачкой
    const restDelay = 80 + Math.random() * 600
    const tid = setTimeout(() => {
      if (!playing.value) return
      // пачка: 2–8 щелчков с интервалом 15–80мс
      const count = 2 + Math.floor(Math.random() * 7)
      for (let i = 0; i < count; i++) {
        const snapTid = setTimeout(() => {
          if (!playing.value) return
          snap()
        }, i * (15 + Math.random() * 60))
        timers.push(snapTid)
      }
      burst()
    }, restDelay)
    timers.push(tid)
  }
  burst()

  // === хлопок дров: низкий удар + высокий угасающий хвост искр ===
  function pop() {
    const delay = 600 + Math.random() * 2000
    const tid = setTimeout(() => {
      if (!playing.value) return
      const now = c.currentTime

      // r()*r()*r() — сильно скошено к тихим: большинство тихие, редко громкие
      const intensity = Math.random() * Math.random()

      // низкий удар
      const thudG = c.createGain()
      thudG.connect(master)
      const thudS = c.createBufferSource()
      thudS.buffer = makeNoise('brown')
      const thudF = c.createBiquadFilter()
      thudF.type = 'lowpass'
      thudF.frequency.value = 70 + Math.random() * 60
      thudS.connect(thudF); thudF.connect(thudG)
      const thudDur = 0.15 + intensity * 0.5 + Math.random() * 0.15
      thudG.gain.setValueAtTime(volume.value * (0.05 + intensity * 4.0), now)
      thudG.gain.exponentialRampToValueAtTime(0.001, now + thudDur)
      thudS.start(); thudS.stop(now + thudDur + 0.01)
      nodes.push(thudS, thudG)

      // высокий хвост искр
      const tailG = c.createGain()
      tailG.connect(master)
      const tailS = c.createBufferSource()
      tailS.buffer = makeNoise('white')
      const tailF = c.createBiquadFilter()
      tailF.type = 'bandpass'
      tailF.frequency.value = 3000 + Math.random() * 2000
      tailF.Q.value = 6
      tailS.connect(tailF); tailF.connect(tailG)
      const td = 0.03
      const tailDur = 0.2 + intensity * 0.4 + Math.random() * 0.15
      tailG.gain.setValueAtTime(0, now)
      tailG.gain.linearRampToValueAtTime(volume.value * (0.02 + intensity * 0.9), now + td)
      tailG.gain.exponentialRampToValueAtTime(0.001, now + td + tailDur)
      tailS.start(); tailS.stop(now + td + tailDur + 0.01)
      nodes.push(tailS, tailG)

      pop()
    }, delay)
    timers.push(tid)
  }
  pop(); pop()

  nodes.push(hissSrc, hissGain, heatSrc, heatGain, lfo, lfoGain, master)
  return master
}

function playPixel() {
  const c = createCtx()
  const master = c.createGain()
  master.gain.setValueAtTime(0, c.currentTime)
  master.gain.linearRampToValueAtTime(volume.value * 0.25, c.currentTime + 0.3)
  master.connect(c.destination)

  // тихий фоновый хам: square wave низко
  const drone = c.createOscillator()
  drone.type = 'square'
  drone.frequency.value = 55
  const droneG = c.createGain()
  droneG.gain.value = 0.04
  drone.connect(droneG); droneG.connect(master)
  drone.start()
  nodes.push(drone, droneG)

  // пентатоника для случайных нот
  const scale = [261, 294, 329, 392, 440, 523, 587, 659, 784, 880]

  function beep() {
    const delay = 300 + Math.random() * 1200
    const tid = setTimeout(() => {
      if (!playing.value) return
      const freq = scale[Math.floor(Math.random() * scale.length)]
      const osc = c.createOscillator()
      osc.type = Math.random() > 0.5 ? 'square' : 'triangle'
      osc.frequency.value = freq * (Math.random() > 0.8 ? 2 : 1)
      const og = c.createGain()
      og.connect(master)
      osc.connect(og)
      const dur = 0.04 + Math.random() * 0.12
      og.gain.setValueAtTime(volume.value * (0.3 + Math.random() * 0.4), c.currentTime)
      og.gain.exponentialRampToValueAtTime(0.001, c.currentTime + dur)
      osc.start(); osc.stop(c.currentTime + dur + 0.01)
      nodes.push(osc, og)
      beep()
    }, delay)
    timers.push(tid)
  }
  beep(); beep()
  nodes.push(master)
  return master
}

function playMetal() {
  const c = createCtx()
  const master = c.createGain()
  master.gain.setValueAtTime(0, c.currentTime)
  master.gain.linearRampToValueAtTime(volume.value * 0.3, c.currentTime + 2.0)
  master.connect(c.destination)

  // 6 гармонических синусоид — металлический дрон
  const fundamentals = [55, 82.4, 110, 146.8, 164.8, 220]
  fundamentals.forEach((freq, i) => {
    const osc = c.createOscillator()
    osc.type = 'sine'
    osc.frequency.value = freq
    // AM модуляция — тремоло на каждой частоте своё
    const am = c.createOscillator()
    am.frequency.value = 0.05 + i * 0.03
    const amG = c.createGain()
    amG.gain.value = 0.3
    am.connect(amG)
    const og = c.createGain()
    og.gain.value = 0.12 - i * 0.01
    amG.connect(og.gain)
    osc.connect(og); og.connect(master)
    osc.start(); am.start()
    nodes.push(osc, og, am, amG)
  })

  // редкие удары как колокол
  function bell() {
    const delay = 3000 + Math.random() * 8000
    const tid = setTimeout(() => {
      if (!playing.value) return
      const freq = [220, 293.7, 329.6, 440][Math.floor(Math.random() * 4)]
      const osc = c.createOscillator()
      osc.type = 'sine'
      osc.frequency.value = freq
      const og = c.createGain()
      og.connect(master)
      osc.connect(og)
      og.gain.setValueAtTime(volume.value * 1.2, c.currentTime)
      og.gain.exponentialRampToValueAtTime(0.001, c.currentTime + 3.5)
      osc.start(); osc.stop(c.currentTime + 4)
      nodes.push(osc, og)
      bell()
    }, delay)
    timers.push(tid)
  }
  bell()
  nodes.push(master)
  return master
}

function playGravel() {
  const c = createCtx()
  const master = c.createGain()
  master.gain.setValueAtTime(0, c.currentTime)
  master.gain.linearRampToValueAtTime(volume.value * 0.4, c.currentTime + 0.5)
  master.connect(c.destination)

  // фоновый шелест бумаги: white → bandpass 1200Hz
  const paperBuf = makeNoise('white')
  const paperSrc = c.createBufferSource()
  paperSrc.buffer = paperBuf
  paperSrc.loop = true
  const paperF = c.createBiquadFilter()
  paperF.type = 'bandpass'
  paperF.frequency.value = 1200
  paperF.Q.value = 0.8
  const paperG = c.createGain()
  paperG.gain.value = 0.15
  paperSrc.connect(paperF); paperF.connect(paperG); paperG.connect(master)
  paperSrc.start()

  // шаги по гравию: нерегулярные короткие всплески
  function step() {
    const delay = 400 + Math.random() * 1800
    const tid = setTimeout(() => {
      if (!playing.value) return
      const sg = c.createGain()
      sg.connect(master)
      const ss = c.createBufferSource()
      ss.buffer = makeNoise('white')
      const sf = c.createBiquadFilter()
      sf.type = 'bandpass'
      sf.frequency.value = 600 + Math.random() * 1400
      sf.Q.value = 1.2
      ss.connect(sf); sf.connect(sg)
      const dur = 0.03 + Math.random() * 0.08
      sg.gain.setValueAtTime(volume.value * (0.6 + Math.random() * 0.8), c.currentTime)
      sg.gain.exponentialRampToValueAtTime(0.001, c.currentTime + dur)
      ss.start(); ss.stop(c.currentTime + dur + 0.01)
      nodes.push(ss, sg)
      step()
    }, delay)
    timers.push(tid)
  }
  step()

  // редкое шуршание бумаги
  function rustle() {
    const delay = 2000 + Math.random() * 5000
    const tid = setTimeout(() => {
      if (!playing.value) return
      const rg = c.createGain()
      rg.connect(master)
      const rs = c.createBufferSource()
      rs.buffer = makeNoise('white')
      const rf = c.createBiquadFilter()
      rf.type = 'bandpass'
      rf.frequency.value = 2000 + Math.random() * 1000
      rf.Q.value = 0.5
      rs.connect(rf); rf.connect(rg)
      const dur = 0.15 + Math.random() * 0.3
      rg.gain.setValueAtTime(0, c.currentTime)
      rg.gain.linearRampToValueAtTime(volume.value * 0.5, c.currentTime + dur * 0.3)
      rg.gain.exponentialRampToValueAtTime(0.001, c.currentTime + dur)
      rs.start(); rs.stop(c.currentTime + dur + 0.01)
      nodes.push(rs, rg)
      rustle()
    }, delay)
    timers.push(tid)
  }
  rustle()

  nodes.push(paperSrc, paperG, master)
  return master
}

function playUnderwater() {
  const c = createCtx()
  const master = c.createGain()
  master.gain.setValueAtTime(0, c.currentTime)
  master.gain.linearRampToValueAtTime(volume.value * 0.5, c.currentTime + 2.0)
  master.connect(c.destination)

  // глубокий гул воды: brown → lowpass 200Hz + LFO давление
  const baseBuf = makeNoise('brown')
  const baseSrc = c.createBufferSource()
  baseSrc.buffer = baseBuf
  baseSrc.loop = true
  const baseF = c.createBiquadFilter()
  baseF.type = 'lowpass'
  baseF.frequency.value = 200
  const lfo = c.createOscillator()
  lfo.frequency.value = 0.08
  const lfoG = c.createGain()
  lfoG.gain.value = 80
  lfo.connect(lfoG); lfoG.connect(baseF.frequency)
  lfo.start()
  baseSrc.connect(baseF); baseF.connect(master)
  baseSrc.start()

  // пузыри: короткие синус-блипы с нарастающей частотой
  function bubble() {
    const delay = 500 + Math.random() * 3000
    const tid = setTimeout(() => {
      if (!playing.value) return
      const osc = c.createOscillator()
      osc.type = 'sine'
      const startF = 200 + Math.random() * 300
      osc.frequency.setValueAtTime(startF, c.currentTime)
      osc.frequency.linearRampToValueAtTime(startF * 2.5, c.currentTime + 0.15)
      const og = c.createGain()
      og.connect(master)
      osc.connect(og)
      og.gain.setValueAtTime(volume.value * (0.1 + Math.random() * 0.15), c.currentTime)
      og.gain.exponentialRampToValueAtTime(0.001, c.currentTime + 0.18)
      osc.start(); osc.stop(c.currentTime + 0.2)
      nodes.push(osc, og)
      bubble()
    }, delay)
    timers.push(tid)
  }
  bubble(); bubble()

  nodes.push(baseSrc, lfo, lfoG, master)
  return master
}

function playScene(type) {
  if (type === 'thunder') return playThunder()
  if (type === 'forest') return playForest()
  if (type === 'fire') return playFire()
  if (type === 'ocean') return playOcean()
  if (type === 'embers') return playEmbers()
  if (type === 'fireplace') return playFireplace()
  if (type === 'pixel') return playPixel()
  if (type === 'metal') return playMetal()
  if (type === 'gravel') return playGravel()
  if (type === 'underwater') return playUnderwater()
  const c = createCtx()
  const gainNode = c.createGain()
  gainNode.gain.setValueAtTime(0, c.currentTime)
  gainNode.gain.linearRampToValueAtTime(volume.value * 0.4, c.currentTime + 0.5)
  gainNode.connect(c.destination)

  if (type === 'rain') {
    const buf = makeNoise('white')
    const src = c.createBufferSource()
    src.buffer = buf
    src.loop = true

    const filter = c.createBiquadFilter()
    filter.type = 'bandpass'
    filter.frequency.value = 800
    filter.Q.value = 0.5

    const filter2 = c.createBiquadFilter()
    filter2.type = 'highpass'
    filter2.frequency.value = 400

    src.connect(filter)
    filter.connect(filter2)
    filter2.connect(gainNode)
    src.start()
    nodes.push(src, gainNode)
  } else {
    const buf = makeNoise(type)
    const src = c.createBufferSource()
    src.buffer = buf
    src.loop = true

    if (type === 'brown' || type === 'deep') {
      const filter = c.createBiquadFilter()
      filter.type = 'lowpass'
      filter.frequency.value = type === 'deep' ? 300 : 800
      src.connect(filter)
      filter.connect(gainNode)
    } else {
      src.connect(gainNode)
    }

    src.start()
    nodes.push(src, gainNode)
  }

  return gainNode
}

let activeGain = null

function toggle() {
  if (playing.value) {
    stopAll()
    playing.value = false
    activeGain = null
  } else {
    activeGain = playScene(scenes[current.value].type)
    playing.value = true
  }
}

function selectScene(i) {
  stopAll()
  current.value = i
  activeGain = playScene(scenes[i].type)
  playing.value = true
}

function onVolumeChange() {
  if (activeGain) {
    activeGain.gain.setTargetAtTime(volume.value * 0.4, createCtx().currentTime, 0.1)
  }
}

onUnmounted(() => {
  stopAll()
  ctx?.close()
})
</script>

<template>
  <div class="player-card">
    <div class="player-top">
      <div class="player-left">
        <div class="music-icon" :class="{ active: playing }">
          {{ scenes[current].icon }}
        </div>
        <div class="track-info">
          <div class="track-name">{{ scenes[current].name }}</div>
          <div class="track-status">{{ playing ? '● Playing' : 'Paused' }}</div>
        </div>
      </div>
      <button class="play-btn" @click="toggle">{{ playing ? '⏸' : '▶' }}</button>
    </div>

    <div class="scene-list">
      <button
        v-for="(s, i) in scenes"
        :key="i"
        :class="['scene-item', { active: current === i && playing }]"
        @click="selectScene(i)"
      >
        <span class="scene-icon">{{ s.icon }}</span>
        <span>{{ s.name }}</span>
      </button>
    </div>

    <div class="volume-row">
      <span class="vol-label">🔈</span>
      <input
        type="range" min="0" max="1" step="0.05"
        v-model.number="volume"
        @input="onVolumeChange"
        class="vol-slider"
      />
      <span class="vol-label">🔊</span>
    </div>
  </div>
</template>

<style scoped>
.player-card {
  background: var(--bg2);
  border: 1px solid var(--border);
  border-radius: 20px;
  padding: 20px;
  box-shadow: var(--shadow);
}

.player-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 16px;
}

.player-left { display: flex; align-items: center; gap: 12px; }

.music-icon {
  font-size: 22px;
  background: var(--bg3);
  border-radius: 10px;
  padding: 7px 10px;
}
.music-icon.active { animation: pulse 2s ease-in-out infinite; }

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.1); }
}

.track-name { font-size: 14px; font-weight: 600; color: var(--text); }
.track-status { font-size: 12px; color: var(--accent); margin-top: 2px; }

.play-btn {
  background: var(--accent);
  color: white;
  border: none;
  border-radius: 10px;
  padding: 9px 18px;
  font-size: 15px;
  cursor: pointer;
  transition: opacity 0.2s;
}
.play-btn:hover { opacity: 0.85; }

.scene-list {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 6px;
  margin-bottom: 14px;
}

.scene-item {
  display: flex;
  align-items: center;
  gap: 8px;
  background: var(--bg3);
  border: 1px solid transparent;
  border-radius: 10px;
  padding: 9px 12px;
  font-size: 13px;
  color: var(--text2);
  cursor: pointer;
  transition: all 0.15s;
}
.scene-item:hover { color: var(--text); border-color: var(--border); }
.scene-item.active { border-color: var(--accent); color: var(--accent); background: var(--bg3); }

.scene-icon { font-size: 16px; }

.volume-row {
  display: flex;
  align-items: center;
  gap: 10px;
}

.vol-label { font-size: 14px; }

.vol-slider {
  flex: 1;
  height: 4px;
  accent-color: var(--accent);
  cursor: pointer;
}
</style>
