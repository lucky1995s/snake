<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue';

const canvasRef = ref<HTMLCanvasElement | null>(null);
const score = ref(0);
const bestScore = ref(0);
const isRunning = ref(false);
const gameOver = ref(false);
const speedLevel = ref<'slow' | 'normal' | 'fast'>('normal');

// 网格配置
const tileSize = 20;
const tileCountX = 20;
const tileCountY = 20;

type Direction = 'left' | 'right' | 'up' | 'down';

let snake: { x: number; y: number }[] = [];
let food: { x: number; y: number } = { x: 10, y: 10 };
let direction: Direction = 'right';
let pendingDirection: Direction | null = null;
let loopId: number | null = null;

const getSpeedMs = () => {
  if (speedLevel.value === 'slow') return 200;
  if (speedLevel.value === 'fast') return 80;
  return 120;
};

const initGame = () => {
  score.value = 0;
  gameOver.value = false;
  isRunning.value = true;
  direction = 'right';
  pendingDirection = null;
  snake = [
    { x: 8, y: 10 },
    { x: 7, y: 10 },
    { x: 6, y: 10 }
  ];
  spawnFood();
};

const spawnFood = () => {
  while (true) {
    const x = Math.floor(Math.random() * tileCountX);
    const y = Math.floor(Math.random() * tileCountY);
    const onSnake = snake.some((s) => s.x === x && s.y === y);
    if (!onSnake) {
      food = { x, y };
      break;
    }
  }
};

const changeDirection = (next: Direction) => {
  // 防止反向
  if (direction === 'left' && next === 'right') return;
  if (direction === 'right' && next === 'left') return;
  if (direction === 'up' && next === 'down') return;
  if (direction === 'down' && next === 'up') return;
  pendingDirection = next;
};

const handleKeydown = (e: KeyboardEvent) => {
  if (!isRunning.value && !gameOver.value && (e.key === ' ' || e.key === 'Enter')) {
    startGame();
    return;
  }

  if (gameOver.value && (e.key === ' ' || e.key === 'Enter')) {
    restartGame();
    return;
  }

  switch (e.key) {
    case 'ArrowLeft':
    case 'a':
    case 'A':
      changeDirection('left');
      break;
    case 'ArrowRight':
    case 'd':
    case 'D':
      changeDirection('right');
      break;
    case 'ArrowUp':
    case 'w':
    case 'W':
      changeDirection('up');
      break;
    case 'ArrowDown':
    case 's':
    case 'S':
      changeDirection('down');
      break;
  }
};

const draw = () => {
  const canvas = canvasRef.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  // 背景
  ctx.fillStyle = '#020617';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  // 网格
  ctx.strokeStyle = 'rgba(148, 163, 184, 0.08)';
  ctx.lineWidth = 1;
  for (let x = 0; x <= tileCountX; x++) {
    ctx.beginPath();
    ctx.moveTo(x * tileSize + 0.5, 0);
    ctx.lineTo(x * tileSize + 0.5, canvas.height);
    ctx.stroke();
  }
  for (let y = 0; y <= tileCountY; y++) {
    ctx.beginPath();
    ctx.moveTo(0, y * tileSize + 0.5);
    ctx.lineTo(canvas.width, y * tileSize + 0.5);
    ctx.stroke();
  }

  // 食物
  const foodGradient = ctx.createRadialGradient(
    food.x * tileSize + tileSize / 2,
    food.y * tileSize + tileSize / 2,
    2,
    food.x * tileSize + tileSize / 2,
    food.y * tileSize + tileSize / 2,
    tileSize / 1.4
  );
  foodGradient.addColorStop(0, '#22c55e');
  foodGradient.addColorStop(1, '#16a34a');
  ctx.fillStyle = foodGradient;
  ctx.beginPath();
  ctx.roundRect(food.x * tileSize + 3, food.y * tileSize + 3, tileSize - 6, tileSize - 6, 5);
  ctx.fill();

  // 蛇
  snake.forEach((segment, index) => {
    const isHead = index === 0;
    const x = segment.x * tileSize;
    const y = segment.y * tileSize;

    const gradient = ctx.createLinearGradient(x, y, x, y + tileSize);
    if (isHead) {
      gradient.addColorStop(0, '#38bdf8');
      gradient.addColorStop(1, '#0ea5e9');
    } else {
      gradient.addColorStop(0, '#1d4ed8');
      gradient.addColorStop(1, '#0b4eb8');
    }

    ctx.fillStyle = gradient;
    ctx.beginPath();
    ctx.roundRect(x + 2, y + 2, tileSize - 4, tileSize - 4, isHead ? 6 : 4);
    ctx.fill();

    if (isHead) {
      // 眼睛
      ctx.fillStyle = '#0f172a';
      const eyeSize = 3;
      let eyeOffsetX = 4;
      let eyeOffsetY = 4;
      if (direction === 'left') {
        eyeOffsetX = 4;
        eyeOffsetY = 5;
      } else if (direction === 'right') {
        eyeOffsetX = tileSize - 4 - eyeSize;
        eyeOffsetY = 5;
      } else if (direction === 'up') {
        eyeOffsetX = 4;
        eyeOffsetY = 4;
      } else if (direction === 'down') {
        eyeOffsetX = 4;
        eyeOffsetY = tileSize - 4 - eyeSize;
      }

      ctx.fillRect(x + eyeOffsetX, y + eyeOffsetY, eyeSize, eyeSize);
      ctx.fillRect(x + tileSize - eyeOffsetX - eyeSize, y + eyeOffsetY, eyeSize, eyeSize);
    }
  });

  // Game Over 遮罩
  if (gameOver.value) {
    ctx.fillStyle = 'rgba(15,23,42,0.85)';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    ctx.fillStyle = '#e2e8f0';
    ctx.font = 'bold 26px system-ui, -apple-system, BlinkMacSystemFont, "SF Pro Text"';
    ctx.textAlign = 'center';
    ctx.fillText('游戏结束', canvas.width / 2, canvas.height / 2 - 10);

    ctx.fillStyle = '#94a3b8';
    ctx.font = '14px system-ui, -apple-system, BlinkMacSystemFont, "SF Pro Text"';
    ctx.fillText(`得分：${score.value}`, canvas.width / 2, canvas.height / 2 + 18);
    ctx.fillText('按空格或回车重新开始', canvas.width / 2, canvas.height / 2 + 40);
  }
};

