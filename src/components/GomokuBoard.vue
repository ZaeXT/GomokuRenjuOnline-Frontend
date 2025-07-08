<template>
  <div class="board-container">
    <svg 
      :width="boardSize" 
      :height="boardSize" 
      :viewBox="`0 0 ${boardSize} ${boardSize}`"
      xmlns="http://www.w3.org/2000/svg"
      @click="handleClick"
      @mousemove="handleMovement"
      @mouseleave="ghostPiece.visible = false"
      @mouseenter="ghostPiece.visible = true;handleMovement($event)"
    >
      <!-- 1. 棋盘背景 (一个带边框的矩形) -->
      <rect 
        x="0" 
        y="0" 
        :width="boardSize" 
        :height="boardSize" 
        class="board-bg"
      />

      <!-- 2. 绘制所有横线 -->
      <g class="board-lines">
        <line 
          v-for="i in GoBoardVisibleSize" 
          :key="'h' + i"
          :x1="svgPos(0)" 
          :y1="svgPos(i - 1)" 
          :x2="svgPos(GoBoardVisibleSize - 1)" 
          :y2="svgPos(i - 1)"
        />
      </g>

      <!-- 3. 绘制所有竖线 -->
      <g class="board-lines">
        <line 
          v-for="i in GoBoardVisibleSize" 
          :key="'v' + i"
          :x1="svgPos(i - 1)" 
          :y1="svgPos(0)" 
          :x2="svgPos(i - 1)" 
          :y2="svgPos(GoBoardVisibleSize - 1)"
        />
      </g>
      
      <!-- 4. 绘制星位 -->
      <g class="star-points">
        <circle 
          v-for="(point, index) in starPoints"
          :key="'star' + index"
          :cx="point.x"
          :cy="point.y"
          r="5" 
        />
      </g>

      <!-- 5. 绘制棋子 -->
      <g class="pieces">
        <circle 
          v-for="piece in visiblePieces" 
          :key="`${piece.dataX}-${piece.dataY}`"
          :cx="piece.cx" 
          :cy="piece.cy" 
          r="18" 
          :class="`player-${piece.player-1}`"
        />
      </g>

      <!-- 6. 绘制幽灵棋子 -->
       <g class="ghost-piece" v-if="ghostPiece.visible && !GameOver">
        <circle 
          :cx="ghostPiece.cx" 
          :cy="ghostPiece.cy" 
          r="18" 
          :class="`player-${CurrentPlayer}`"
        />
      </g>

    </svg>
    <div v-if="GameOver" class="victory-overlay">
      <div class="victory-content">
        <p>GameOver</p>
        <h2>Player {{ Winner }} Wins!</h2>
        <button @click="resetGame">Restart Game</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';


// 棋局上限
const DATA_GRID_SIZE = 4096;
// 格子大小
const CELL_SIZE = 40;
// 棋盘边距
const PADDING = 20;
// 棋盘总大小
const boardSize = computed(() => {
  return (GoBoardVisibleSize.value - 1) * CELL_SIZE + PADDING * 2;
});
// 棋盘数据
const boardData = createDataGrid(DATA_GRID_SIZE);
function createDataGrid(size) {
  console.log(`Initializing a huge ${size}x${size} data grid...`);
  return Array(size).fill(0).map(() => Array(size).fill(0));
}
// 棋盘大小
const GoBoardVisibleSize = ref(15);
// 棋子个数
const PieceCount = ref(0);
// 棋手个数
const PlayerCount = ref(2);
// 当前棋手
const CurrentPlayer = ref(0);
// 修改进度
const DataModificationVersion = ref(0);
// 禁手规则
const BalanceBreaker = ref({ 'DoubleThree': false, 'DoubleFour': false, 'Overline': false });
// 获胜玩家
const Winner = ref(null);
// 游戏结束
const GameOver = ref(false);
// 获胜条件
const winPieces = ref(5);
// const board = computed(() => {
//   return Array(GoBoardVisibleSize.value).fill(0).map(() => Array(GoBoardVisibleSize.value).fill(0));
// });
// 棋子数据位置
const visibleBoardDataPos = computed(() => {
  const dataCenterPos = Math.floor(DATA_GRID_SIZE / 2);
  const halfVisibleSize = Math.floor(GoBoardVisibleSize.value / 2);
  const start = dataCenterPos - halfVisibleSize;
  const end = dataCenterPos + halfVisibleSize;
  return { start, end };
});
// 提取棋子数据
const visiblePieces = computed(() => {
  DataModificationVersion.value;
  const pieces = [];
  const { start, end } = visibleBoardDataPos.value;
  for (let y = start; y <= end; y++) {
    for (let x = start; x <= end; x++) {
      if (boardData[y][x] !== 0) {
        pieces.push({
          dataX: x,
          dataY: y,
          cx: svgPos(x - start),
          cy: svgPos(y - start),
          player: boardData[y][x]
        });
      }
    }
  }
  return pieces;
})

