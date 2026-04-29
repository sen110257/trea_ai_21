<template>
  <div class="game-container" ref="gameContainer">
    <!-- 动态背景 -->
    <div class="animated-bg">
      <div class="cloud" v-for="i in 5" :key="i" :style="getCloudStyle(i)"></div>
      <div class="glow" v-for="i in 8" :key="'glow'+i" :style="getGlowStyle(i)"></div>
    </div>
    
    <!-- 游戏画布 -->
    <canvas ref="gameCanvas" :width="canvasWidth" :height="canvasHeight"></canvas>
    
    <!-- UI层 -->
    <div class="ui-layer">
      <!-- 分数面板 -->
      <div class="score-panel glass-card" v-if="gameState !== 'idle'">
        <div class="score-item">
          <span class="label">层数</span>
          <span class="value">{{ floor }}</span>
        </div>
        <div class="score-item">
          <span class="label">得分</span>
          <span class="value">{{ score }}</span>
        </div>
        <div class="score-item">
          <span class="label">最高分</span>
          <span class="value">{{ highScore }}</span>
        </div>
      </div>
      
      <!-- 控制按钮 -->
      <div class="control-panel">
        <button 
          class="control-btn glass-btn" 
          @click="startGame"
          v-if="gameState === 'idle'"
        >
          <span class="btn-icon">🎮</span>
          <span>开始游戏</span>
        </button>
        
        <template v-else-if="gameState === 'playing'">
          <button class="control-btn glass-btn" @click="pauseGame">
            <span class="btn-icon">⏸️</span>
            <span>暂停</span>
          </button>
        </template>
        
        <template v-else-if="gameState === 'paused'">
          <button class="control-btn glass-btn" @click="resumeGame">
            <span class="btn-icon">▶️</span>
            <span>继续</span>
          </button>
          <button class="control-btn glass-btn" @click="restartGame">
            <span class="btn-icon">🔄</span>
            <span>重来</span>
          </button>
        </template>
      </div>
      
      <!-- 游戏结束弹窗 -->
      <div class="game-over-modal" v-if="gameState === 'gameover'">
        <div class="modal-content glass-card">
          <h2 class="modal-title">游戏结束</h2>
          <div class="modal-stats">
            <div class="stat-item">
              <span class="stat-label">到达层数</span>
              <span class="stat-value">{{ floor }}</span>
            </div>
            <div class="stat-item">
              <span class="stat-label">本局得分</span>
              <span class="stat-value">{{ score }}</span>
            </div>
            <div class="stat-item highlight">
              <span class="stat-label">最高分</span>
              <span class="stat-value">{{ highScore }}</span>
            </div>
          </div>
          <div class="modal-buttons">
            <button class="control-btn glass-btn primary" @click="restartGame">
              <span class="btn-icon">🔄</span>
              <span>再来一局</span>
            </button>
          </div>
        </div>
      </div>
      
      <!-- 开始界面 -->
      <div class="start-screen" v-if="gameState === 'idle'">
        <h1 class="game-title">是男人就下100层</h1>
        <p class="game-subtitle">经典休闲小游戏 · 挑战你的反应极限</p>
        <div class="highscore-display" v-if="highScore > 0">
          <span>历史最高分: </span>
          <span class="highscore-value">{{ highScore }}</span>
        </div>
      </div>
      
      <!-- 触摸控制区域 -->
      <div class="touch-zone left" 
           @touchstart="handleTouchStart('left')"
           @touchend="handleTouchEnd"
           @mousedown="handleTouchStart('left')"
           @mouseup="handleTouchEnd"
           @mouseleave="handleTouchEnd"
      ></div>
      <div class="touch-zone right"
           @touchstart="handleTouchStart('right')"
           @touchend="handleTouchEnd"
           @mousedown="handleTouchStart('right')"
           @mouseup="handleTouchEnd"
           @mouseleave="handleTouchEnd"
      ></div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'

// 游戏状态
const gameContainer = ref(null)
const gameCanvas = ref(null)
const canvasWidth = ref(0)
const canvasHeight = ref(0)
const gameState = ref('idle') // idle, playing, paused, gameover

