<template>
  <div class="game-wrapper" @mousemove="onMouseMove" @click="shoot">
    <!-- Custom crosshair cursor -->
    <div class="crosshair" :style="{ left: cursor.x + 'px', top: cursor.y + 'px' }" />

    <!-- Sky background -->
    <div class="sky">
      <!-- Clouds -->
      <div
        v-for="c in clouds"
        :key="c.id"
        class="cloud"
        :style="{ left: c.x + '%', top: c.y + '%', opacity: c.opacity }"
      />

      <!-- Sun -->
      <div class="sun" />

      <!-- Trees at bottom -->
      <div class="ground-area">
        <div v-for="t in trees" :key="t.id" class="tree" :style="{ left: t.x + '%' }">
          <div class="tree-top" :style="{ width: t.size + 'px', height: t.size + 'px' }" />
          <div class="tree-trunk" />
        </div>
      </div>
    </div>

    <!-- HUD -->
    <div class="hud">
      <div class="hud-left">
        <div class="stat-box">
          <span class="stat-label">🎯 Bắn</span>
          <span class="stat-value">{{ shots }}</span>
        </div>
        <div class="stat-box">
          <span class="stat-label">💨 Trượt</span>
          <span class="stat-value miss-count">{{ misses }}</span>
        </div>
        <div class="stat-box">
          <span class="stat-label">🏆 Trúng</span>
          <span class="stat-value hit-count">{{ hits }}</span>
        </div>
      </div>
      <div class="hud-title">🔫 BIRD HUNTER</div>
      <div class="hud-right">
        <div class="ammo-box">
          <span v-for="i in 10" :key="i" class="bullet-icon" :class="{ used: i > ammo }">🔴</span>
        </div>
        <button class="reload-btn" @click.stop="reload">↺ NẠP ĐẠN</button>
        <router-link to="/" class="home-btn" @click.stop>🏠 Trang chủ</router-link>
      </div>
    </div>

    <!-- Bird -->
    <div
      class="bird"
      :class="[`bird-dir-${birdDir}`, { 'bird-dodge': isDodging, 'bird-hurt': isHurt }]"
      :style="{ left: bird.x + 'px', top: bird.y + 'px' }"
    >
      <div class="bird-body">
        <!-- Body -->
        <div class="bird-svg">
          <svg viewBox="0 0 80 60" xmlns="http://www.w3.org/2000/svg">
            <!-- Tail -->
            <ellipse cx="12" cy="35" rx="12" ry="6" fill="#e67e22" transform="rotate(-15 12 35)" />
            <!-- Body -->
            <ellipse cx="42" cy="32" rx="22" ry="16" fill="#f39c12" />
            <!-- Wing -->
            <ellipse
              cx="42"
              cy="26"
              rx="18"
              ry="10"
              fill="#e67e22"
              :style="{ transform: `rotate(${wingAnim}deg)`, transformOrigin: '42px 26px' }"
            />
            <!-- Belly -->
            <ellipse cx="46" cy="36" rx="12" ry="9" fill="#ffeaa7" />
            <!-- Head -->
            <circle cx="62" cy="22" r="14" fill="#f39c12" />
            <!-- Eye white -->
            <circle cx="66" cy="19" r="5" fill="white" />
            <!-- Eye pupil -->
            <circle :cx="eyeX" :cy="eyeY" r="3" fill="#2d3436" />
            <!-- Eye shine -->
            <circle :cx="eyeX + 1" :cy="eyeY - 1" r="1" fill="white" />
            <!-- Beak -->
            <polygon points="74,22 82,19 74,26" fill="#e17055" />
            <!-- Sunglasses (when trolling) -->
            <template v-if="isTrolling">
              <rect x="58" y="14" width="14" height="8" rx="3" fill="#2d3436" opacity="0.9" />
              <line x1="72" y1="18" x2="76" y2="17" stroke="#2d3436" stroke-width="1.5" />
            </template>
          </svg>
        </div>
      </div>
    </div>

    <!-- Troll speech bubble - ĐỨNG YÊN, không bay theo chim -->
    <transition name="bubble">
      <div
        v-if="trollMessage"
        class="speech-bubble-fixed"
        :style="{ left: bubblePos.x + 'px', top: bubblePos.y + 'px' }"
      >
        <span>{{ trollMessage }}</span>
      </div>
    </transition>

    <!-- Bullets flying -->
    <div
      v-for="b in bullets"
      :key="b.id"
      class="bullet"
      :style="{ left: b.x + 'px', top: b.y + 'px', transform: `rotate(${b.angle}deg)` }"
    >
      💛
    </div>

    <!-- Miss flash effect -->
    <transition name="miss-flash">
      <div v-if="showMissFlash" class="miss-flash" />
    </transition>

    <!-- Hit effect (rare) -->
    <transition name="hit-effect">
      <div v-if="showHit" class="hit-effect">
        <div class="hit-text">💥 TRÚNG RỒI!!!</div>
        <div class="feathers">🪶🪶🪶🪶🪶</div>
      </div>
    </transition>

    <!-- Instruction -->
    <div class="instruction" v-if="shots === 0">
      <p>🖱️ Click để bắn • Con Chim <strong>NON TRÊN TRỜI CAO</strong> 😈</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted } from 'vue'

