<script setup>
import { ref, watch, onMounted, onUnmounted } from 'vue'

const props = defineProps({
  progress:  { type: Number, default: 0 },
  theme:     { type: String, default: 'dark' },
  treeSeed:  { type: Number, default: 1 }
})

const canvas = ref(null)
let animFrame    = null
let displayProgress = 0
let treeCache    = null

// ============================================================
// RNG
// ============================================================
function makeRng(seed) {
  let s = seed >>> 0
  return () => { s = (Math.imul(1664525, s) + 1013904223) >>> 0; return s / 4294967296 }
}

// ============================================================
// Palettes
// ============================================================
const BG = {
  dark:  { skyTop:'#0d0f1a', skyBot:'#1a1d2e', ground:'#1e2235', groundEdge:'#252840' },
  light: { skyTop:'#e0f2fe', skyBot:'#f0fdf4', ground:'#dcfce7', groundEdge:'#bbf7d0' }
}
const STYLES = [
  { name:'oak',
    trunk:'#7c5c2e', branch:'#6b4c22',
    leaves:['#14532d','#166534','#15803d','#16a34a','#22c55e','#4ade80','#65a30d','#86efac'],
    glowD:['rgba(34,197,94,0.4)','rgba(124,106,247,0.2)'],
    glowL:['rgba(134,239,172,0.5)','rgba(187,247,208,0.35)'] },
  { name:'sakura',
    trunk:'#9c6b4e', branch:'#8b5e42',
    leaves:['#f43f5e','#fb7185','#fda4af','#fecdd3','#ffe4e6','#fff1f2'],
    glowD:['rgba(251,113,133,0.45)','rgba(253,164,175,0.25)'],
    glowL:['rgba(253,164,175,0.5)','rgba(255,228,230,0.4)'] },
  { name:'autumn',
    trunk:'#6b3a1a', branch:'#5c3015',
    leaves:['#7f1d1d','#991b1b','#b91c1c','#c2410c','#ea580c','#f97316','#d97706','#fbbf24'],
    glowD:['rgba(234,88,12,0.45)','rgba(251,191,36,0.25)'],
    glowL:['rgba(253,186,116,0.5)','rgba(254,215,170,0.4)'] }
]

// ============================================================
// Recursive branch generation (from any spawn point)
// Straighter: smaller spread angle
// ============================================================
function genBranch(r, palette, x, y, angle, length, width, depth, revealAt, out, heightFactor = 0.5) {
  if (depth < 0 || length < 3) return

  const x2 = x + Math.cos(angle) * length
  const y2 = y - Math.sin(angle) * length
  const tipW = Math.max(0.3, width * 0.25)

  out.branches.push({ x1:x, y1:y, x2, y2, baseW:width, tipW, color:palette.branch, revealAt, depth })

  // leaves on all branches — more on tips, sparse on thick branches
  const hf2      = heightFactor * heightFactor
  // fewer leaves toward end of growth (but not zero)
  const lateFactor = 0.4 + 0.6 * Math.max(0, 1 - revealAt * 0.85)
  const lCount   = depth <= 2
    ? Math.max(1, Math.floor((1 + hf2 * 10 + r() * 3) * lateFactor))
    : (r() < 0.5 * lateFactor ? 1 : 0)
  const leafAt   = Math.min(0.94, revealAt + 0.13 + r() * 0.06)

  // branch direction vector (normalized)
  const bLen  = Math.sqrt((x2-x)*( x2-x) + (y2-y)*(y2-y)) || 1
  const bdx   = (x2 - x) / bLen
  const bdy   = (y2 - y) / bLen
  // perpendicular to branch
  const bpx   = -bdy
  const bpy   =  bdx

  for (let i = 0; i < lCount; i++) {
    // mostly perpendicular with chaotic spread angle ±90°
    const side    = r() > 0.5 ? 1 : -1
    const scatter = (r() - 0.5) * Math.PI * 0.9  // ±81° around perpendicular
    const ca      = scatter
    const maxSpread = depth <= 2 ? (14 + hf2 * 14) * lateFactor : 6 * lateFactor
    const perpD   = 4 + r() * maxSpread
    const alongD  = (r() - 0.5) * 10
    const lx      = x2 + (bpx * side * Math.cos(ca) - bdx * Math.sin(ca)) * perpD + bdx * alongD
    const ly      = y2 + (bpy * side * Math.cos(ca) - bdy * Math.sin(ca)) * perpD + bdy * alongD
    const ci      = Math.floor(r() * palette.leaves.length)
    out.leaves.push({
      x: lx, y: ly,
      size:     depth <= 2 ? 4 + heightFactor * 5 + r() * 6 : 3 + r() * 3,
      angle:    r() * Math.PI * 2,
      color:    palette.leaves[ci],
      revealAt: leafAt + r() * 0.04
    })
  }

  if (depth === 0) return

  const num  = r() > 0.60 ? 3 : 2
  const spread = 0.28 + r() * 0.28

  for (let i = 0; i < num; i++) {
    if (r() < 0.14) continue
    const sign  = i === 0 ? 1 : (i === 1 ? -1 : (r() > 0.5 ? 0.5 : -0.5))
    const cAngle  = angle + sign * spread + (r() - 0.5) * 0.20
    const cLen    = length * (0.62 + r() * 0.18)
    const cW      = Math.max(0.4, width * (0.55 + r() * 0.08))
    const cReveal = revealAt + 0.11 + r() * 0.11
    genBranch(r, palette, x2, y2, cAngle, cLen, cW, depth - 1, cReveal, out, heightFactor)
  }
}

