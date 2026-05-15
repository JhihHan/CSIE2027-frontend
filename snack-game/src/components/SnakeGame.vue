<template>
  <main class="container">
    <div class="center-box">
      <h1>貪食蛇 (20×20)</h1>

      <div class="status-row">
        <div class="state">狀態: <strong>{{ playStateLabel }}</strong></div>
        <div class="score-inline">分數: <span class="score-value">{{ score }}</span></div>
      </div>

      <div class="center-wrapper">
        <div class="center-box">
          <div class="game-area">
            <div class="canvas-wrap">
              <canvas ref="canvasRef"></canvas>
            </div>
          </div>

          <div class="controls-row">
            <button @click="start">開始</button>
            <button @click="pause">暫停</button>
            <button @click="reset">重置</button>
          </div>
          <div class="info">
            <div class="hint">使用方向鍵或 W A S D 控制，無法直接反向</div>
          </div>
        </div>

        <aside class="legend">
          <h3>食物說明</h3>
          <ul>
            <li><span class="swatch normal"></span> 普通：+1 分</li>
            <li><span class="swatch gold"></span> 黃金：+3 分</li>
            <li><span class="swatch poison"></span> 毒物：-1 分，縮短蛇身</li>
          </ul>
          <h3>道具</h3>
          <ul>
            <li><span class="swatch speed"></span> 加速：暫時加速</li>
            <li><span class="swatch slow"></span> 減速：暫時變慢</li>
            <li><span class="swatch invincible"></span> 無敵：暫時不會被自己咬到</li>
          </ul>
        </aside>
      </div>

      
    </div>
  </main>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, computed } from 'vue'

interface Point { x: number; y: number }

const gridSize = 20
const cellSize = 20
const tickIntervalDefault = 150

const canvasRef = ref<HTMLCanvasElement | null>(null)
let ctx: CanvasRenderingContext2D | null = null

const snake = ref<Point[]>([])
type FoodType = 'normal' | 'gold' | 'poison'
const food = ref<{ x: number; y: number; type: FoodType }>({ x: 0, y: 0, type: 'normal' })
const direction = ref<Point>({ x: 1, y: 0 })
const nextDirection = ref<Point>({ x: 1, y: 0 })
const score = ref(0)
const running = ref(false)
const isGameOver = ref(false)
const gameOverMessage = ref('')
const playState = ref<'ready'|'running'|'paused'|'gameover'>('ready')
// combo multiplier
const comboMultiplier = ref(1)
let lastEatTime: number | null = null
const comboTimeout = 2000 // ms to continue combo

// power-ups
type PowerUpType = 'speed' | 'slow' | 'invincible'
const powerUps = ref<{ x:number; y:number; type: PowerUpType; duration: number }[]>([])
const activeEffects = ref<Record<string, number>>({}) // effect -> expiry timestamp
let powerUpSpawnTimer: number | null = null
let comboTimer: number | null = null

const playStateLabel = computed(() => {
  switch(playState.value){
    case 'running': return '遊玩中'
    case 'paused': return '暫停中'
    case 'gameover':
      return gameOverMessage.value ? `已結束 — ${gameOverMessage.value}` : '已結束'
    default: return '準備中'
  }
})
let timerId: number | null = null
let tickInterval = tickIntervalDefault
let foodTimer: number | null = null
const foodLifetime = 5000 // ms before food disappears and respawns

function currentSpeedFactor(){
  const now = Date.now()
  if(activeEffects.value['speed'] && activeEffects.value['speed'] > now) return 0.6
  if(activeEffects.value['slow'] && activeEffects.value['slow'] > now) return 1.6
  return 1
}

function updateTimer(){
  if(timerId != null) window.clearInterval(timerId)
  if(running.value){
    const factor = currentSpeedFactor()
    timerId = window.setInterval(moveSnake, Math.max(40, Math.round(tickIntervalDefault * factor)))
  }else{
    timerId = null
  }
}

function initGame(){
  snake.value = [ {x:10,y:10}, {x:9,y:10}, {x:8,y:10} ]
  direction.value = {x:1,y:0}
  nextDirection.value = {x:1,y:0}
  score.value = 0
  placeFood()
  draw()
}

function placeFood(){
  const occupied = new Set(snake.value.map(p => `${p.x},${p.y}`))
  let x:number, y:number
  // pick a random empty cell
  do{
    x = Math.floor(Math.random()*gridSize)
    y = Math.floor(Math.random()*gridSize)
  }while(occupied.has(`${x},${y}`))

  // determine food type (weights)
  const r = Math.random()
  let type: FoodType = 'normal'
  if(r < 0.1) type = 'poison'
  else if(r < 0.25) type = 'gold'
  else type = 'normal'

  food.value = {x,y,type}
  // reset food lifetime timer
  if(foodTimer) window.clearTimeout(foodTimer)
  foodTimer = window.setTimeout(()=>{
    // if food still at same position (not eaten), respawn
    placeFood()
  }, foodLifetime)
}

