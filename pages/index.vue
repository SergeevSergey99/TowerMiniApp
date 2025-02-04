<template>
  <div class="game-container">
    <canvas ref="gameCanvas"></canvas>

    <!-- Оверлей стартового экрана -->
    <div v-if="screen === 'start'" class="overlay start-screen">
      <h1>Tower Game</h1>
      <button @click="startGame">Start</button>
    </div>

    <!-- Оверлей экрана окончания игры -->
    <div v-if="screen === 'gameover'" class="overlay gameover-screen">
      <h1>Game Over</h1>
      <p>Score: {{ score }}</p>
      <button @click="restart">Restart</button>
    </div>

    <!-- Индикатор счета во время игры -->
    <div v-if="screen === 'game'" class="score-display">
      Score: {{ score }}
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      // Режим экрана: 'start', 'game', 'gameover'
      screen: 'start',

      // Параметры канваса и игры
      canvas: null,
      ctx: null,
      gameWidth: 300,
      gameHeight: 500,
      floorWidth: 150,
      floorHeight: 20,

      // Массив уже поставленных этажей и активный этаж
      tower: [],
      currentFloor: null,
      score: 0,
      highScore: 0,
      gameOver: false,
      animationFrameId: null,

      // Физика и параметры качания
      gravity: 0.5,
      maxSwingAngle: 0.5,

      // Накопленное горизонтальное отклонение (игровая логика)
      towerSway: 0,
      towerSwayVelocity: 0,
      swayThreshold: 100,  // теперь 100

      // Для вертикального скролла (не более 10 этажей на экране)
      cameraOffset: 0,
      targetCameraOffset: 0,

      // Константы для фиксированного положения троса и активного этажа (режим качания)
      fixedPivotY: 50,    // точка, где находится верхняя точка троса (фиксированная)
      activeFloorY: 100,   // в режиме качания верхняя грань активного этажа всегда на этой высоте (экранная координата)
      ropeLength: 50,      // расстояние между fixedPivotY и activeFloorY

      // Для падения: сохраняем экранную координату, откуда начинается падение
      // (используется только в режиме falling)
      // fallScreenY – экранная координата верхней границы блока во время падения
      // При переключении из swinging в falling мы зададим fallScreenY = activeFloorY
      // и будем её обновлять независимо от cameraOffset.
      // После приземления итоговая башенная координата блока: fallScreenY - cameraOffset.
      // Это позволяет блоку начинать падать с того же места, где он качался.
      overThresholdCount: 0
    }
  },
  mounted() {
    this.canvas = this.$refs.gameCanvas;
    this.canvas.width = this.gameWidth;
    this.canvas.height = this.gameHeight;
    this.ctx = this.canvas.getContext('2d');

    // Обработка тач- и click-событий (отпускание этажа)
    this.canvas.addEventListener('touchstart', (e) => {
      e.preventDefault();
      if (this.screen === 'game') this.handleTap();
    });
    this.canvas.addEventListener('click', () => {
      if (this.screen === 'game') this.handleTap();
    });

    const storedScore = localStorage.getItem('highScore');
    if (storedScore) {
      this.highScore = parseInt(storedScore);
    }
  },
  methods: {
    startGame() {
      this.screen = 'game';
      this.tower = [];
      this.score = 0;
      this.gameOver = false;
      this.towerSway = 0;
      this.towerSwayVelocity = 0;
      this.cameraOffset = 0;
      this.targetCameraOffset = 0;
      this.overThresholdCount = 0;

      // Базовый этаж – фиксирован внизу экрана
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
      // Опорная точка для троса: x — центр предыдущего этажа, y — фиксированное значение fixedPivotY
      const pivotX = lastFloor.x + this.floorWidth / 2;
      const pivotY = this.fixedPivotY;
      this.currentFloor = {
        state: 'swinging',  // 'swinging' – качается, 'falling' – падает
        angle: 0,
        angularVelocity: 0.03,
        pivotX: pivotX,
        pivotY: pivotY,
        ropeLength: this.ropeLength,
        // В режиме качания: верхняя грань этажа всегда на activeFloorY, x вычисляется по углу
        x: pivotX + this.ropeLength * Math.sin(0) - this.floorWidth / 2,
        y: this.activeFloorY,
        fallVY: 0,
        lockedX: null,  // При отпускании фиксируется горизонтальное положение
        fallScreenY: null // Будет установлена при переключении в режим falling
      }
    },
    handleTap() {
      if (this.currentFloor && this.currentFloor.state === 'swinging') {
        // Фиксируем горизонтальное положение
        this.currentFloor.lockedX = this.currentFloor.x;
        // При переключении в falling устанавливаем экранную координату падения равной activeFloorY
        this.currentFloor.fallScreenY = this.activeFloorY;
        this.currentFloor.state = 'falling';
      }
    },
    animate() {
      this.animationFrameId = requestAnimationFrame(this.animate);
      this.update();
      this.render();
    },
    update() {
      if (this.currentFloor) {
        if (this.currentFloor.state === 'swinging') {
          this.currentFloor.angle += this.currentFloor.angularVelocity;
          if (this.currentFloor.angle > this.maxSwingAngle || this.currentFloor.angle < -this.maxSwingAngle) {
            this.currentFloor.angularVelocity = -this.currentFloor.angularVelocity;
          }
          this.currentFloor.x = this.currentFloor.pivotX + this.currentFloor.ropeLength * Math.sin(this.currentFloor.angle) - this.floorWidth / 2;
          // В режиме swinging y всегда равен activeFloorY
          this.currentFloor.y = this.activeFloorY;
        } else if (this.currentFloor.state === 'falling') {
          // Фиксированное горизонтальное положение
          if (this.currentFloor.lockedX !== null) {
            this.currentFloor.x = this.currentFloor.lockedX;
          }
          // Обновляем fallScreenY (экранную координату падения) независимо от cameraOffset
          this.currentFloor.fallVY += this.gravity;
          this.currentFloor.fallScreenY += this.currentFloor.fallVY;
          // Логика столкновения: сравниваем fallScreenY + floorHeight с верхом последнего этажа (в экранных координатах)
          const lastFloor = this.tower[this.tower.length - 1];
          if (this.currentFloor.fallScreenY + this.floorHeight >= lastFloor.y + this.cameraOffset) {
            // При столкновении вычисляем итоговую башенную координату: fallScreenY - cameraOffset
            this.currentFloor.y = this.currentFloor.fallScreenY - this.cameraOffset;
            // Корректируем так, чтобы блок точно прилегал к предыдущему
            this.currentFloor.y = lastFloor.y - this.floorHeight;
            const currentCenter = this.currentFloor.x + this.floorWidth / 2;
            const lastCenter = lastFloor.x + this.floorWidth / 2;
            const deltaX = currentCenter - lastCenter;
            this.towerSwayVelocity += deltaX * 0.1;
            // Добавляем блок в башню
            this.tower.push({
              x: this.currentFloor.x,
              y: this.currentFloor.y
            });
            this.score++;
            this.createNextFloor();
          }
        }
      }

      // Обновляем горизонтальное отклонение (эффект "шатания")
      this.towerSwayVelocity *= 0.98;
      this.towerSway += this.towerSwayVelocity;

      // Плавный вертикальный скролл: если башня содержит более 10 этажей,
      // нижний из последних 10 этажей должен быть на позиции (gameHeight - floorHeight)
      if (this.tower.length > 10) {
        const bottomVisibleFloor = this.tower[this.tower.length - 10];
        this.targetCameraOffset = (this.gameHeight - this.floorHeight) - bottomVisibleFloor.y;
      } else {
        this.targetCameraOffset = 0;
      }
      this.cameraOffset += (this.targetCameraOffset - this.cameraOffset) * 0.1;

      // Удаляем этажи, вышедшие за нижнюю границу
      while (this.tower.length && (this.tower[0].y + this.cameraOffset) > this.gameHeight) {
        this.tower.shift();
      }

      // Проверка горизонтального отклонения для поражения (как и раньше)
      if (Math.abs(this.towerSway + this.towerSwayVelocity) > this.swayThreshold) {
        this.overThresholdCount++;
        if (this.overThresholdCount > 20) {
          this.endGame();
          return;
        }
      } else {
        this.overThresholdCount = 0;
      }
    },
    render() {
      this.ctx.clearRect(0, 0, this.gameWidth, this.gameHeight);

      // Вычисляем горизонтальное смещение для центровки башни по верхнему блоку
      let centerOffset = 0;
      if (this.tower.length) {
        const topBlock = this.tower[this.tower.length - 1];
        centerOffset = (this.gameWidth / 2) - (topBlock.x + this.floorWidth / 2);
      }

      // 1. Отрисовка уже поставленных этажей (с вертикальным скроллом и центровкой)
      this.ctx.save();
      this.ctx.translate(centerOffset, this.cameraOffset);
      this.tower.forEach(floor => {
        this.ctx.fillStyle = '#3498db';
        this.ctx.fillRect(floor.x, floor.y, this.floorWidth, this.floorHeight);
      });
      this.ctx.restore();

      // 2. Отрисовка активного блока
      this.ctx.save();
      // Если блок качается, он отрисовывается без vertical cameraOffset (фиксированная высота)
      // Если падает, то мы НЕ добавляем cameraOffset, чтобы его позиция соответствовала fallScreenY
      this.ctx.translate(centerOffset, 0);
      if (this.currentFloor) {
        if (this.currentFloor.state === 'swinging') {
          this.ctx.strokeStyle = '#555';
          this.ctx.lineWidth = 2;
          this.ctx.beginPath();
          this.ctx.moveTo(this.currentFloor.pivotX, this.fixedPivotY);
          const blockTopCenterX = this.currentFloor.x + this.floorWidth / 2;
          this.ctx.lineTo(blockTopCenterX, this.activeFloorY);
          this.ctx.stroke();
          // Рисуем блок в положении (x, activeFloorY)
          this.ctx.fillStyle = '#e74c3c';
          this.ctx.fillRect(this.currentFloor.x, this.activeFloorY, this.floorWidth, this.floorHeight);
        } else if (this.currentFloor.state === 'falling') {
          // Рисуем падающий блок по его экранной координате fallScreenY
          this.ctx.fillStyle = '#e74c3c';
          this.ctx.fillRect(this.currentFloor.x, this.currentFloor.fallScreenY, this.floorWidth, this.floorHeight);
        }
      }
      this.ctx.restore();

      // 3. Отрисовка шкалы отклонения в нижней части экрана
      const scaleWidth = 200;
      const scaleHeight = 10;
      const scaleX = (this.gameWidth - scaleWidth) / 2;
      const scaleY = this.gameHeight - 30;

      // Зона безопасности (зеленая)
      this.ctx.fillStyle = "#0f0";
      this.ctx.fillRect(scaleX, scaleY, scaleWidth, scaleHeight);

      // Границы шкалы
      this.ctx.strokeStyle = "#000";
      this.ctx.strokeRect(scaleX, scaleY, scaleWidth, scaleHeight);

      // Вычисляем позицию маркера текущего отклонения
      const markerX = scaleX + ((this.towerSway + this.swayThreshold) / (this.swayThreshold * 2)) * scaleWidth;

      this.ctx.strokeStyle = (Math.abs(this.towerSway) > this.swayThreshold) ? "#f00" : "#000";
      this.ctx.beginPath();
      this.ctx.moveTo(markerX, scaleY);
      this.ctx.lineTo(markerX, scaleY + scaleHeight);
      this.ctx.stroke();

      // Подписи шкалы
      this.ctx.font = "12px Arial";
      this.ctx.fillStyle = "#000";
      this.ctx.fillText("-100", scaleX - 25, scaleY + scaleHeight);
      this.ctx.fillText("0", scaleX + scaleWidth / 2 - 5, scaleY + scaleHeight + 15);
      this.ctx.fillText("+100", scaleX + scaleWidth + 5, scaleY + scaleHeight);
    },
    endGame() {
      this.gameOver = true;
      cancelAnimationFrame(this.animationFrameId);
      this.screen = 'gameover';
      if (this.score > this.highScore) {
        this.highScore = this.score;
        localStorage.setItem('highScore', this.highScore);
      }
    },
    restart() {
      cancelAnimationFrame(this.animationFrameId);
      this.screen = 'start';
    }
  }
}
</script>

<style scoped>
.game-container {
  position: relative;
  margin: 0 auto;
  width: 300px;
  height: 500px;
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
.score-display {
  position: absolute;
  top: 10px;
  left: 10px;
  font-size: 18px;
  color: #333;
}
.overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 300px;
  height: 500px;
  background: rgba(255, 255, 255, 0.9);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}
.overlay h1 {
  margin-bottom: 20px;
}
.overlay button {
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}
</style>
