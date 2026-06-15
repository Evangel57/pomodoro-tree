<script setup>
import { ref, watch, onMounted, onUnmounted } from 'vue'

const props = defineProps({
  progress: { type: Number, default: 0 },
  theme:    { type: String, default: 'dark' },
  treeSeed: { type: Number, default: 1 }
})

const canvas     = ref(null)
const glowCanvas = ref(null)
let animFrame = null
let displayProgress = 0
let treeCache = null

const GLOW_COLORS = {
  oak:    { d: 'rgba(34,197,94,0.45)',   l: 'rgba(134,239,172,0.5)' },
  sakura: { d: 'rgba(251,113,133,0.5)',  l: 'rgba(253,164,175,0.55)' },
  autumn: { d: 'rgba(234,88,12,0.5)',    l: 'rgba(253,186,116,0.5)' },
  ghost:  { d: 'rgba(56,189,248,0.55)',  l: 'rgba(147,197,253,0.5)' },
  neon:   { d: 'rgba(168,85,247,0.6)',   l: 'rgba(52,211,153,0.5)' },
  void:   { d: 'rgba(139,92,246,0.65)',  l: 'rgba(167,139,250,0.55)' },
}

// ========== PIXEL CANVAS CONFIG ==========
const PW = 85
const PH = 75
// generation space is 340×300, scale down by 4
const SX = PW / 340
const SY = PH / 300

const TRUNK_H  = 220
const TRUNK_BW = 5.5

// ========== RNG ==========
function makeRng(seed) {
  let s = seed >>> 0
  return () => { s = (Math.imul(1664525, s) + 1013904223) >>> 0; return s / 4294967296 }
}

const _sr = makeRng(77)
const STARS = Array.from({ length: 22 }, () => ({
  x: Math.round(_sr() * 84) * 4,   // snapped to 4px grid
  y: Math.round(_sr() * 48) * 4,
  size: 4,
  phase: _sr() * Math.PI * 2,
  speed: 1.5 + _sr() * 2.5,
}))

// ========== PALETTES ==========
const COMMON_STYLES = [
  { name:'oak', trunk:'#7c5c2e', branch:'#6b4c22',
    leaves:['#14532d','#166534','#15803d','#16a34a','#22c55e','#4ade80','#65a30d'] },
  { name:'sakura', trunk:'#9c6b4e', branch:'#8b5e42',
    leaves:['#f43f5e','#fb7185','#fda4af','#fecdd3','#ffe4e6'] },
  { name:'autumn', trunk:'#6b3a1a', branch:'#5c3015',
    leaves:['#7f1d1d','#b91c1c','#c2410c','#ea580c','#f97316','#d97706','#fbbf24'] },
]

const RARE_STYLES = [
  { name:'ghost', trunk:'#4a6fa5', branch:'#3a5f95',
    leaves:['#f0f9ff','#e0f2fe','#bae6fd','#7dd3fc','#38bdf8','#93c5fd','#dbeafe'] },
  { name:'neon', trunk:'#1e1b4b', branch:'#312e81',
    leaves:['#a855f7','#ec4899','#06b6d4','#10b981','#f59e0b','#8b5cf6','#34d399'] },
  { name:'void', trunk:'#1e1b4b', branch:'#2e1065',
    leaves:['#4c1d95','#5b21b6','#6d28d9','#7c3aed','#8b5cf6','#a78bfa','#c4b5fd'] },
]

function pickStyle(r) {
  if (r() < 0.15) return RARE_STYLES[Math.floor(r() * RARE_STYLES.length)]
  return COMMON_STYLES[Math.floor(r() * COMMON_STYLES.length)]
}