// 棋盘星位
const starPoints = computed(() => {
  const center = Math.floor(GoBoardVisibleSize.value / 2); // 7
  const points = [
    { x: center, y: center }, // 天元
    { x: center - 4, y: center - 4 }, // 左上角
    { x: center - 4, y: center + 4 }, // 左下角
    { x: center + 4, y: center - 4 }, // 右上角
    { x: center + 4, y: center + 4 } // 右下角
  ];
  return points.map(p => ({ x: svgPos(p.x), y: svgPos(p.y) }));
});

// 幽灵棋子
const ghostPiece = ref(
  {
    cx: 0,
    cy: 0,
    visible: false
  }
)

// 坐标计算
function svgPos(index) {
  return PADDING + index * CELL_SIZE;
}
// 点击事件处理
function handleClick(event) {
  if (GameOver.value) {
    console.log("Game is over, no more moves allowed.");
    return; // 游戏已结束，不允许再下棋
  }
  const rect = event.currentTarget.getBoundingClientRect();
  // 处理缩放问题
  const scaleX = boardSize.value / rect.width;
  const scaleY = boardSize.value / rect.height;
  const svgX = (event.clientX - rect.left) * scaleX;
  const svgY = (event.clientY - rect.top) * scaleY;
  const visibleX = Math.round((svgX - PADDING) / CELL_SIZE);
  const visibleY = Math.round((svgY - PADDING) / CELL_SIZE);
  if (visibleX < 0 || visibleX >= GoBoardVisibleSize.value || visibleY < 0 || visibleY >= GoBoardVisibleSize.value) {
    return; // 点击在棋盘外
  }
  const dataX = visibleX + visibleBoardDataPos.value.start;
  const dataY = visibleY + visibleBoardDataPos.value.start;
  if (boardData[dataY][dataX] === 0) {
    ghostPiece.value.visible = false;
    boardData[dataY][dataX] = CurrentPlayer.value + 1;
    DataModificationVersion.value++;
    PieceCount.value++;
    CurrentPlayer.value = (CurrentPlayer.value + 1) % PlayerCount.value; // 切换棋手
    checkExpansion();
    checkBalanceBreaker(dataY, dataX);
    checkGameOver(dataY, dataX);
  }
  else {
    return; // 位置已被占用
  }

}
function handleMovement(event) {
  const rect = event.currentTarget.getBoundingClientRect();
  // 处理缩放问题
  const scaleX = boardSize.value / rect.width;
  const scaleY = boardSize.value / rect.height;
  const svgX = (event.clientX - rect.left) * scaleX;
  const svgY = (event.clientY - rect.top) * scaleY;
  const visibleX = Math.round((svgX - PADDING) / CELL_SIZE);
  const visibleY = Math.round((svgY - PADDING) / CELL_SIZE);
  if (visibleX < 0 || visibleX >= GoBoardVisibleSize.value || visibleY < 0 || visibleY >= GoBoardVisibleSize.value) {
    ghostPiece.value.visible = false; // 鼠标移出棋盘区域
    return;
  }
  if (boardData[visibleY + visibleBoardDataPos.value.start][visibleX + visibleBoardDataPos.value.start] !== 0) {
    ghostPiece.value.visible = false; // 鼠标移到已占用位置
    return;
  }
  ghostPiece.value.cx = svgPos(visibleX);
  ghostPiece.value.cy = svgPos(visibleY);
  ghostPiece.value.visible = true;
}
function checkExpansion() {
  const currentAreaSize = GoBoardVisibleSize.value * GoBoardVisibleSize.value;
  const threshold = currentAreaSize * 0.8;
  if (PieceCount.value >= threshold) {
    console.log("Expansion triggered!");
    let newSize = Math.floor(GoBoardVisibleSize.value * 1.5);
    if (newSize % 2 === 0) {
      newSize++; // 确保新大小为奇数
    }
    GoBoardVisibleSize.value = newSize;
    console.log(`Board size expanded to ${newSize}x${newSize}`);
  }
}
function checkBalanceBreaker(Y, X) {
  BalanceBreaker.value.DoubleThree = false;
  console.log(`Checking balance breaker rules for player ${boardData[Y][X]} at (${Y}, ${X})`);
}