// --- Types ---
interface Vec2 {
  x: number
  y: number
}
interface Bullet {
  id: number
  x: number
  y: number
  angle: number
  vx: number
  vy: number
}
interface Cloud {
  id: number
  x: number
  y: number
  opacity: number
}
interface Tree {
  id: number
  x: number
  size: number
}

// --- Troll messages ---
const trollMessages = [
  // --- Cà khịa cơ bản ---
  '😂 Tệ vậy bạn ơi!',
  '🤣 Nhắm mắt bắn à?',
  '😎 Quá chậm rồi bro!',
  '🐦 Chim mà cũng bắn không trúng!',
  '💀 Mua súng ở đâu vậy?',
  '🤡 Cần kính không bạn?',
  '😜 Ờ thì bắn tiếp đi!',
  '🦅 Bay đây này, đuổi theo nè!',
  '😏 Haha trật rồi!',
  '🎯 Ngắm kỹ hơn đi bro!',
  '🔫 Súng hay đồ chơi vậy?',
  '😴 Tôi ngủ gật vẫn tránh được!',
  '🍌 Bắn chuối còn dễ hơn!',
  '💸 Tiền đạn đâu tiêu hết rồi!',
  '🤦 Trời ơi...',
  // --- Cà khịa nâng cao ---
  '🪃 Đạn trật rồi bay về nhà chủ!',
  '📞 Gọi thợ săn thật đến đây đi!',
  '🎪 Bắn súng hay biểu diễn xiếc vậy?',
  '🐌 Mày chậm hơn ốc sên luôn!',
  '🧠 Mượn não ai đó đi rồi bắn lại!',
  '🤸 Tránh dễ như ăn kẹo á!',
  '🎭 Diễn hay lắm, nhưng trật rồi!',
  '🏃 Tao chạy mà mày vẫn bắn trật!',
  '📸 Chụp ảnh tao đi, bắn không vô đâu!',
  '🎵 Bắn trật đều nhịp ghê, như nhạc vậy!',
  '🌪️ Gió thổi đạn bay đi rồi bro ơi!',
  '🍕 Về ăn pizza đi, bắn kiểu này đói lắm!',
  // --- Gợi ý "thân thiện" ---
  '💡 Tip: Ngắm vào chỗ tao KHÔNG đứng nhé!',
  '📖 Đọc hướng dẫn sử dụng súng chưa?',
  '🏫 Có lớp học bắn súng nè, ghi danh đi!',
  '🎮 Chơi game bắn súng trước đi rồi quay lại!',
  '🔧 Súng hỏng rồi à? Hay tay hỏng?',
  '📐 Học hình học lại đi, đường thẳng là gì?',
  // --- Triết học chim ---
  '🧘 Tao đã đạt giác ngộ, đạn không thể chạm!',
  '🌈 Tao bay trên cả không gian và thời gian!',
  '🦜 Tổ tiên tao từng né đại bác, mày là gì?',
  '⚡ Tao là chim, tao là gió, tao là tự do!',
  '🌊 Như nước chảy, tao không thể bị bắt giữ!',
  // --- Khuyến khích (giả tạo) ---
  '👏 Cố lên! Còn khoảng 999 phát nữa may ra!',
  '🌟 Bắn thêm đi, tao cần tập thể dục!',
  '💪 Mày đang giúp tao warm-up đó, cảm ơn!',
  '🎁 Thưởng cho mày 1 cái: tiếp tục trật nhé!',
  '🤝 Hợp đồng: mày bắn, tao né, win-win!',
  // --- Ngẫu hứng ---
  '🎬 Cut! Cảnh này bắn trật quá nhiều take rồi!',
  '📺 Nếu đây là phim, đạo diễn đã la rồi!',
  '🎲 Xác suất trúng tao: 0.1%, mày đang ở 0%!',
  '🤖 AI tao né đạn giỏi hơn AI mày ngắm!',
  '👻 Tao đã biến thành ma, đạn xuyên qua luôn!',
  '🦸 Tao là siêu chim, bất khả xâm phạm!',
  '🌙 Đợi trăng rằm đi rồi bắn lại, may hơn!',
  '🍀 Mày cần 4 lá clover may mắn gấp!',
  '🎰 Slot machine còn dễ trúng hơn bắn tao!',
  '🚀 Đạn mày bay lên tận vũ trụ rồi kìa!',
]