const update = () => {
  if (!isRunning.value || gameOver.value) return;

  // 应用等待中的方向变更（每帧只应用一次，防止折返）
  if (pendingDirection) {
    direction = pendingDirection;
    pendingDirection = null;
  }

  const head = snake[0];
  let newHead = { ...head };

  switch (direction) {
    case 'left':
      newHead.x -= 1;
      break;
    case 'right':
      newHead.x += 1;
      break;
    case 'up':
      newHead.y -= 1;
      break;
    case 'down':
      newHead.y += 1;
      break;
  }

  // 撞墙
  if (newHead.x < 0 || newHead.x >= tileCountX || newHead.y < 0 || newHead.y >= tileCountY) {
    handleGameOver();
    return;
  }

  // 撞到自己
  if (snake.some((s) => s.x === newHead.x && s.y === newHead.y)) {
    handleGameOver();
    return;
  }

  // 移动
  snake.unshift(newHead);

  // 吃到食物
  if (newHead.x === food.x && newHead.y === food.y) {
    score.value += 1;
    if (score.value > bestScore.value) {
      bestScore.value = score.value;
      if (process.client) {
        localStorage.setItem('snake-best-score', String(bestScore.value));
      }
    }
    spawnFood();
  } else {
    snake.pop();
  }

  draw();
};

const handleGameOver = () => {
  gameOver.value = true;
  isRunning.value = false;
  if (loopId !== null) {
    clearInterval(loopId);
    loopId = null;
  }
  draw();
};

const loop = () => {
  if (loopId !== null) {
    clearInterval(loopId);
  }
  loopId = window.setInterval(update, getSpeedMs());
};

const startGame = () => {
  if (isRunning.value) return;
  initGame();
  draw();
  loop();
};

const restartGame = () => {
  initGame();
  draw();
  loop();
};

const stopGame = () => {
  isRunning.value = false;
  if (loopId !== null) {
    clearInterval(loopId);
    loopId = null;
  }
};

const handleSpeedChange = (level: 'slow' | 'normal' | 'fast') => {
  speedLevel.value = level;
  if (isRunning.value) {
    loop();
  }
};

onMounted(() => {
  if (process.client) {
    const stored = localStorage.getItem('snake-best-score');
    if (stored) bestScore.value = Number(stored) || 0;
  }

  const canvas = canvasRef.value;
  if (canvas) {
    canvas.width = tileSize * tileCountX;
    canvas.height = tileSize * tileCountY;
    draw();
  }

  window.addEventListener('keydown', handleKeydown);
});

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeydown);
  if (loopId !== null) {
    clearInterval(loopId);
  }
});
</script>