// ========== TREE GENERATION ==========
function genBranch(r, palette, x, y, angle, length, width, depth, revealAt, out, heightFactor = 0.5) {
  if (heightFactor > 0.80) length = Math.min(length, 16 + (1 - heightFactor) * 30)
  if (depth < 0 || length < 3) return
  const x2 = x + Math.cos(angle) * length
  const y2 = y - Math.sin(angle) * length
  const branchObj = { x1:x, y1:y, x2, y2, color:palette.branch, revealAt, depth }
  out.branches.push(branchObj)

  const hf2 = heightFactor * heightFactor
  const lCount = depth <= 2
    ? Math.max(2, Math.floor(3 + hf2 * 12 + r() * 4))
    : Math.max(2, Math.floor(2 + heightFactor * 5 + r() * 3))

  const bLen = Math.sqrt((x2-x)*(x2-x) + (y2-y)*(y2-y)) || 1
  const bdx = (x2-x)/bLen, bdy = (y2-y)/bLen
  const bpx = -bdy,        bpy =  bdx

  for (let i = 0; i < lCount; i++) {
    const side    = r() > 0.5 ? 1 : -1
    const scatter = (r()-0.5) * Math.PI * 0.9
    const maxSpread = depth <= 2 ? 14 + hf2 * 14 : 6 + heightFactor * 6
    const perpD   = 4 + r()*maxSpread
    const alongD  = (r()-0.5)*10
    const lx = x2 + (bpx*side*Math.cos(scatter) - bdx*Math.sin(scatter))*perpD + bdx*alongD
    const ly = y2 + (bpy*side*Math.cos(scatter) - bdy*Math.sin(scatter))*perpD + bdy*alongD
    out.leaves.push({
      x: lx, y: ly,
      color: palette.leaves[Math.floor(r()*palette.leaves.length)],
      revealAt,
      parentBranch: branchObj
    })
  }

  if (depth === 0) return
  const num    = r() < 0.25 ? 1 : (r() < 0.65 ? 2 : 3)
  const spread = 0.30 + r()*0.60
  for (let i = 0; i < num; i++) {
    if (r() < 0.08) continue
    const sign   = i === 0 ? 1 : (i === 1 ? -1 : (r()>0.5 ? 0.5 : -0.5))
    const cAngle = angle + sign*spread + (r()-0.5)*0.35
    const cLen   = length * (0.55 + r()*0.35)
    const cW     = Math.max(0.4, width*(0.55 + r()*0.08))
    genBranch(r, palette, x2, y2, cAngle, cLen, cW, depth-1, revealAt+0.11+r()*0.11, out, heightFactor)
  }
}

function buildTree(seed, cx, baseY) {
  const r       = makeRng(seed)
  const palette = pickStyle(r)
  const out     = { branches:[], leaves:[], palette }
  const spawns  = [
    [0.18, 0.50, 2, 18], [0.32, 0.70, 2, 24],
    [0.48, 0.75, 3, 30], [0.60, 0.65, 3, 36],
    [0.72, 0.70, 4, 44], [0.84, 0.60, 4, 48],
    [0.93, 0.50, 4, 32], [0.99, 0.35, 4, 26],
  ]
  let totalSpawned = 0, gapLeft = 0, gapRight = 0

  for (const [frac, prob, depth, lenBase] of spawns) {
    const sy      = baseY - frac*TRUNK_H
    const revealAt = 0.03 + frac*0.84 + r()*0.05
    let spawnedHere = 0
    for (const side of [-1, 1]) {
      const gap       = side === -1 ? gapLeft : gapRight
      const sideVar   = side === -1 ? r()*0.25 : -r()*0.15  // left grows more freely
      const adjProb   = Math.min(0.95, (prob + sideVar) * Math.min(2.0, 1+gap*0.5))
      const mustSpawn = spawnedHere === 0 && frac <= 0.84 && r() < 0.65
      if (!mustSpawn && r() > adjProb) { if(side===-1)gapLeft++; else gapRight++; continue }
      genBranch(r, palette, cx, sy, Math.PI/2+side*(0.60+r()*0.80),
        lenBase*(0.5+r()*1.1), 0.7+frac*2.2+r()*0.5, depth, revealAt, out, frac)
      spawnedHere++; totalSpawned++
      if(side===-1) gapLeft=0; else gapRight=0
    }
  }
  if (totalSpawned === 0) {
    for (const [frac,,depth,lenBase] of spawns.slice(2,5)) {
      const sy = baseY - frac*TRUNK_H
      const rv = 0.02 + frac*0.82 + r()*0.06
      for (const side of [-1,1])
        genBranch(r, palette, cx, sy, Math.PI/2+side*(0.60+r()*0.80),
          lenBase*(0.5+r()*1.1), 0.7+frac*2.2, depth, rv, out, frac)
    }
  }
  return out
}