// --- State ---
const cursor = reactive<Vec2>({ x: -100, y: -100 })
const bird = reactive<Vec2>({ x: 300, y: 200 })
const birdVel = reactive<Vec2>({ x: 2.5, y: 1.5 })
const birdDir = ref<'left' | 'right'>('right')
const isDodging = ref(false)
const isHurt = ref(false)
const isTrolling = ref(false)
const trollMessage = ref('')
const bubblePos = reactive<Vec2>({ x: 0, y: 0 })
const wingAnim = ref(0)
const eyeX = ref(66)
const eyeY = ref(19)

const shots = ref(0)
const misses = ref(0)
const hits = ref(0)
const ammo = ref(10)
const bullets = ref<Bullet[]>([])
const showMissFlash = ref(false)
const showHit = ref(false)
let bulletId = 0

const clouds: Cloud[] = Array.from({ length: 5 }, (_, i) => ({
  id: i,
  x: i * 20 + 5,
  y: Math.random() * 30 + 5,
  opacity: 0.6 + Math.random() * 0.4,
}))
const trees: Tree[] = Array.from({ length: 8 }, (_, i) => ({
  id: i,
  x: i * 13 + 2,
  size: 40 + Math.random() * 30,
}))

// --- Game loop ---
let animFrame = 0
let wingDir = 1
let dodgeTimer = 0
let trollTimer = 0

function gameLoop() {
  // Wing flap
  wingAnim.value += wingDir * 3
  if (wingAnim.value > 20 || wingAnim.value < -20) wingDir *= -1

  // Bird movement
  const W = window.innerWidth
  const H = window.innerHeight

  if (isDodging.value && dodgeTimer > 0) {
    dodgeTimer--
    if (dodgeTimer <= 0) isDodging.value = false
  }

  bird.x += birdVel.x
  bird.y += birdVel.y

  // Bounce off walls
  if (bird.x < 50 || bird.x > W - 120) {
    birdVel.x *= -1
    birdDir.value = birdVel.x > 0 ? 'right' : 'left'
  }
  if (bird.y < 60 || bird.y > H - 180) {
    birdVel.y *= -1
  }

  // Bird eyes follow cursor lazily
  const dx = cursor.x - (bird.x + 60)
  const dy = cursor.y - (bird.y + 20)
  const dist = Math.sqrt(dx * dx + dy * dy)
  if (dist > 0) {
    eyeX.value = 66 + Math.min(2, (dx / dist) * 2)
    eyeY.value = 19 + Math.min(2, (dy / dist) * 2)
  }

  // Move bullets
  bullets.value = bullets.value.filter((b) => {
    b.x += b.vx
    b.y += b.vy
    return b.x > -50 && b.x < W + 50 && b.y > -50 && b.y < H + 50
  })

  // Troll timer
  if (trollTimer > 0) {
    trollTimer--
    if (trollTimer <= 0) {
      trollMessage.value = ''
      isTrolling.value = false
    }
  }

  animFrame = requestAnimationFrame(gameLoop)
}