// 游戏数据
const score = ref(0)
const floor = ref(0)
const highScore = ref(parseInt(localStorage.getItem('fall100_highscore') || '0'))

// 游戏引擎
let ctx = null
let animationId = null
let gameLoop = null

// 游戏配置
const config = {
  player: {
    width: 40,
    height: 50,
    speed: 6,
    jumpPower: -15,
    gravity: 0.6
  },
  platform: {
    baseWidth: 120,
    minWidth: 60,
    height: 15,
    baseCount: 8,
    verticalSpacing: 80
  },
  difficulty: {
    speedIncrease: 0.02,
    widthDecrease: 0.5,
    spacingIncrease: 0.5
  }
}

// 游戏对象
let player = null
let platforms = []
let cameraY = 0
let keys = { left: false, right: false }
let touchDirection = null

// 背景样式
const getCloudStyle = (i) => {
  const delay = i * 3
  const left = Math.random() * 100
  const size = 60 + Math.random() * 80
  return {
    animationDelay: `${delay}s`,
    left: `${left}%`,
    width: `${size}px`,
    height: `${size * 0.5}px`,
    opacity: 0.1 + Math.random() * 0.2
  }
}

const getGlowStyle = (i) => {
  const delay = i * 2
  const x = Math.random() * 100
  const y = Math.random() * 100
  const size = 100 + Math.random() * 200
  const colors = ['#ff9a9e', '#a8edea', '#fccb90', '#d299c2', '#a1c4fd']
  return {
    animationDelay: `${delay}s`,
    left: `${x}%`,
    top: `${y}%`,
    width: `${size}px`,
    height: `${size}px`,
    background: `radial-gradient(circle, ${colors[i % colors.length]}40 0%, transparent 70%)`,
    filter: 'blur(40px)'
  }
}

// 初始化画布尺寸
const initCanvas = () => {
  if (!gameContainer.value) return
  
  const container = gameContainer.value
  const aspectRatio = 9 / 16
  
  let width = container.clientWidth
  let height = container.clientHeight
  
  // 保持竖屏比例
  if (width > height * aspectRatio) {
    width = height * aspectRatio
  } else {
    height = width / aspectRatio
  }
  
  canvasWidth.value = width
  canvasHeight.value = height
  
  if (gameCanvas.value) {
    ctx = gameCanvas.value.getContext('2d')
  }
}

// 初始化游戏对象
const initGame = () => {
  player = {
    x: canvasWidth.value / 2 - config.player.width / 2,
    y: canvasHeight.value * 0.3,
    vx: 0,
    vy: 0,
    width: config.player.width,
    height: config.player.height,
    onGround: false,
    direction: 1,
    animFrame: 0
  }
  
  platforms = []
  cameraY = 0
  score.value = 0
  floor.value = 0
  
  // 生成初始平台
  generateInitialPlatforms()
}

// 生成初始平台
const generateInitialPlatforms = () => {
  platforms = []
  let y = canvasHeight.value * 0.4
  
  // 玩家起始平台
  platforms.push(createPlatform(
    canvasWidth.value / 2 - config.platform.baseWidth / 2,
    y,
    config.platform.baseWidth,
    'normal'
  ))
  
  // 生成更多平台
  for (let i = 1; i < config.platform.baseCount; i++) {
    y -= config.platform.verticalSpacing
    const width = config.platform.baseWidth
    const x = Math.random() * (canvasWidth.value - width)
    platforms.push(createPlatform(x, y, width, getRandomPlatformType()))
  }
}

// 创建平台
const createPlatform = (x, y, width, type) => {
  return {
    x,
    y,
    width,
    height: config.platform.height,
    type, // normal, disappearing, bouncy, bonus
    visible: true,
    disappearTimer: 0,
    touched: false
  }
}

// 获取随机平台类型
const getRandomPlatformType = () => {
  const rand = Math.random()
  const difficulty = Math.min(floor.value / 100, 1)
  
  // 随难度增加特殊平台概率
  if (rand < 0.05 + difficulty * 0.1) return 'bonus'
  if (rand < 0.15 + difficulty * 0.15) return 'bouncy'
  if (rand < 0.25 + difficulty * 0.2) return 'disappearing'
  return 'normal'
}