function getTree() {
  if (!treeCache) treeCache = buildTree(props.treeSeed, 158, 272)
  return treeCache
}

// ========== PIXEL DRAWING ==========
const rgbCache = {}
function hexRgb(hex) {
  if (rgbCache[hex]) return rgbCache[hex]
  const v = parseInt(hex.slice(1), 16)
  return rgbCache[hex] = [(v>>16)&255, (v>>8)&255, v&255]
}

function setPixel(buf, x, y, rgb, a=255) {
  x=x|0; y=y|0
  if (x<0||x>=PW||y<0||y>=PH) return
  const i=(y*PW+x)*4
  buf[i]=rgb[0]; buf[i+1]=rgb[1]; buf[i+2]=rgb[2]; buf[i+3]=a
}

function bline(buf, x0, y0, x1, y1, color, alpha=255) {
  const rgb=hexRgb(color)
  x0=Math.round(x0)|0; y0=Math.round(y0)|0
  x1=Math.round(x1)|0; y1=Math.round(y1)|0
  let dx=Math.abs(x1-x0), dy=Math.abs(y1-y0)
  let sx=x0<x1?1:-1, sy=y0<y1?1:-1, err=dx-dy
  for(;;){
    setPixel(buf,x0,y0,rgb,alpha)
    if(x0===x1&&y0===y1) break
    const e2=2*err
    if(e2>-dy){err-=dy;x0+=sx}
    if(e2<dx){err+=dx;y0+=sy}
  }
}

function leafCluster(buf, x, y, color, alpha=255) {
  const rgb=hexRgb(color)
  const cx=Math.round(x)|0, cy=Math.round(y)|0
  setPixel(buf,cx,  cy,  rgb,alpha)
  setPixel(buf,cx+1,cy,  rgb,alpha)
  setPixel(buf,cx,  cy+1,rgb,alpha)
  setPixel(buf,cx-1,cy,  rgb,Math.round(alpha*0.7))
  setPixel(buf,cx,  cy-1,rgb,Math.round(alpha*0.7))
}

// ========== BACKGROUND ==========
function drawBg(buf) {
  const dark = props.theme === 'dark'
  const skyRgb  = hexRgb(dark ? '#141728' : '#e8f4fd')
  const sky2Rgb = hexRgb(dark ? '#1c2540' : '#f0fdf4')
  const grRgb   = hexRgb(dark ? '#232844' : '#dcfce7')
  const geRgb   = hexRgb(dark ? '#2c3358' : '#bbf7d0')

  for (let y=0; y<PH-7; y++) {
    const t = y/(PH-7)
    const r = Math.round(skyRgb[0] + (sky2Rgb[0]-skyRgb[0])*t)
    const g = Math.round(skyRgb[1] + (sky2Rgb[1]-skyRgb[1])*t)
    const b = Math.round(skyRgb[2] + (sky2Rgb[2]-skyRgb[2])*t)
    for (let x=0; x<PW; x++) setPixel(buf,x,y,[r,g,b])
  }
  for (let y=PH-7; y<PH; y++)
    for (let x=0; x<PW; x++) setPixel(buf,x,y,grRgb)
  for (let x=0; x<PW; x++) setPixel(buf,x,PH-7,geRgb)

}

// ========== TRUNK ==========
function drawTrunk(buf, prog, palette) {
  const t = Math.min(1, Math.max(0, (prog-0.03)/0.90))
  if (t <= 0) return
  const rgb   = hexRgb(palette.trunk)
  const pcx   = Math.round(158*SX)
  const pbase = Math.round(272*SY)
  const h     = Math.round(TRUNK_H*SY*t)
  const alpha = Math.min(255, Math.round(Math.min(1,t*3.5)*255))
  // width grows with t: starts at 1px (matches seedling), expands to 3px
  const wideBase = t > 0.5

  for (let y=pbase; y>=pbase-h; y--) {
    const fromBase = pbase - y
    setPixel(buf, pcx, y, rgb, alpha)
    if (wideBase && fromBase < 8) {
      setPixel(buf, pcx-1, y, rgb, alpha)
      setPixel(buf, pcx+1, y, rgb, alpha)
    } else if (wideBase && fromBase < 16) {
      setPixel(buf, pcx-1, y, rgb, Math.round(alpha*0.7))
    }
  }
}