function draw(){
  if(!ctx || !canvasRef.value) return
  const canvas = canvasRef.value
  ctx.fillStyle = '#fff'
  ctx.fillRect(0,0,canvas.width,canvas.height)

  ctx.strokeStyle = '#f0f0f0'
  for(let i=0;i<=gridSize;i++){
    const pos = i*cellSize
    ctx.beginPath(); ctx.moveTo(pos,0); ctx.lineTo(pos,canvas.height); ctx.stroke();
    ctx.beginPath(); ctx.moveTo(0,pos); ctx.lineTo(canvas.width,pos); ctx.stroke();
  }

  ctx.fillStyle = '#e63946'
  // draw food by type
  if(food.value.type === 'normal') ctx.fillStyle = '#e63946'
  else if(food.value.type === 'gold') ctx.fillStyle = '#f4c430'
  else if(food.value.type === 'poison') ctx.fillStyle = '#6a1b9a'
  ctx.fillRect(food.value.x*cellSize+1, food.value.y*cellSize+1, cellSize-2, cellSize-2)

  // draw power-ups
  for(const pu of powerUps.value){
    ctx.fillStyle = pu.type === 'speed' ? '#1e90ff' : pu.type === 'slow' ? '#00b894' : '#ff6b6b'
    ctx.fillRect(pu.x*cellSize+3, pu.y*cellSize+3, cellSize-6, cellSize-6)
  }

  for(let i=0;i<snake.value.length;i++){
    const s = snake.value[i]
    ctx.fillStyle = i===0 ? '#2a9d8f' : '#264653'
    ctx.fillRect(s.x*cellSize+1, s.y*cellSize+1, cellSize-2, cellSize-2)
  }
}

function checkCollision(head: Point): string | null{
  const now = Date.now()
  for(const p of snake.value){
    if(p.x === head.x && p.y === head.y){
      // allow passing through self if invincible
      if(activeEffects.value['invincible'] && activeEffects.value['invincible'] > now) continue
      return 'self'
    }
  }
  return null
}

function moveSnake(){
  if(isGameOver.value) return
  direction.value = nextDirection.value
  const rawHead = { x: snake.value[0].x + direction.value.x, y: snake.value[0].y + direction.value.y }
  // wrap-around (穿牆)
  const head = { x: (rawHead.x + gridSize) % gridSize, y: (rawHead.y + gridSize) % gridSize }
  const collision = checkCollision(head)
  if(collision){
    if(collision === 'wall') gameOverMessage.value = '你碰到牆壁了。'
    else if(collision === 'self') gameOverMessage.value = '你咬到自己了。'
    gameOver()
    return
  }
  snake.value.unshift(head)
  // check power-up pickup
  const puIndex = powerUps.value.findIndex(p => p.x === head.x && p.y === head.y)
  if(puIndex >= 0){
    const pu = powerUps.value.splice(puIndex,1)[0]
    applyPowerUp(pu.type, pu.duration)
  }

  if(head.x === food.value.x && head.y === food.value.y){
    // combo multiplier handling
    const now = Date.now()
    if(lastEatTime && now - lastEatTime <= comboTimeout){
      comboMultiplier.value += 1
    }else{
      comboMultiplier.value = 1
    }
    lastEatTime = now
    // schedule combo reset
    if(comboTimer) window.clearTimeout(comboTimer)
    comboTimer = window.setTimeout(()=>{ comboMultiplier.value = 1; comboTimer = null }, comboTimeout)

    // score by food type
    if(food.value.type === 'normal') score.value += 1 * comboMultiplier.value
    else if(food.value.type === 'gold') score.value += 3 * comboMultiplier.value
    else if(food.value.type === 'poison'){
      score.value = Math.max(0, score.value - 1)
      // shrink snake if possible
      if(snake.value.length > 3) snake.value.splice(-2)
    }

    placeFood()
  }else{
    snake.value.pop()
  }
  draw()
}

function changeDirectionFromKey(e: KeyboardEvent){
  const k = e.key
  const code = e.code

  // Space: start / restart / resume toggle
  if(code === 'Space'){
    e.preventDefault()
    if(playState.value === 'gameover'){
      reset()
      start()
    }else if(playState.value === 'ready'){
      start()
    }else if(playState.value === 'paused'){
      start()
    }
    return
  }

  // 'p' to toggle pause
  if(k === 'p' || k === 'P'){
    e.preventDefault()
    if(running.value) pause()
    else start()
    return
  }

  let nd: Point | null = null
  if(k === 'ArrowUp' || k === 'w' || k === 'W') nd = {x:0,y:-1}
  else if(k === 'ArrowDown' || k === 's' || k === 'S') nd = {x:0,y:1}
  else if(k === 'ArrowLeft' || k === 'a' || k === 'A') nd = {x:-1,y:0}
  else if(k === 'ArrowRight' || k === 'd' || k === 'D') nd = {x:1,y:0}

  if(nd){
    e.preventDefault()
    if(nd.x === -direction.value.x && nd.y === -direction.value.y) return
    nextDirection.value = nd
  }
}

function start(){
  if(running.value) return
  running.value = true
  playState.value = 'running'
  updateTimer()
}