// --- Dodge logic ---
function dodgeBullet(bx: number, by: number) {
  // Move bird away from bullet direction
  const dx = bird.x - bx
  const dy = bird.y - by
  const dist = Math.sqrt(dx * dx + dy * dy)

  if (dist < 200) {
    // Evade perpendicular
    birdVel.x = (dx / dist) * (4 + Math.random() * 3)
    birdVel.y = (dy / dist) * (4 + Math.random() * 3) - 2
    birdDir.value = birdVel.x > 0 ? 'right' : 'left'
    isDodging.value = true
    dodgeTimer = 40
  }
}

// --- Shoot ---
function shoot() {
  if (ammo.value <= 0) return

  shots.value++
  ammo.value--

  // Bullet from cursor toward bird (intentionally slightly off)
  const targetX = bird.x + 40 + (Math.random() - 0.5) * 20
  const targetY = bird.y + 30 + (Math.random() - 0.5) * 20
  const dx = targetX - cursor.x
  const dy = targetY - cursor.y
  const dist = Math.sqrt(dx * dx + dy * dy)
  const speed = 18

  const bullet: Bullet = {
    id: bulletId++,
    x: cursor.x,
    y: cursor.y,
    angle: (Math.atan2(dy, dx) * 180) / Math.PI,
    vx: (dx / dist) * speed,
    vy: (dy / dist) * speed,
  }
  bullets.value.push(bullet)

  // Bird ALWAYS dodges when shot near
  dodgeBullet(cursor.x, cursor.y)

  // 1/1000 chance to hit
  const lucky = Math.random() < 0.001

  setTimeout(() => {
    if (lucky) {
      hits.value++
      showHit.value = true
      isHurt.value = true
      setTimeout(() => {
        showHit.value = false
        isHurt.value = false
      }, 2500)
    } else {
      misses.value++
      showMissFlash.value = true
      setTimeout(() => {
        showMissFlash.value = false
      }, 150)

      // Troll! Lưu vị trí HIỆN TẠI của chim để bubble đứng yên
      isTrolling.value = true
      trollMessage.value = trollMessages[Math.floor(Math.random() * trollMessages.length)]
      // Gắn bubble phía trên đầu chim tại vị trí lúc bắn, rồi không di chuyển nữa
      bubblePos.x = bird.x - 60
      bubblePos.y = bird.y - 70
      trollTimer = 300 + Math.floor(Math.random() * 120)
    }
  }, 300)
}

function reload() {
  ammo.value = 10
}

function onMouseMove(e: MouseEvent) {
  cursor.x = e.clientX
  cursor.y = e.clientY
}

onMounted(() => {
  gameLoop()
})
onUnmounted(() => {
  cancelAnimationFrame(animFrame)
})
</script>

<style scoped>
/* Hide default cursor */
.game-wrapper {
  position: fixed;
  inset: 0;
  overflow: hidden;
  cursor: none;
  user-select: none;
  font-family: 'Segoe UI', sans-serif;
}

/* Sky gradient */
.sky {
  width: 100%;
  height: 100%;
  background: linear-gradient(
    180deg,
    #87ceeb 0%,
    #b8e4f9 55%,
    #c8f0a0 55%,
    #7ec850 70%,
    #5a9e3a 100%
  );
  position: relative;
}