<template>
  <div class="page">
    <div class="container">
      <header class="header">
        <div>
          <p class="eyebrow">
            小游戏 · 贪吃蛇
          </p>
          <h1 class="title">
            霓虹贪吃蛇
          </h1>
          <p class="subtitle">
            使用键盘方向键或
            <span class="kbd">WASD</span>
            控制蛇移动，吃到能量方块获得分数，注意不要撞墙或者撞到自己。
          </p>
        </div>

        <div class="score-panel">
          <div class="score-item">
            <span class="score-label">当前分数</span>
            <span class="score-value">{{ score }}</span>
          </div>
          <div class="score-item">
            <span class="score-label">历史最高</span>
            <span class="score-value score-best">{{ bestScore }}</span>
          </div>
        </div>
      </header>

      <main class="main">
        <section class="board-card">
          <div class="board-header">
            <div class="status">
              <span class="status-dot" :class="{ 'status-dot-running': isRunning }" />
              <span class="status-text">
                {{ gameOver ? '游戏结束' : isRunning ? '游戏中' : '待开始' }}
              </span>
            </div>

            <div class="speed-group">
              <button
                type="button"
                class="speed-btn"
                :class="{ 'speed-btn-active': speedLevel === 'slow' }"
                @click="handleSpeedChange('slow')"
              >
                慢速
              </button>
              <button
                type="button"
                class="speed-btn"
                :class="{ 'speed-btn-active': speedLevel === 'normal' }"
                @click="handleSpeedChange('normal')"
              >
                正常
              </button>
              <button
                type="button"
                class="speed-btn"
                :class="{ 'speed-btn-active': speedLevel === 'fast' }"
                @click="handleSpeedChange('fast')"
              >
                极速
              </button>
            </div>
          </div>

          <div class="board-wrapper">
            <canvas ref="canvasRef" class="board-canvas" />

            <div v-if="!isRunning && !gameOver" class="overlay">
              <p class="overlay-title">
                准备好了吗？
              </p>
              <p class="overlay-subtitle">
                点击下方按钮或按空格 / 回车开始游戏
              </p>
            </div>
          </div>

          <div class="actions">
            <button
              v-if="!isRunning && !gameOver"
              type="button"
              class="btn btn-primary"
              @click="startGame"
            >
              开始游戏
            </button>
            <button
              v-else-if="isRunning"
              type="button"
              class="btn btn-secondary"
              @click="stopGame"
            >
              暂停
            </button>
            <button
              v-else
              type="button"
              class="btn btn-primary"
              @click="restartGame"
            >
              重新开始
            </button>
          </div>

          <p class="hint">
            操作提示：方向键 或 WASD 控制方向，空格 / 回车 开始或重新开始。
          </p>
        </section>
      </main>
    </div>
  </div>
</template>