// 更新游戏状态
const update = () => {
  if (gameState.value !== 'playing') return
  
  // 更新玩家速度
  if (keys.left || touchDirection === 'left') {
    player.vx = -config.player.speed
    player.direction = -1
  } else if (keys.right || touchDirection === 'right') {
    player.vx = config.player.speed
    player.direction = 1
  } else {
    player.vx *= 0.8
  }
  
  // 应用重力
  player.vy += config.player.gravity
  
  // 更新位置
  player.x += player.vx
  player.y += player.vy
  
  // 边界循环（从一边出去从另一边进来）
  if (player.x + player.width < 0) {
    player.x = canvasWidth.value
  } else if (player.x > canvasWidth.value) {
    player.x = -player.width
  }
  
  // 检测平台碰撞
  player.onGround = false
  platforms.forEach(platform => {
    if (!platform.visible) return
    
    // 简单的碰撞检测（只检测从上方落下）
    if (
      player.vy > 0 &&
      player.x + player.width > platform.x &&
      player.x < platform.x + platform.width &&
      player.y + player.height > platform.y &&
      player.y + player.height < platform.y + platform.height + player.vy
    ) {
      handlePlatformCollision(platform)
    }
  })
  
  // 更新消失平台
  platforms.forEach(platform => {
    if (platform.type === 'disappearing' && platform.disappearTimer > 0) {
      platform.disappearTimer--
      if (platform.disappearTimer <= 0) {
        platform.visible = false
      }
    }
  })
  
  // 相机跟随
  const targetCameraY = player.y - canvasHeight.value * 0.4
  if (targetCameraY < cameraY) {
    cameraY = targetCameraY
  }
  
  // 更新层数
  const currentFloor = Math.floor((-cameraY) / config.platform.verticalSpacing) + 1
  if (currentFloor > floor.value) {
    floor.value = currentFloor
    addScore(10 + Math.floor(Math.random() * 20))
  }
  
  // 生成新平台
  generateNewPlatforms()
  
  // 移除屏幕外的平台
  removeOffscreenPlatforms()
  
  // 检测游戏结束（玩家掉出屏幕下方）
  if (player.y - cameraY > canvasHeight.value + 100) {
    gameOver()
  }
  
  // 动画帧
  player.animFrame = (player.animFrame + 0.2) % 2
}

// 处理平台碰撞
const handlePlatformCollision = (platform) => {
  player.y = platform.y - player.height
  player.onGround = true
  
  switch (platform.type) {
    case 'normal':
      player.vy = 0
      break
      
    case 'bouncy':
      player.vy = config.player.jumpPower * 1.5
      addScore(5)
      break
      
    case 'disappearing':
      player.vy = 0
      if (!platform.touched) {
        platform.touched = true
        platform.disappearTimer = 30 // 0.5秒后消失
      }
      break
      
    case 'bonus':
      player.vy = 0
      if (!platform.touched) {
        platform.touched = true
        addScore(50 + Math.floor(Math.random() * 100))
      }
      break
  }
}

// 加分
const addScore = (points) => {
  score.value += points
  if (score.value > highScore.value) {
    highScore.value = score.value
    localStorage.setItem('fall100_highscore', highScore.value.toString())
  }
}

// 生成新平台
const generateNewPlatforms = () => {
  const difficulty = Math.min(floor.value / 100, 1)
  
  // 计算当前平台的最高Y值
  let highestY = Infinity
  platforms.forEach(p => {
    if (p.y < highestY) highestY = p.y
  })
  
  // 在屏幕上方生成新平台
  const spacing = config.platform.verticalSpacing + difficulty * config.difficulty.spacingIncrease
  while (highestY > cameraY - spacing * 2) {
    highestY -= spacing
    const width = Math.max(
      config.platform.minWidth,
      config.platform.baseWidth - difficulty * config.difficulty.widthDecrease
    )
    const x = Math.random() * (canvasWidth.value - width)
    platforms.push(createPlatform(x, highestY, width, getRandomPlatformType()))
  }
}