function checkGameOver(Y, X) { 
  if (GameOver.value) return;

  const checkDirections = [
    { dx: 1, dy: 0 }, // 横向
    { dx: 0, dy: 1 }, // 竖向
    { dx: 1, dy: 1 }, // 左斜
    { dx: -1, dy: 1 } // 右斜
  ];

  for (const Direction of checkDirections) {
    let cnt = 1;
    for (let step = 1; step < winPieces.value; step++) {
      if (boardData[Y + Direction.dy * step] &&
        boardData[Y + Direction.dy * step][X + Direction.dx * step] === boardData[Y][X]) {
        cnt++;
      } else {
        break;
      }
    }
    for (let step = 1; step < winPieces.value; step++) {
      if (boardData[Y - Direction.dy * step] && 
          boardData[Y - Direction.dy * step][X - Direction.dx * step] === boardData[Y][X]) {
        cnt++;
        } else {
        break;
      }
    }
    if (cnt >= winPieces.value) {
      Winner.value = boardData[Y][X];
      GameOver.value = true;
      console.log(`Player ${Winner.value} wins!`);
      return;
    }
  }
}
function resetGame() {
  console.log("Resetting game...");
  for (let y = 0; y < DATA_GRID_SIZE; y++) {
    for (let x = 0; x < DATA_GRID_SIZE; x++) {
      if (boardData[y][x] !== 0) {
        boardData[y][x] = 0;
      }
    }
  }
  GameOver.value = false;
  Winner.value = null;
  GoBoardVisibleSize.value = 15;
  PieceCount.value = 0;
  CurrentPlayer.value = 0;
  DataModificationVersion.value++;
}

</script>

<style scoped>
.board-container {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-top: 50px;
}

.board-bg {
  fill: #e3c16f; /* 棋盘背景填充色 */
  stroke: #5a3a1a; /* 棋盘边框颜色 */
  stroke-width: 2px; /* 棋盘边框宽度 */
}

.board-lines line {
  stroke: #5a3a1a; /* 线条颜色 */
  stroke-width: 1px; /* 线条宽度 */
}

.star-points circle {
  fill: #5a3a1a; /* 星位填充色 */
}

/* SVG阴影 */
svg {
  box-shadow: 5px 5px 15px rgba(0, 0, 0, 0.5);
  border-radius: 4px; /* 圆角 */
}

.pieces circle {
  stroke-width: 1px;
}

.ghost-piece circle {
  opacity: 0.5;
  pointer-events: none;
}

.player-0 {
  fill: #111;
  stroke: #555;
}

.player-1 {
  fill: #eee;
  stroke: #bbb;
}

/* ... 胜利样式 ... */

.victory-overlay {
  position: absolute; /* 覆盖在棋盘上 */
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.6);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 100; /* 确保在最上层 */
}

.victory-content {
  background-color: white;
  padding: 20px 40px;
  border-radius: 10px;
  text-align: center;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
  transform: scale(0.9);
  animation: pop-in 0.3s forwards;
}

@keyframes pop-in {
  from {
    opacity: 0;
    transform: scale(0.8);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

.victory-content h2 {
  margin-top: 0;
  color: #333;
}

.victory-content p {
  font-size: 1.2em;
  margin: 15px 0;
}

.victory-content button {
  padding: 10px 20px;
  font-size: 1em;
  border: none;
  border-radius: 5px;
  background-color: #42b983; /* Vue 绿色 */
  color: white;
  cursor: pointer;
  transition: background-color 0.2s;
}

.victory-content button:hover {
  background-color: #33a06f;
}

</style>