// ============================================================
// Tree builder — straight trunk + spawn points at any height
// ============================================================
const TRUNK_H  = 108
const TRUNK_BW = 5.5  // half-width base
const TRUNK_TW = 1.8  // half-width top

function buildTree(seed, cx, baseY) {
  const r        = makeRng(seed)
  const palette  = STYLES[Math.floor(r() * STYLES.length)]
  const out      = { branches:[], leaves:[], palette }

  // Spawn definitions: [heightFraction, probability, depth, lengthBase]
  // fracs < 0.4 = lower trunk (rare, short)  — creates character
  // fracs > 0.6 = upper trunk (common, long) — main canopy
  const spawns = [
    [0.28, 0.55, 2,  12],
    [0.48, 0.60, 2,  16],
    [0.65, 0.58, 3,  20],
    [0.80, 0.50, 4,  26],
    [0.92, 0.35, 4,  30],
    [0.99, 0.22, 5,  28],
  ]

  let totalSpawned = 0
  let gapLeft  = 0  // consecutive levels with no left branch
  let gapRight = 0  // consecutive levels with no right branch

  for (const [frac, prob, depth, lenBase] of spawns) {
    const sy       = baseY - frac * TRUNK_H
    const revealAt = 0.02 + frac * 0.82 + r() * 0.06
    let spawnedHere = 0

    for (const side of [-1, 1]) {
      const gap        = side === -1 ? gapLeft : gapRight
      const gapBonus   = Math.min(2.0, 1 + gap * 0.5)  // up to 2× more likely after 2 empty levels
      const adjProb    = Math.min(0.92, prob * gapBonus)
      const mustSpawn  = spawnedHere === 0 && frac <= 0.80 && r() < 0.55
      if (!mustSpawn && r() > adjProb) {
        if (side === -1) gapLeft++; else gapRight++
        continue
      }
      const baseAngle = Math.PI / 2 + side * (0.55 + r() * 0.35)
      const length    = lenBase * (0.5 + r() * 1.1)
      const w = 0.7 + frac * 2.2 + r() * 0.5
      genBranch(r, palette, cx, sy, baseAngle, length, w, depth, revealAt, out, frac)
      spawnedHere++
      totalSpawned++
      if (side === -1) gapLeft = 0; else gapRight = 0
    }
  }

  // if somehow still bare (very unlucky seed), force 3 mid-trunk branches
  if (totalSpawned === 0) {
    for (const [frac, , depth, lenBase] of spawns.slice(2, 5)) {
      const sy       = baseY - frac * TRUNK_H
      const revealAt = 0.02 + frac * 0.82 + r() * 0.06
      for (const side of [-1, 1]) {
        const baseAngle = Math.PI / 2 + side * (0.55 + r() * 0.35)
        genBranch(r, palette, cx, sy, baseAngle, lenBase * (0.5 + r() * 1.1),
          0.7 + frac * 2.2, depth, revealAt, out, frac)
      }
    }
  }

  // sparse leaves directly on trunk
  for (let i = 0; i < 5; i++) {
    const frac = 0.25 + r() * 0.75
    const side = r() > 0.5 ? 1 : -1
    const cx2  = cx + side * (TRUNK_BW * (1 - frac) + 2 + r() * 4)
    const cy2  = baseY - frac * TRUNK_H
    const ci   = Math.floor(r() * palette.leaves.length)
    out.leaves.push({
      x: cx2, y: cy2,
      size:     3 + r() * 4,
      angle:    r() * Math.PI * 2,
      color:    palette.leaves[ci],
      revealAt: 0.22 + frac * 0.55 + r() * 0.06
    })
  }

  return out
}

function getTree(w, h) {
  if (!treeCache) treeCache = buildTree(props.treeSeed, w / 2, h - 28)
  return treeCache
}