// 移除屏幕外的平台
const removeOffscreenPlatforms = () => {
  platforms = platforms.filter(p => p.y - cameraY < canvasHeight.value + 100)
}

// 绘制游戏
const render = () => {
  if (!ctx) return
  
  // 清空画布
  ctx.clearRect(0, 0, canvasWidth.value, canvasHeight.value)
  
  // 绘制平台
  platforms.forEach(platform => {
    if (!platform.visible) return
    
    const screenY = platform.y - cameraY
    if (screenY < -50 || screenY > canvasHeight.value + 50) return
    
    drawPlatform(platform, screenY)
  })
  
  // 绘制玩家
  if (player) {
    drawPlayer()
  }
}

// 绘制平台
const drawPlatform = (platform, screenY) => {
  ctx.save()
  
  // 平台颜色
  let gradient, shadowColor, glowColor
  
  switch (platform.type) {
    case 'normal':
      gradient = ctx.createLinearGradient(platform.x, screenY, platform.x, screenY + platform.height)
      gradient.addColorStop(0, '#a8edea')
      gradient.addColorStop(1, '#fed6e3')
      shadowColor = 'rgba(168, 237, 234, 0.3)'
      glowColor = 'rgba(168, 237, 234, 0.5)'
      break
      
    case 'bouncy':
      gradient = ctx.createLinearGradient(platform.x, screenY, platform.x, screenY + platform.height)
      gradient.addColorStop(0, '#fccb90')
      gradient.addColorStop(1, '#d57eeb')
      shadowColor = 'rgba(252, 203, 144, 0.3)'
      glowColor = 'rgba(252, 203, 144, 0.5)'
      break
      
    case 'disappearing':
      const opacity = platform.disappearTimer > 0 ? platform.disappearTimer / 30 : 1
      gradient = ctx.createLinearGradient(platform.x, screenY, platform.x, screenY + platform.height)
      gradient.addColorStop(0, `rgba(255, 154, 158, ${opacity})`)
      gradient.addColorStop(1, `rgba(250, 208, 196, ${opacity})`)
      shadowColor = `rgba(255, 154, 158, ${opacity * 0.3})`
      glowColor = `rgba(255, 154, 158, ${opacity * 0.5})`
      break
      
    case 'bonus':
      gradient = ctx.createLinearGradient(platform.x, screenY, platform.x, screenY + platform.height)
      gradient.addColorStop(0, '#a1c4fd')
      gradient.addColorStop(1, '#c2e9fb')
      shadowColor = 'rgba(161, 196, 253, 0.3)'
      glowColor = 'rgba(161, 196, 253, 0.5)'
      
      // 星星标记
      ctx.fillStyle = '#fff'
      ctx.font = '12px Arial'
      ctx.textAlign = 'center'
      ctx.fillText('⭐', platform.x + platform.width / 2, screenY - 5)
      break
  }
  
  // 发光效果
  ctx.shadowColor = glowColor
  ctx.shadowBlur = 10
  
  // 圆角矩形
  const radius = 8
  ctx.beginPath()
  ctx.moveTo(platform.x + radius, screenY)
  ctx.lineTo(platform.x + platform.width - radius, screenY)
  ctx.quadraticCurveTo(platform.x + platform.width, screenY, platform.x + platform.width, screenY + radius)
  ctx.lineTo(platform.x + platform.width, screenY + platform.height - radius)
  ctx.quadraticCurveTo(platform.x + platform.width, screenY + platform.height, platform.x + platform.width - radius, screenY + platform.height)
  ctx.lineTo(platform.x + radius, screenY + platform.height)
  ctx.quadraticCurveTo(platform.x, screenY + platform.height, platform.x, screenY + platform.height - radius)
  ctx.lineTo(platform.x, screenY + radius)
  ctx.quadraticCurveTo(platform.x, screenY, platform.x + radius, screenY)
  ctx.closePath()
  
  ctx.fillStyle = gradient
  ctx.fill()
  
  // 磨砂玻璃效果边框
  ctx.shadowBlur = 0
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.5)'
  ctx.lineWidth = 1
  ctx.stroke()
  
  // 高光
  ctx.beginPath()
  ctx.moveTo(platform.x + radius, screenY + 2)
  ctx.lineTo(platform.x + platform.width - radius, screenY + 2)
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.8)'
  ctx.lineWidth = 1.5
  ctx.stroke()
  
  ctx.restore()
}