/* Sun */
.sun {
  position: absolute;
  top: 6%;
  right: 10%;
  width: 70px;
  height: 70px;
  background: radial-gradient(circle, #fff176 30%, #ffd54f 70%, transparent 100%);
  border-radius: 50%;
  box-shadow: 0 0 40px 20px rgba(255, 220, 50, 0.4);
  animation: sunPulse 3s ease-in-out infinite;
}

@keyframes sunPulse {
  0%,
  100% {
    box-shadow: 0 0 40px 20px rgba(255, 220, 50, 0.4);
  }

  50% {
    box-shadow: 0 0 60px 30px rgba(255, 220, 50, 0.6);
  }
}

/* Clouds */
.cloud {
  position: absolute;
  width: 100px;
  height: 40px;
  background: white;
  border-radius: 50px;
  animation: cloudDrift 30s linear infinite;
}

.cloud::before,
.cloud::after {
  content: '';
  position: absolute;
  background: white;
  border-radius: 50%;
}

.cloud::before {
  width: 50px;
  height: 50px;
  top: -20px;
  left: 15px;
}

.cloud::after {
  width: 35px;
  height: 35px;
  top: -12px;
  left: 50px;
}

@keyframes cloudDrift {
  from {
    transform: translateX(0);
  }

  to {
    transform: translateX(120vw);
  }
}

/* Ground / Trees */
.ground-area {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 120px;
}

.tree {
  position: absolute;
  bottom: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.tree-top {
  background: radial-gradient(circle, #4caf50, #2e7d32);
  border-radius: 50% 50% 40% 40%;
}

.tree-trunk {
  width: 12px;
  height: 30px;
  background: linear-gradient(#795548, #5d4037);
  border-radius: 3px;
}

/* Crosshair */
.crosshair {
  position: fixed;
  width: 40px;
  height: 40px;
  pointer-events: none;
  z-index: 9999;
  transform: translate(-50%, -50%);
}

.crosshair::before,
.crosshair::after {
  content: '';
  position: absolute;
  background: rgba(255, 50, 50, 0.9);
}

.crosshair::before {
  width: 2px;
  height: 40px;
  left: 19px;
  top: 0;
  box-shadow: 0 0 4px red;
}

.crosshair::after {
  width: 40px;
  height: 2px;
  top: 19px;
  left: 0;
  box-shadow: 0 0 4px red;
}

/* HUD */
.hud {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: 64px;
  background: rgba(0, 0, 0, 0.65);
  backdrop-filter: blur(8px);
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 20px;
  z-index: 100;
  border-bottom: 2px solid rgba(255, 200, 0, 0.3);
}

.hud-title {
  font-size: 1.4rem;
  font-weight: 800;
  color: #ffd700;
  letter-spacing: 3px;
  text-shadow: 0 0 10px rgba(255, 215, 0, 0.6);
}

.hud-left,
.hud-right {
  display: flex;
  align-items: center;
  gap: 12px;
}

.stat-box {
  display: flex;
  flex-direction: column;
  align-items: center;
  background: rgba(255, 255, 255, 0.08);
  border: 1px solid rgba(255, 255, 255, 0.15);
  border-radius: 8px;
  padding: 4px 12px;
  min-width: 60px;
}

.stat-label {
  font-size: 0.65rem;
  color: #aaa;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.stat-value {
  font-size: 1.2rem;
  font-weight: 700;
  color: white;
}

.miss-count {
  color: #ff6b6b;
}

.hit-count {
  color: #51cf66;
}

.ammo-box {
  display: flex;
  gap: 2px;
  font-size: 0.8rem;
}

.bullet-icon {
  transition: opacity 0.3s;
}

.bullet-icon.used {
  opacity: 0.15;
  filter: grayscale(1);
}

.reload-btn {
  background: linear-gradient(135deg, #f39c12, #e67e22);
  color: white;
  border: none;
  border-radius: 8px;
  padding: 6px 14px;
  font-weight: 700;
  font-size: 0.8rem;
  cursor: pointer;
  letter-spacing: 1px;
  transition: all 0.2s;
  box-shadow: 0 2px 8px rgba(243, 156, 18, 0.4);
}

.home-btn {
  background: rgba(255, 255, 255, 0.1);
  color: #fff;
  border: 1.5px solid rgba(255, 255, 255, 0.3);
  border-radius: 8px;
  padding: 6px 14px;
  font-weight: 700;
  font-size: 0.8rem;
  cursor: pointer;
  letter-spacing: 1px;
  text-decoration: none;
  transition: all 0.2s;
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.home-btn:hover {
  background: rgba(255, 255, 255, 0.2);
  border-color: rgba(255, 255, 255, 0.6);
  transform: translateY(-1px);
}

.reload-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(243, 156, 18, 0.6);
}

/* Bird */
.bird {
  position: fixed;
  width: 90px;
  height: 70px;
  z-index: 50;
  transition: filter 0.1s;
}

.bird-body {
  position: relative;
}

.bird-svg svg {
  width: 90px;
  height: 70px;
  overflow: visible;
}

.bird-dir-left .bird-svg svg {
  transform: scaleX(-1);
}

.bird-dir-left .speech-bubble {
  right: auto;
  left: 100%;
  margin-left: 8px;
}

.bird-dodge {
  animation: dodgeShake 0.3s ease-out;
}

@keyframes dodgeShake {
  0% {
    filter: drop-shadow(0 0 8px rgba(255, 255, 0, 0.8));
  }

  50% {
    filter: drop-shadow(0 0 15px rgba(255, 100, 0, 1));
  }

  100% {
    filter: none;
  }
}

.bird-hurt {
  animation: hurtFlash 0.4s steps(2) 3;
}

@keyframes hurtFlash {
  0% {
    filter: brightness(3) saturate(0);
  }

  100% {
    filter: none;
  }
}

/* Speech bubble */
.speech-bubble-fixed {
  position: fixed;
  background: white;
  border: 2.5px solid #333;
  border-radius: 14px;
  padding: 10px 16px;
  font-size: 1rem;
  font-weight: 800;
  color: #222;
  white-space: nowrap;
  box-shadow: 4px 4px 0 rgba(0, 0, 0, 0.25);
  z-index: 200;
  pointer-events: none;
  transform: translate(-50%, -100%);
}

.speech-bubble-fixed::after {
  content: '';
  position: absolute;
  bottom: -12px;
  left: 50%;
  transform: translateX(-50%);
  border: 7px solid transparent;
  border-top-color: white;
}

.bubble-enter-active {
  animation: bubblePop 0.3s ease;
}

.bubble-leave-active {
  animation: bubblePop 0.2s ease reverse;
}

@keyframes bubblePop {
  from {
    transform: scale(0.5);
    opacity: 0;
  }

  to {
    transform: scale(1);
    opacity: 1;
  }
}

/* Bullet */
.bullet {
  position: fixed;
  font-size: 12px;
  z-index: 60;
  pointer-events: none;
  filter: drop-shadow(0 0 3px rgba(255, 200, 0, 0.8));
}

/* Miss flash */
.miss-flash {
  position: fixed;
  inset: 0;
  background: rgba(255, 0, 0, 0.12);
  z-index: 200;
  pointer-events: none;
  animation: flashAnim 0.15s ease-out;
}

@keyframes flashAnim {
  from {
    opacity: 1;
  }

  to {
    opacity: 0;
  }
}

/* Hit effect */
.hit-effect {
  position: fixed;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: rgba(0, 0, 0, 0.5);
  z-index: 300;
}

.hit-text {
  font-size: 3rem;
  font-weight: 900;
  color: #ff4757;
  text-shadow: 0 0 20px rgba(255, 71, 87, 0.8);
  animation: hitBounce 0.5s ease;
}

.feathers {
  font-size: 2rem;
  animation: feathersFall 1s ease-out forwards;
  margin-top: 12px;
}

@keyframes hitBounce {
  0% {
    transform: scale(0.3) rotate(-10deg);
    opacity: 0;
  }

  60% {
    transform: scale(1.2) rotate(3deg);
  }

  100% {
    transform: scale(1) rotate(0deg);
    opacity: 1;
  }
}

@keyframes feathersFall {
  from {
    transform: translateY(-20px);
    opacity: 1;
  }

  to {
    transform: translateY(60px);
    opacity: 0;
  }
}

/* Instruction */
.instruction {
  position: fixed;
  bottom: 30px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 10px 24px;
  border-radius: 30px;
  font-size: 0.9rem;
  border: 1px solid rgba(255, 255, 255, 0.2);
  z-index: 100;
  animation: fadeInUp 1s ease forwards;
}

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translate(-50%, 20px);
  }

  to {
    opacity: 1;
    transform: translate(-50%, 0);
  }
}
</style>