// ============================================================
// Draw helpers
// ============================================================
function drawTapered(ctx, x1, y1, x2, y2, bw, tw, color, alpha) {
  const dx = x2-x1, dy = y2-y1
  const len = Math.sqrt(dx*dx + dy*dy)
  if (len < 0.5) return
  const nx = -dy/len, ny = dx/len
  ctx.save()
  ctx.globalAlpha = alpha
  ctx.beginPath()
  ctx.moveTo(x1+nx*bw, y1+ny*bw)
  ctx.lineTo(x2+nx*tw, y2+ny*tw)
  ctx.lineTo(x2-nx*tw, y2-ny*tw)
  ctx.lineTo(x1-nx*bw, y1-ny*bw)
  ctx.closePath()
  ctx.fillStyle = color
  ctx.fill()
  ctx.restore()
}

function drawLeafShape(ctx, x, y, sz, angle, color, alpha) {
  ctx.save()
  ctx.globalAlpha = alpha
  ctx.translate(x, y); ctx.rotate(angle)
  ctx.beginPath()
  ctx.moveTo(0, -sz)
  ctx.bezierCurveTo( sz*0.65,-sz*0.5,  sz*0.65, sz*0.35, 0, sz*0.42)
  ctx.bezierCurveTo(-sz*0.65, sz*0.35,-sz*0.65,-sz*0.5,  0,-sz)
  ctx.fillStyle = color; ctx.fill()
  ctx.restore()
}

function drawFlower(ctx, x, y, sz, color, alpha) {
  ctx.save()
  ctx.globalAlpha = alpha
  ctx.translate(x, y)
  for (let i = 0; i < 5; i++) {
    ctx.save(); ctx.rotate(i*Math.PI*2/5)
    ctx.beginPath()
    ctx.ellipse(0,-sz*0.6, sz*0.28,sz*0.44, 0, 0, Math.PI*2)
    ctx.fillStyle = color; ctx.fill(); ctx.restore()
  }
  ctx.beginPath(); ctx.arc(0,0,sz*0.2,0,Math.PI*2)
  ctx.fillStyle='#fef9c3'; ctx.fill()
  ctx.restore()
}

// ============================================================
// Trunk — straight rectangle, grows upward
// ============================================================
function drawTrunk(ctx, prog, cx, baseY, palette) {
  const t = Math.min(1, Math.max(0, (prog - 0.03) / 0.25))
  if (t <= 0) return

  const currentH = TRUNK_H * t
  const topY     = baseY - currentH
  // start thin like seedling (0.8px half-width), grow to full TRUNK_BW
  const wt      = Math.min(1, t * 2.0)
  const hw_base = 0.8 + (TRUNK_BW - 0.8) * wt
  const hw_top  = 0.5 + (TRUNK_TW  - 0.5) * wt

  ctx.save()
  ctx.globalAlpha = Math.min(1, t * 3.5)
  ctx.beginPath()
  ctx.moveTo(cx - hw_base, baseY)
  ctx.lineTo(cx - hw_top,  topY)
  ctx.lineTo(cx + hw_top,  topY)
  ctx.lineTo(cx + hw_base, baseY)
  ctx.closePath()
  ctx.fillStyle = palette.trunk
  ctx.fill()
  ctx.restore()
}

// ============================================================
// Seedling (fades out as trunk appears)
// ============================================================
function drawSeedling(ctx, prog, cx, baseY, palette) {
  const alpha = Math.min(1, prog/0.04) * Math.max(0, 1-(prog-0.04)/0.26)
  if (alpha <= 0) return

  const stemH = 12 + prog * 60
  const ty    = baseY - stemH

  ctx.save(); ctx.globalAlpha = alpha
  ctx.beginPath(); ctx.moveTo(cx, baseY); ctx.lineTo(cx, ty)
  ctx.strokeStyle = palette.branch; ctx.lineWidth = 2; ctx.lineCap='round'; ctx.stroke()
  ctx.restore()

  const lf = (ox,oy,sz,a,ci) => drawLeafShape(ctx, cx+ox, ty+oy, sz, a, palette.leaves[ci%palette.leaves.length], alpha)
  if (prog>0.01){ lf(-7,12, 6,-0.4,2); lf(7,14, 5, 2.8,1) }
  if (prog>0.04){ lf(-6, 4, 6,-0.6,3); lf(6, 6, 5, 2.6,2) }
  if (prog>0.07){ lf( 0,-2, 6, 0.1,4) }
  if (prog>0.11){ lf(-8, 8, 7,-0.3,0); lf(8, 9, 6, 3.0,3) }
  if (prog>0.15){ lf(-5,-6, 5,-0.7,1); lf(5,-5, 5, 2.5,4) }
}