// 绘制玩家
const drawPlayer = () => {
  if (!player) return
  
  const screenY = player.y - cameraY
  
  ctx.save()
  
  // 绘制Q版卡通人物
  const centerX = player.x + player.width / 2
  const centerY = screenY + player.height / 2
  
  // 身体渐变
  const bodyGradient = ctx.createRadialGradient(centerX, centerY, 0, centerX, centerY, player.width)
  bodyGradient.addColorStop(0, '#ff9a9e')
  bodyGradient.addColorStop(1, '#fecfef')
  
  // 身体阴影
  ctx.shadowColor = 'rgba(255, 154, 158, 0.5)'
  ctx.shadowBlur = 15
  ctx.shadowOffsetY = 5
  
  // 身体（椭圆形）
  ctx.beginPath()
  ctx.ellipse(centerX, centerY + 5, player.width / 2 - 5, player.height / 2 - 8, 0, 0, Math.PI * 2)
  ctx.fillStyle = bodyGradient
  ctx.fill()
  
  ctx.shadowBlur = 0
  
  // 头部
  const headGradient = ctx.createRadialGradient(centerX, screenY + 12, 0, centerX, screenY + 12, 20)
  headGradient.addColorStop(0, '#ffecd2')
  headGradient.addColorStop(1, '#fcb69f')
  
  ctx.beginPath()
  ctx.arc(centerX, screenY + 15, 15, 0, Math.PI * 2)
  ctx.fillStyle = headGradient
  ctx.fill()
  
  // 眼睛
  const eyeOffset = player.direction * 3
  ctx.fillStyle = '#333'
  ctx.beginPath()
  ctx.arc(centerX - 5 + eyeOffset, screenY + 13, 3, 0, Math.PI * 2)
  ctx.fill()
  ctx.beginPath()
  ctx.arc(centerX + 5 + eyeOffset, screenY + 13, 3, 0, Math.PI * 2)
  ctx.fill()
  
  // 眼睛高光
  ctx.fillStyle = '#fff'
  ctx.beginPath()
  ctx.arc(centerX - 4 + eyeOffset, screenY + 12, 1.5, 0, Math.PI * 2)
  ctx.fill()
  ctx.beginPath()
  ctx.arc(centerX + 6 + eyeOffset, screenY + 12, 1.5, 0, Math.PI * 2)
  ctx.fill()
  
  // 腮红
  ctx.fillStyle = 'rgba(255, 182, 193, 0.6)'
  ctx.beginPath()
  ctx.arc(centerX - 10, screenY + 17, 4, 0, Math.PI * 2)
  ctx.fill()
  ctx.beginPath()
  ctx.arc(centerX + 10, screenY + 17, 4, 0, Math.PI * 2)
  ctx.fill()
  
  // 嘴巴
  ctx.strokeStyle = '#ff6b6b'
  ctx.lineWidth = 1.5
  ctx.beginPath()
  ctx.arc(centerX, screenY + 18, 4, 0.1 * Math.PI, 0.9 * Math.PI)
  ctx.stroke()
  
  // 帽子
  ctx.fillStyle = '#667eea'
  ctx.beginPath()
  ctx.moveTo(centerX - 12, screenY + 2)
  ctx.lineTo(centerX, screenY - 15)
  ctx.lineTo(centerX + 12, screenY + 2)
  ctx.closePath()
  ctx.fill()
  
  // 帽子高光
  ctx.fillStyle = 'rgba(255, 255, 255, 0.3)'
  ctx.beginPath()
  ctx.moveTo(centerX - 8, screenY + 2)
  ctx.lineTo(centerX - 2, screenY - 8)
  ctx.lineTo(centerX + 2, screenY - 3)
  ctx.lineTo(centerX - 4, screenY + 2)
  ctx.closePath()
  ctx.fill()
  
  // 腿部动画
  const legOffset = Math.sin(player.animFrame * Math.PI) * 3
  ctx.fillStyle = '#fecfef'
  ctx.beginPath()
  ctx.ellipse(centerX - 8, screenY + player.height - 8 + legOffset, 5, 8, 0, 0, Math.PI * 2)
  ctx.fill()
  ctx.beginPath()
  ctx.ellipse(centerX + 8, screenY + player.height - 8 - legOffset, 5, 8, 0, 0, Math.PI * 2)
  ctx.fill()
  
  ctx.restore()
}