// ========== SEEDLING ==========
function drawSeedling(buf, prog, palette) {
  const alpha = Math.min(1,prog/0.04) * Math.max(0,1-(prog-0.04)/0.38)
  if (alpha <= 0) return
  const a   = Math.round(alpha*255)
  const pcx = Math.round(158*SX)
  const pby = Math.round(272*SY)
  const stemH = Math.round((12+prog*60)*SY)
  const rgb = hexRgb(palette.branch)

  for (let y=pby; y>=pby-stemH; y--) setPixel(buf,pcx,y,rgb,a)

  const ty = pby - stemH
  const lc = palette.leaves
  const la = Math.round(alpha*200)
  if (prog>0.01){ leafCluster(buf,pcx-2,ty+3,lc[2],la); leafCluster(buf,pcx+2,ty+3,lc[1],la) }
  if (prog>0.04){ leafCluster(buf,pcx-2,ty+1,lc[3],la); leafCluster(buf,pcx+2,ty+2,lc[2],la) }
  if (prog>0.07){ leafCluster(buf,pcx,  ty-1,lc[4%lc.length],la) }
  if (prog>0.11){ leafCluster(buf,pcx-3,ty+2,lc[0],la); leafCluster(buf,pcx+3,ty+2,lc[3],la) }
}


// ========== MAIN RENDER ==========
function render(prog) {
  const cvs = canvas.value; if (!cvs) return
  const ctx = cvs.getContext('2d')
  const tree = getTree()
  const p    = tree.palette

  const imgData = ctx.createImageData(PW, PH)
  const buf     = imgData.data

  drawBg(buf)
  drawSeedling(buf, prog, p)
  drawTrunk(buf, prog, p)

  for (const b of tree.branches) {
    const t = Math.min(1, Math.max(0,(prog-b.revealAt)/0.13))
    if (t <= 0) continue
    const ex = (b.x1+(b.x2-b.x1)*t)*SX
    const ey = (b.y1+(b.y2-b.y1)*t)*SY
    const alpha = Math.min(255,Math.round(Math.min(1,t*2.2)*255))
    bline(buf, b.x1*SX,b.y1*SY, ex,ey, b.color, alpha)
  }

  for (const lf of tree.leaves) {
    const b  = lf.parentBranch
    const bt = b ? Math.min(1, Math.max(0, (prog - b.revealAt) / 0.13)) : 0
    if (bt <= 0) continue

    let px, py, alpha
    if (bt < 1) {
      // follow branch tip while growing
      px    = (b.x1 + (b.x2 - b.x1) * bt) * SX
      py    = (b.y1 + (b.y2 - b.y1) * bt) * SY
      alpha = Math.min(1, bt * 3)
    } else {
      // branch done — settle at final position
      px    = lf.x * SX
      py    = lf.y * SY
      alpha = 1
    }

    if (px<0||px>=PW||py<0||py>=PH) continue
    leafCluster(buf, px, py, lf.color, Math.round(alpha*230))
  }

  ctx.putImageData(imgData, 0, 0)
}

// ========== ANIMATION ==========
function animateTo(target) {
  cancelAnimationFrame(animFrame)
  const start=displayProgress, t0=performance.now(), dur=1400
  function step(now) {
    const t=Math.min((now-t0)/dur,1)
    displayProgress=start+(target-start)*(1-Math.pow(1-t,4))
    render(displayProgress)
    if(t<1) animFrame=requestAnimationFrame(step)
    else displayProgress=target
  }
  animFrame=requestAnimationFrame(step)
}

// ========== CELEBRATION ==========
let celebrationStart = null
let celebParticles = []

function spawnCelebration() {
  celebrationStart = performance.now()
  const _r = makeRng(props.treeSeed + 1)
  celebParticles = Array.from({ length: 28 }, () => ({
    x: 158 * SX + (_r() - 0.5) * 20,
    y: 272 * SY - TRUNK_H * SY * 0.6 + (_r() - 0.5) * 16,
    vx: (_r() - 0.5) * 3.5,
    vy: -1.5 - _r() * 3,
    color: getTree().palette.leaves[Math.floor(_r() * getTree().palette.leaves.length)],
    size: _r() > 0.5 ? 8 : 4,
  }))
}