<style scoped>
.page {
  min-height: 100vh;
  display: flex;
  justify-content: center;
  padding: 3.5rem 1.5rem;
  background: radial-gradient(circle at top, #0f172a 0, #020617 50%, #020617 100%);
  color: #e2e8f0;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'SF Pro Text', sans-serif;
}

.container {
  width: 100%;
  max-width: 960px;
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

.header {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

@media (min-width: 768px) {
  .header {
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
  }
}

.eyebrow {
  font-size: 0.75rem;
  letter-spacing: 0.2em;
  text-transform: uppercase;
  color: #22d3ee;
  font-weight: 600;
}

.title {
  margin-top: 0.35rem;
  font-size: 2rem;
  font-weight: 600;
  letter-spacing: -0.03em;
  color: #f9fafb;
}

@media (min-width: 640px) {
  .title {
    font-size: 2.4rem;
  }
}

.subtitle {
  margin-top: 0.5rem;
  font-size: 0.85rem;
  line-height: 1.6;
  color: #cbd5f5;
  max-width: 32rem;
}

.kbd {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0 0.35rem;
  margin: 0 0.15rem;
  height: 1.1rem;
  border-radius: 0.3rem;
  background-color: rgba(15, 23, 42, 0.9);
  border: 1px solid rgba(148, 163, 184, 0.7);
  font-size: 0.7rem;
  font-weight: 500;
}

.score-panel {
  display: flex;
  gap: 0.75rem;
}

@media (max-width: 480px) {
  .score-panel {
    width: 100%;
    justify-content: flex-start;
  }
}

.score-item {
  min-width: 7rem;
  padding: 0.6rem 0.85rem;
  border-radius: 0.9rem;
  border: 1px solid rgba(148, 163, 184, 0.45);
  background: radial-gradient(circle at top left, rgba(37, 99, 235, 0.4), rgba(15, 23, 42, 0.9));
  box-shadow: 0 18px 40px rgba(15, 23, 42, 0.9);
}

.score-label {
  display: block;
  font-size: 0.7rem;
  color: #9ca3af;
}

.score-value {
  display: block;
  margin-top: 0.15rem;
  font-size: 1.3rem;
  font-weight: 600;
  letter-spacing: 0.04em;
}

.score-best {
  color: #22c55e;
}

.main {
  display: flex;
  flex-direction: column;
}

.board-card {
  border-radius: 1.5rem;
  border: 1px solid rgba(30, 64, 175, 0.9);
  background: radial-gradient(circle at top, rgba(56, 189, 248, 0.18), rgba(15, 23, 42, 0.98));
  padding: 1.4rem;
  box-shadow:
    0 22px 70px rgba(15, 23, 42, 1),
    0 0 0 1px rgba(15, 23, 42, 1);
}

@media (min-width: 768px) {
  .board-card {
    padding: 1.6rem 1.7rem;
  }
}

.board-header {
  display: flex;
  flex-direction: column;
  gap: 0.8rem;
}

@media (min-width: 640px) {
  .board-header {
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
  }
}

.status {
  display: inline-flex;
  align-items: center;
  gap: 0.4rem;
  padding: 0.25rem 0.6rem;
  border-radius: 999px;
  border: 1px solid rgba(51, 65, 85, 0.8);
  background-color: rgba(15, 23, 42, 0.9);
}

.status-dot {
  width: 0.55rem;
  height: 0.55rem;
  border-radius: 999px;
  background-color: #64748b;
  box-shadow: 0 0 0 0 rgba(148, 163, 184, 0.5);
  transition:
    background-color 0.2s ease,
    box-shadow 0.2s ease;
}

.status-dot-running {
  background-color: #22c55e;
  box-shadow: 0 0 0 6px rgba(34, 197, 94, 0.25);
}

.status-text {
  font-size: 0.75rem;
  color: #cbd5f5;
}

.speed-group {
  display: inline-flex;
  padding: 0.18rem;
  border-radius: 999px;
  border: 1px solid rgba(51, 65, 85, 0.85);
  background-color: rgba(15, 23, 42, 0.9);
}

.speed-btn {
  position: relative;
  min-width: 3.4rem;
  padding: 0.3rem 0.7rem;
  border-radius: 999px;
  border: none;
  background: transparent;
  color: #9ca3af;
  font-size: 0.75rem;
  cursor: pointer;
  transition:
    color 0.2s ease,
    background 0.2s ease,
    box-shadow 0.2s ease;
}

.speed-btn-active {
  color: #e5f3ff;
  background: radial-gradient(circle at top, rgba(96, 165, 250, 0.5), rgba(37, 99, 235, 0.8));
  box-shadow: 0 0 0 1px rgba(56, 189, 248, 0.8);
}

.speed-btn:not(.speed-btn-active):hover {
  background-color: rgba(30, 64, 175, 0.5);
  color: #e5f3ff;
}

.board-wrapper {
  position: relative;
  margin-top: 1.2rem;
  border-radius: 1.2rem;
  overflow: hidden;
  border: 1px solid rgba(15, 23, 42, 0.9);
  box-shadow:
    0 0 0 1px rgba(30, 64, 175, 0.9),
    0 24px 80px rgba(15, 23, 42, 0.95);
}

.board-canvas {
  display: block;
  width: 100%;
  height: auto;
  background-color: #020617;
}

.overlay {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: radial-gradient(circle at top, rgba(30, 64, 175, 0.4), rgba(15, 23, 42, 0.94));
  text-align: center;
  padding: 1.5rem;
}

.overlay-title {
  font-size: 1.2rem;
  font-weight: 600;
  letter-spacing: 0.08em;
}

.overlay-subtitle {
  margin-top: 0.5rem;
  font-size: 0.8rem;
  color: #cbd5f5;
}

.actions {
  margin-top: 1.1rem;
  display: flex;
  flex-wrap: wrap;
  gap: 0.6rem;
  justify-content: space-between;
  align-items: center;
}

.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0.55rem 1.4rem;
  border-radius: 999px;
  font-size: 0.85rem;
  font-weight: 500;
  border: none;
  cursor: pointer;
  transition:
    transform 0.1s ease,
    box-shadow 0.2s ease,
    background 0.2s ease,
    color 0.2s ease;
}

.btn-primary {
  background: linear-gradient(135deg, #22c55e, #16a34a);
  color: #052e16;
  box-shadow:
    0 14px 30px rgba(22, 163, 74, 0.45),
    0 0 0 1px rgba(21, 128, 61, 0.9);
}

.btn-primary:hover {
  transform: translateY(-1px);
  box-shadow:
    0 18px 40px rgba(22, 163, 74, 0.6),
    0 0 0 1px rgba(21, 128, 61, 0.9);
}

.btn-secondary {
  background: linear-gradient(135deg, #3b82f6, #1d4ed8);
  color: #e5f3ff;
  box-shadow:
    0 14px 30px rgba(37, 99, 235, 0.55),
    0 0 0 1px rgba(37, 99, 235, 0.9);
}

.btn-secondary:hover {
  transform: translateY(-1px);
  box-shadow:
    0 18px 40px rgba(37, 99, 235, 0.75),
    0 0 0 1px rgba(37, 99, 235, 0.9);
}

.hint {
  margin-top: 0.75rem;
  font-size: 0.78rem;
  color: #9ca3af;
}
</style>