// 游戏循环
const gameLoopFn = () => {
  update()
  render()
  animationId = requestAnimationFrame(gameLoopFn)
}

// 游戏控制
const startGame = () => {
  gameState.value = 'playing'
  initGame()
  if (!animationId) {
    gameLoopFn()
  }
}

const pauseGame = () => {
  gameState.value = 'paused'
}

const resumeGame = () => {
  gameState.value = 'playing'
}

const restartGame = () => {
  initGame()
  gameState.value = 'playing'
}

const gameOver = () => {
  gameState.value = 'gameover'
  // 保存最高分
  if (score.value > parseInt(localStorage.getItem('fall100_highscore') || '0')) {
    localStorage.setItem('fall100_highscore', score.value.toString())
  }
}

// 键盘控制
const handleKeyDown = (e) => {
  if (e.key === 'ArrowLeft' || e.key === 'a') {
    keys.left = true
  }
  if (e.key === 'ArrowRight' || e.key === 'd') {
    keys.right = true
  }
  if (e.key === ' ' && gameState.value === 'playing') {
    pauseGame()
  } else if (e.key === ' ' && gameState.value === 'paused') {
    resumeGame()
  }
}

const handleKeyUp = (e) => {
  if (e.key === 'ArrowLeft' || e.key === 'a') {
    keys.left = false
  }
  if (e.key === 'ArrowRight' || e.key === 'd') {
    keys.right = false
  }
}

// 触摸控制
const handleTouchStart = (direction) => {
  if (gameState.value !== 'playing') return
  touchDirection = direction
}

const handleTouchEnd = () => {
  touchDirection = null
}

// 窗口大小变化
const handleResize = () => {
  initCanvas()
}

// 生命周期
onMounted(() => {
  initCanvas()
  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('keyup', handleKeyUp)
  window.addEventListener('resize', handleResize)
  
  // 开始渲染循环（即使游戏没开始也渲染背景）
  gameLoopFn()
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown)
  window.removeEventListener('keyup', handleKeyUp)
  window.removeEventListener('resize', handleResize)
  
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
})
</script>

<style scoped>
.game-container {
  position: relative;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  overflow: hidden;
}

/* 动态背景 */
.animated-bg {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  overflow: hidden;
}

.cloud {
  position: absolute;
  background: linear-gradient(180deg, rgba(255,255,255,0.3) 0%, rgba(255,255,255,0.1) 100%);
  border-radius: 50%;
  animation: floatCloud 20s ease-in-out infinite;
}

@keyframes floatCloud {
  0%, 100% { transform: translateY(0) translateX(0); }
  25% { transform: translateY(-20px) translateX(10px); }
  50% { transform: translateY(-10px) translateX(-10px); }
  75% { transform: translateY(-30px) translateX(5px); }
}

.glow {
  position: absolute;
  border-radius: 50%;
  animation: pulseGlow 8s ease-in-out infinite;
}

@keyframes pulseGlow {
  0%, 100% { transform: scale(1); opacity: 0.5; }
  50% { transform: scale(1.2); opacity: 0.8; }
}

/* 游戏画布 */
canvas {
  position: relative;
  z-index: 1;
  max-width: 100%;
  max-height: 100%;
}

/* UI层 */
.ui-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 10;
  pointer-events: none;
}

.ui-layer > * {
  pointer-events: auto;
}

/* 开始界面 */
.start-screen {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 15;
}