function drawCelebration(gcvs) {
  if (!celebrationStart || !celebParticles.length) return
  const elapsed = (performance.now() - celebrationStart) / 1000
  if (elapsed > 2.5) { celebrationStart = null; celebParticles = []; return }

  const ctx = gcvs.getContext('2d')
  for (const p of celebParticles) {
    const t     = elapsed / 2.5
    const alpha = Math.max(0, 1 - t * 1.2)
    const px    = p.x * 4 + p.vx * elapsed * 60
    const py    = p.y * 4 + p.vy * elapsed * 60 + 80 * elapsed * elapsed
    ctx.globalAlpha = alpha
    ctx.fillStyle   = p.color
    ctx.fillRect(Math.round(px / p.size) * p.size, Math.round(py / p.size) * p.size, p.size, p.size)
  }
  ctx.globalAlpha = 1
}

let starFrame = null
function starLoop() {
  const gcvs = glowCanvas.value; if (!gcvs) { starFrame = requestAnimationFrame(starLoop); return }
  const ctx  = gcvs.getContext('2d')
  ctx.clearRect(0, 0, gcvs.width, gcvs.height)

  if (props.theme === 'dark') {
    const now = performance.now() / 1000
    ctx.fillStyle = '#ffffff'
    for (const s of STARS) {
      const a = 0.25 + 0.65 * (0.5 + 0.5 * Math.sin(now * s.speed + s.phase))
      ctx.globalAlpha = a
      ctx.fillRect(s.x, s.y, s.size, s.size)
    }
    ctx.globalAlpha = 1
  }

  if (displayProgress >= 0.75) {
    const palette = getTree().palette
    const t   = Math.min(1, (displayProgress - 0.75) / 0.25)
    const key = props.theme === 'light' ? 'l' : 'd'
    const c   = GLOW_COLORS[palette.name]?.[key] ?? GLOW_COLORS.oak[key]
    const W = gcvs.width, H = gcvs.height
    ctx.globalAlpha = t * 0.55
    const g1 = ctx.createRadialGradient(W/2, H*0.55, 6, W/2, H*0.55, 90)
    g1.addColorStop(0, c); g1.addColorStop(1, 'transparent')
    ctx.fillStyle = g1; ctx.fillRect(0, 0, W, H)
    ctx.globalAlpha = t * 0.3
    const g2 = ctx.createRadialGradient(W/2, H*0.68, 10, W/2, H*0.68, 130)
    g2.addColorStop(0, c); g2.addColorStop(1, 'transparent')
    ctx.fillStyle = g2; ctx.fillRect(0, 0, W, H)
    ctx.globalAlpha = 1
  }

  drawCelebration(gcvs)
  starFrame = requestAnimationFrame(starLoop)
}

watch(()=>props.progress, v=>{
  const wasComplete = displayProgress >= 1
  animateTo(v)
  if (v >= 1 && !wasComplete) spawnCelebration()
})
watch(()=>props.theme, ()=>{ render(displayProgress) })
watch(()=>props.treeSeed, ()=>{ treeCache=null; cancelAnimationFrame(animFrame); displayProgress=0; render(0) })
onMounted(()=>{ displayProgress=props.progress; render(props.progress); starLoop() })
onUnmounted(()=>{ cancelAnimationFrame(animFrame); cancelAnimationFrame(starFrame) })
</script>

<template>
  <div class="tree-wrap">
    <div class="canvas-stack">
      <canvas ref="canvas" :width="PW" :height="PH" class="tree-canvas" />
      <canvas ref="glowCanvas" width="340" height="300" class="glow-canvas" />
    </div>
  </div>
</template>

<style scoped>
.tree-wrap {
  background: var(--bg2);
  border: 1px solid var(--border);
  border-radius: 20px;
  overflow: hidden;
  box-shadow: var(--shadow);
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
}

.canvas-stack {
  position: relative;
  width: 100%;
  max-width: 340px;
  aspect-ratio: 340 / 300;
}

.tree-canvas {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  display: block;
  image-rendering: pixelated;
  image-rendering: crisp-edges;
}

.glow-canvas {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  display: block;
  pointer-events: none;
}
</style>