function pause(){
  if(!running.value) return
  running.value = false
  playState.value = 'paused'
  if(timerId != null) window.clearInterval(timerId)
  timerId = null
}

function reset(){
  pause()
  isGameOver.value = false
  gameOverMessage.value = ''
  initGame()
  playState.value = 'ready'
}


function spawnPowerUp(){
  // small chance to spawn a power-up on empty cell
  const occupied = new Set([...snake.value.map(p=>`${p.x},${p.y}`), `${food.value.x},${food.value.y}`, ...powerUps.value.map(p=>`${p.x},${p.y}`)])
  let x,y
  let attempts = 0
  do{ x = Math.floor(Math.random()*gridSize); y = Math.floor(Math.random()*gridSize); attempts++ } while(occupied.has(`${x},${y}`) && attempts < 50)
  if(attempts >= 50) return
  const r = Math.random()
  let type: PowerUpType = 'speed'
  if(r < 0.33) type = 'speed'
  else if(r < 0.66) type = 'slow'
  else type = 'invincible'
  const duration = 6000
  powerUps.value.push({x,y,type,duration})
}

function applyPowerUp(type: PowerUpType, duration: number){
  const now = Date.now()
  activeEffects.value[type] = now + duration
  // refresh timer
  updateTimer()
}

function gameOver(){
  pause()
  isGameOver.value = true
  playState.value = 'gameover'
  if(!gameOverMessage.value) gameOverMessage.value = '遊戲結束。'
}

onMounted(()=>{
  const canvas = canvasRef.value!
  canvas.width = gridSize * cellSize
  canvas.height = gridSize * cellSize
  ctx = canvas.getContext('2d')
  initGame()
  playState.value = 'ready'
  window.addEventListener('keydown', changeDirectionFromKey)
  // spawn power-ups periodically
  powerUpSpawnTimer = window.setInterval(()=>{
    if(Math.random() < 0.5) spawnPowerUp()
    // clear expired effects
    const now = Date.now()
    for(const k of Object.keys(activeEffects.value)){
      if(activeEffects.value[k] <= now) delete activeEffects.value[k]
    }
  }, 8000)
})

onBeforeUnmount(()=>{
  pause()
  window.removeEventListener('keydown', changeDirectionFromKey)
  if(powerUpSpawnTimer) window.clearInterval(powerUpSpawnTimer)
  if(comboTimer) window.clearTimeout(comboTimer)
  if(foodTimer) window.clearTimeout(foodTimer)
})
</script>

<style scoped>
.container{display:flex;align-items:center;justify-content:center;padding:20px}
.center-box{width:520px;max-width:100%;background:transparent;border-radius:8px;padding:12px}
h1{margin:0 0 12px 0;font-size:20px;text-align:center}
.game-area{display:flex;gap:16px;align-items:center;justify-content:center}
.center-wrapper{position:relative;--game-width:520px;padding:8px}
.center-box{width:var(--game-width);margin:0 auto}
.game-area{display:flex;gap:16px;align-items:center;justify-content:center}
.canvas-wrap{display:flex;align-items:center;justify-content:center}
.legend{position:absolute;left:calc(50% + (var(--game-width)/2) + 16px);top:0;width:180px;padding:8px 12px;border-radius:8px;background:rgba(0,0,0,0.02);text-align:left}
.legend h3{margin:8px 0;font-size:14px}
.legend ul{list-style:none;padding:0;margin:6px 0}
.legend li{display:flex;align-items:center;gap:8px;margin:6px 0;font-size:13px;color:#444}
.swatch{display:inline-block;width:14px;height:14px;border-radius:3px;border:1px solid #ddd}
.swatch.normal{background:#e63946}
.swatch.gold{background:#f4c430}
.swatch.poison{background:#6a1b9a}
.swatch.speed{background:#1e90ff}
.swatch.slow{background:#00b894}
.swatch.invincible{background:#ff6b6b}
canvas{background:#ffffff;border:2px solid #222;box-shadow:0 6px 18px rgba(0,0,0,0.06)}
.info{display:flex;flex-direction:column;gap:10px}
.controls-row{display:flex;gap:10px;justify-content:center;margin-top:12px}
.controls-row button{padding:8px 12px;border-radius:6px;border:1px solid #ddd;background:#fff}
.hint{font-size:12px;color:#666;text-align:center}
.status-row{display:flex;justify-content:space-between;align-items:center;gap:12px;margin:8px 0 12px}
.status-row .state{font-size:18px;font-weight:600;color:#333}
.status-row .score-inline{font-weight:600;color:#2a9d8f}
.score-value{margin-left:6px;color:#2a9d8f}

.overlay{position:fixed;inset:0;display:flex;align-items:center;justify-content:center;background:rgba(0,0,0,0.36)}
.overlay-box{background:#fff;padding:20px;border-radius:10px;min-width:260px;box-shadow:0 12px 30px rgba(0,0,0,0.18);text-align:center}
.overlay-box h2{margin:0 0 8px;font-size:18px}
.overlay-controls{margin-top:12px}
.overlay-controls button{padding:8px 14px;border-radius:6px;border:1px solid #ddd;background:#f6f7fb}
</style>
