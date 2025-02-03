<template>
  <div class="game-container">
    <canvas ref="gameCanvas"></canvas>
    <div class="score">Счёт: {{ score }}</div>
    <div class="sway-indicator" :style="{ color: Math.abs(towerSway) > swayThreshold * 0.8 ? 'red' : 'black' }">
      Сдвиг башни: {{ Math.round(towerSway) }} px / Предел: {{ swayThreshold }} px
    </div>
    <button v-if="!gameOver" @click="handleTap" class="tap-button">
      Отпустить этаж
    </button>
    <button v-if="!gameOver && Math.abs(towerSway) > 5" @click="stabilizeTower" class="stabilize-button">
      Стабилизировать башню
    </button>
    <div v-if="gameOver" class="game-over">
      Игра окончена!<br />
      Ваш счёт: {{ score }}<br />
      Рекорд: {{ highScore }}<br />
      <button @click="restart">Начать заново</button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      // Основные параметры игры и канваса
      canvas: null,
      ctx: null,
      gameWidth: 300,
      gameHeight: 500,
      floorWidth: 150,
      floorHeight: 20,
      tower: [],
      currentFloor: null,
      score: 0,
      highScore: 0,
      gameOver: false,
      animationFrameId: null,
      gravity: 0.5,
      maxSwingAngle: 0.5,
      towerSway: 0,
      towerSwayVelocity: 0,
      swayThreshold: 100,
      cameraOffset: 0,
      targetCameraOffset: 0,
      // Новые константы для фиксированного троса
      fixedPivotY: 50,    // фиксированная высота, где находится верхняя точка (троса)
      activeFloorY: 100,    // активный этаж (его верх) всегда на этой высоте при качании
      ropeLength: 50        // длина троса (activeFloorY - fixedPivotY)
    }
  },
  mounted() {
    this.canvas = this.$refs.gameCanvas;
    this.canvas.width = this.gameWidth;
    this.canvas.height = this.gameHeight;
    this.ctx = this.canvas.getContext('2d');

    // Поддержка тач-событий для мобильных устройств
    this.canvas.addEventListener('touchstart', (e) => {
      e.preventDefault();
      this.handleTap();
    });
    this.canvas.addEventListener('click', () => {
      this.handleTap();
    });

    const storedScore = localStorage.getItem('highScore');
    if (storedScore) {
      this.highScore = parseInt(storedScore);
    }
    this.startGame();
  },
  beforeDestroy() {
    cancelAnimationFrame(this.animationFrameId);
  },
  methods: {
    startGame() {
      // Сброс игровых переменных
      this.tower = [];
      this.score = 0;
      this.gameOver = false;
      this.towerSway = 0;
      this.towerSwayVelocity = 0;
      this.cameraOffset = 0;
      this.targetCameraOffset = 0;

      // Базовый этаж – фиксирован внизу «мира»
      const baseFloor = {
        x: (this.gameWidth - this.floorWidth) / 2,
        y: this.gameHeight - this.floorHeight
      };
      this.tower.push(baseFloor);
      this.createNextFloor();
      this.animate();
    },
    createNextFloor() {
      const lastFloor = this.tower[this.tower.length - 1];
      // Опорная точка для троса: x – центр предыдущего этажа, y – фиксированное значение (fixedPivotY)
      const pivotX = lastFloor.x + this.floorWidth / 2;
      const pivotY = this.fixedPivotY;
      const ropeLen = this.ropeLength;
      this.currentFloor = {
        state: 'swinging',  // состояния: 'swinging' и 'falling'
        angle: 0,
        angularVelocity: 0.03,
        pivotX: pivotX,
        pivotY: pivotY,
        ropeLength: ropeLen,
        // При качании активный этаж всегда находится на высоте activeFloorY,
        // а его x вычисляется от pivotX с использованием sin(angle)
        x: pivotX + ropeLen * Math.sin(0) - this.floorWidth / 2,
        y: this.activeFloorY,
        fallVY: 0
      }
    },
    handleTap() {
      if (this.gameOver || !this.currentFloor) return;
      if (this.currentFloor.state === 'swinging') {
        this.currentFloor.state = 'falling';
      }
    },
    stabilizeTower() {
      this.towerSway *= 0.5;
      this.towerSwayVelocity *= 0.5;
    },
    animate() {
      this.animationFrameId = requestAnimationFrame(this.animate);
      this.update();
      this.render();
    },
    update() {
      if (this.currentFloor) {
        if (this.currentFloor.state === 'swinging') {
          // Обновляем угол качания и только горизонтальное смещение,
          // а вертикальная координата активного этажа остаётся равной activeFloorY.
          this.currentFloor.angle += this.currentFloor.angularVelocity;
          if (this.currentFloor.angle > this.maxSwingAngle || this.currentFloor.angle < -this.maxSwingAngle) {
            this.currentFloor.angularVelocity = -this.currentFloor.angularVelocity;
          }
          this.currentFloor.x = this.currentFloor.pivotX + this.currentFloor.ropeLength * Math.sin(this.currentFloor.angle) - this.floorWidth / 2;
          this.currentFloor.y = this.activeFloorY;
        } else if (this.currentFloor.state === 'falling') {
          // Режим падения: активный этаж начинает двигаться вниз с действием гравитации
          this.currentFloor.fallVY += this.gravity;
          this.currentFloor.y += this.currentFloor.fallVY;
          const lastFloor = this.tower[this.tower.length - 1];
          if (this.currentFloor.y + this.floorHeight >= lastFloor.y) {
            // При столкновении выравниваем этаж точно на верх предыдущего
            this.currentFloor.y = lastFloor.y - this.floorHeight;
            const currentCenter = this.currentFloor.x + this.floorWidth / 2;
            const lastCenter = lastFloor.x + this.floorWidth / 2;
            const deltaX = currentCenter - lastCenter;
            this.towerSwayVelocity += deltaX * 0.1;
            this.tower.push({
              x: this.currentFloor.x,
              y: this.currentFloor.y
            });
            this.score++;
            if (Math.abs(this.towerSway + this.towerSwayVelocity) > this.swayThreshold) {
              this.endGame();
              return;
            }
            this.createNextFloor();
          }
        }
      }
      // Обновляем горизонтальное смещение с эффектом демпфирования
      this.towerSwayVelocity *= 0.98;
      this.towerSway += this.towerSwayVelocity;

      // Если построено более 10 этажей, начинаем плавный вертикальный скролл
      if (this.tower.length > 10) {
        const bottomVisibleFloor = this.tower[this.tower.length - 10];
        this.targetCameraOffset = (this.gameHeight - this.floorHeight) - bottomVisibleFloor.y;
      } else {
        this.targetCameraOffset = 0;
      }
      this.cameraOffset += (this.targetCameraOffset - this.cameraOffset) * 0.1;

      // Удаляем этажи, ушедшие за нижнюю границу экрана
      while (this.tower.length && (this.tower[0].y + this.cameraOffset) > this.gameHeight) {
        this.tower.shift();
      }
    },
    render() {
      this.ctx.clearRect(0, 0, this.gameWidth, this.gameHeight);

      // 1. Отрисовка уже поставленных этажей с вертикальным сдвигом (камера)
      this.ctx.save();
      this.ctx.translate(this.towerSway, this.cameraOffset);
      this.tower.forEach(floor => {
        this.ctx.fillStyle = '#3498db';
        this.ctx.fillRect(floor.x, floor.y, this.floorWidth, this.floorHeight);
      });
      this.ctx.restore();

      // 2. Отрисовка активного этажа и его троса без вертикального сдвига,
      // чтобы он оставался на фиксированной высоте
      if (this.currentFloor) {
        if (this.currentFloor.state === 'swinging') {
          this.ctx.strokeStyle = '#555';
          this.ctx.lineWidth = 2;
          this.ctx.beginPath();
          // Трос всегда начинается от фиксированной точки (pivot) с y = fixedPivotY
          this.ctx.moveTo(this.currentFloor.pivotX, this.currentFloor.pivotY);
          const blockTopCenterX = this.currentFloor.x + this.floorWidth / 2;
          const blockTopCenterY = this.activeFloorY;
          this.ctx.lineTo(blockTopCenterX, blockTopCenterY);
          this.ctx.stroke();
        }
        this.ctx.fillStyle = '#e74c3c';
        this.ctx.fillRect(this.currentFloor.x, this.currentFloor.y, this.floorWidth, this.floorHeight);
      }
    },
    endGame() {
      this.gameOver = true;
      cancelAnimationFrame(this.animationFrameId);
      if (this.score > this.highScore) {
        this.highScore = this.score;
        localStorage.setItem('highScore', this.highScore);
      }
    },
    restart() {
      cancelAnimationFrame(this.animationFrameId);
      this.startGame();
    }
  }
}
</script>

<style scoped>
.game-container {
  margin: 0 auto;
  width: 300px;
  text-align: center;
  user-select: none;
}
canvas {
  border: 1px solid #333;
  background: #ecf0f1;
  display: block;
  margin: 0 auto;
  touch-action: manipulation;
}
.score {
  font-size: 18px;
  margin-top: 10px;
}
.sway-indicator {
  margin-top: 5px;
  font-size: 14px;
}
.tap-button,
.stabilize-button {
  margin-top: 10px;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}
.game-over {
  margin-top: 10px;
  padding: 10px;
  border: 1px solid #333;
  background: #fff;
}
</style>