// ============================================================
// Background & glow
// ============================================================
function drawBg(ctx, w, h, bgP) {
  const sky = ctx.createLinearGradient(0,0,0,h)
  sky.addColorStop(0,bgP.skyTop); sky.addColorStop(1,bgP.skyBot)
  ctx.fillStyle=sky; ctx.fillRect(0,0,w,h)

  if (props.theme === 'dark') {
    const r3 = makeRng(77)
    for (let i = 0; i < 24; i++) {
      const sx=r3()*w, sy=r3()*h*0.55
      const tw=Math.sin(Date.now()/900+i*1.9)*0.25+0.55
      ctx.save(); ctx.globalAlpha=tw*0.45
      ctx.beginPath(); ctx.arc(sx,sy,0.7+r3()*1.2,0,Math.PI*2)
      ctx.fillStyle='rgba(255,255,255,0.8)'; ctx.fill(); ctx.restore()
    }
  }

  const gg = ctx.createLinearGradient(0,h-36,0,h)
  gg.addColorStop(0,bgP.groundEdge); gg.addColorStop(1,bgP.ground)
  ctx.fillStyle=gg
  ctx.beginPath(); ctx.ellipse(w/2,h-8,90,22,0,0,Math.PI*2); ctx.fill()
}

function drawGlow(ctx, w, h, prog, palette) {
  if (prog < 0.75) return
  const t = Math.min(1,(prog-0.75)/0.25)
  const isLight = props.theme==='light'
  const [c1,c2] = isLight ? palette.glowL : palette.glowD
  ctx.save()
  ctx.globalAlpha = t*(isLight?0.45:0.55)
  const g1=ctx.createRadialGradient(w/2,h*0.28,8,w/2,h*0.28,120)
  g1.addColorStop(0,c1); g1.addColorStop(1,'transparent')
  ctx.fillStyle=g1; ctx.fillRect(0,0,w,h)
  ctx.globalAlpha=t*(isLight?0.25:0.3)
  const g2=ctx.createRadialGradient(w/2,h*0.5,20,w/2,h*0.5,160)
  g2.addColorStop(0,c2); g2.addColorStop(1,'transparent')
  ctx.fillStyle=g2; ctx.fillRect(0,0,w,h)
  ctx.restore()
}

// ============================================================
// Main render
// ============================================================
function render(prog) {
  const cvs = canvas.value; if (!cvs) return
  const ctx = cvs.getContext('2d')
  const w=cvs.width, h=cvs.height
  const tree = getTree(w, h)
  const p    = tree.palette
  const isSakura = p.name==='sakura'

  ctx.clearRect(0,0,w,h)
  drawBg(ctx, w, h, BG[props.theme])
  drawTrunk(ctx, prog, w/2, h-28, p)
  drawSeedling(ctx, prog, w/2, h-28, p)

  // branches grow from base toward tip
  for (const b of tree.branches) {
    const t = Math.min(1, Math.max(0, (prog - b.revealAt) / 0.13))
    if (t <= 0) continue
    const ex = b.x1 + (b.x2-b.x1)*t
    const ey = b.y1 + (b.y2-b.y1)*t
    const tw = b.tipW*(0.3+0.7*t)
    drawTapered(ctx, b.x1,b.y1, ex,ey, b.baseW,tw, b.color, Math.min(1,t*2.2))
  }

  // leaves / flowers
  for (const lf of tree.leaves) {
    const alpha = Math.min(1, Math.max(0,(prog-lf.revealAt)/0.05))
    if (alpha<=0) continue
    const m = lf.size
    if (lf.x - m < 0 || lf.x + m > w || lf.y - m < 0 || lf.y + m > h) continue
    if (isSakura) drawFlower(ctx, lf.x,lf.y, lf.size*0.9, lf.color, alpha*0.92)
    else          drawLeafShape(ctx, lf.x,lf.y, lf.size, lf.angle, lf.color, alpha*0.9)
  }

  drawGlow(ctx, w, h, prog, p)
}

// ============================================================
// Animation
// ============================================================
function animateTo(target) {
  cancelAnimationFrame(animFrame)
  const start=displayProgress, t0=performance.now(), dur=1400
  function step(now) {
    const t=Math.min((now-t0)/dur,1)
    displayProgress = start+(target-start)*(1-Math.pow(1-t,4))
    render(displayProgress)
    if(t<1) animFrame=requestAnimationFrame(step)
    else displayProgress=target
  }
  animFrame=requestAnimationFrame(step)
}

watch(()=>props.progress, v=>animateTo(v))
watch(()=>props.theme, ()=>{ treeCache=null; render(displayProgress) })
watch(()=>props.treeSeed, ()=>{ treeCache=null; cancelAnimationFrame(animFrame); displayProgress=0; render(0) })
onMounted(()=>{ displayProgress=props.progress; render(props.progress) })
onUnmounted(()=>cancelAnimationFrame(animFrame))
</script>

<template>
  <div class="tree-wrap">
    <canvas ref="canvas" width="340" height="300" class="tree-canvas" />
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
}
.tree-canvas { width:100%; max-width:340px; height:auto; display:block; }
</style>