.game-title {
  font-size: 2.5rem;
  font-weight: 800;
  color: #fff;
  text-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
  margin-bottom: 1rem;
  background: linear-gradient(135deg, #fff 0%, #f0f0f0 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.game-subtitle {
  font-size: 1rem;
  color: rgba(255, 255, 255, 0.8);
  margin-bottom: 2rem;
}

.highscore-display {
  font-size: 1.1rem;
  color: rgba(255, 255, 255, 0.9);
  margin-bottom: 1rem;
}

.highscore-value {
  font-weight: 700;
  color: #ffd700;
  font-size: 1.3rem;
}

/* 分数面板 */
.score-panel {
  position: absolute;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 2rem;
  padding: 1rem 2rem;
  z-index: 20;
}

.score-item {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.score-item .label {
  font-size: 0.85rem;
  color: rgba(255, 255, 255, 0.8);
  margin-bottom: 0.3rem;
}

.score-item .value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #fff;
  text-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
}

/* 控制面板 */
.control-panel {
  position: absolute;
  bottom: 30px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 1rem;
  z-index: 20;
}

/* 玻璃卡片效果 */
.glass-card {
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
}

.glass-btn {
  position: relative;
  padding: 0.8rem 1.5rem;
  background: rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 50px;
  color: #fff;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  overflow: hidden;
}

.glass-btn::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255,255,255,0.3), transparent);
  transition: left 0.5s ease;
}

.glass-btn:hover::before {
  left: 100%;
}

.glass-btn:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: translateY(-3px);
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
}

.glass-btn:active {
  transform: translateY(-1px);
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.15);
}

.glass-btn.primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border: none;
}

.glass-btn.primary:hover {
  background: linear-gradient(135deg, #764ba2 0%, #667eea 100%);
}

.btn-icon {
  font-size: 1.2rem;
}

/* 游戏结束弹窗 */
.game-over-modal {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  z-index: 30;
}

.modal-content {
  padding: 2rem 3rem;
  text-align: center;
  animation: modalPopIn 0.4s cubic-bezier(0.68, -0.55, 0.265, 1.55);
}

@keyframes modalPopIn {
  0% { transform: scale(0.5); opacity: 0; }
  100% { transform: scale(1); opacity: 1; }
}

.modal-title {
  font-size: 2rem;
  font-weight: 800;
  color: #fff;
  margin-bottom: 1.5rem;
  text-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
}

.modal-stats {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  margin-bottom: 2rem;
}

.stat-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.8rem 1rem;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 12px;
}

.stat-item.highlight {
  background: linear-gradient(135deg, rgba(255, 215, 0, 0.2) 0%, rgba(255, 165, 0, 0.2) 100%);
  border: 1px solid rgba(255, 215, 0, 0.3);
}

.stat-label {
  font-size: 1rem;
  color: rgba(255, 255, 255, 0.8);
}

.stat-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #fff;
}

.modal-buttons {
  display: flex;
  justify-content: center;
}

/* 触摸控制区域 */
.touch-zone {
  position: absolute;
  top: 0;
  width: 50%;
  height: 100%;
  z-index: 5;
  transition: background 0.1s ease;
}

.touch-zone.left {
  left: 0;
}

.touch-zone.right {
  right: 0;
}

.touch-zone:active {
  background: rgba(255, 255, 255, 0.05);
}

/* 响应式适配 */
@media (max-width: 480px) {
  .game-title {
    font-size: 1.8rem;
  }
  
  .game-subtitle {
    font-size: 0.85rem;
  }
  
  .score-panel {
    padding: 0.8rem 1.5rem;
    gap: 1.5rem;
  }
  
  .score-item .value {
    font-size: 1.2rem;
  }
  
  .glass-btn {
    padding: 0.7rem 1.2rem;
    font-size: 0.9rem;
  }
  
  .modal-content {
    padding: 1.5rem 2rem;
    margin: 0 1rem;
  }
  
  .modal-title {
    font-size: 1.5rem;
  }
}

@media (orientation: landscape) {
  .score-panel {
    top: 10px;
  }
  
  .control-panel {
    bottom: 15px;
  }
}
</style>
