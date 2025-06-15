<template>
  <div class="board-container">
    <svg 
      :width="boardSize" 
      :height="boardSize" 
      :viewBox="`0 0 ${boardSize} ${boardSize}`"
      xmlns="http://www.w3.org/2000/svg"
      @click="handleClick"
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
          :class="`player-${piece.player}`"
        />
      </g>

    </svg>
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
// 线条计算
function svgPos(index) {
  return PADDING + index * CELL_SIZE;
}
// 点击事件处理
function handleClick(event) {
  if (GameOver.value) {
    console.log("Game is over, no more moves allowed.");
    return; // 游戏已结束，不允许再下棋
  }
  console.log("Current player:", CurrentPlayer.value);
  const rect = event.currentTarget.getBoundingClientRect();
  console.log("SVG rect:", rect);
  const visibleX = Math.round((event.clientX - rect.left - PADDING) / CELL_SIZE);
  const visibleY = Math.round((event.clientY - rect.top - PADDING) / CELL_SIZE);
  console.log(`Clicked at (${visibleX}, ${visibleY}) relative to SVG`);
  if (visibleX < 0 || visibleX >= GoBoardVisibleSize.value || visibleY < 0 || visibleY >= GoBoardVisibleSize.value) {
    return; // 点击在棋盘外
  }
  const dataX = visibleX + visibleBoardDataPos.value.start;
  const dataY = visibleY + visibleBoardDataPos.value.start;
  if (boardData[dataY][dataX] === 0) {
    boardData[dataY][dataX] = CurrentPlayer.value + 1;
    DataModificationVersion.value++;
    PieceCount.value++;
    CurrentPlayer.value = (CurrentPlayer.value + 1) % PlayerCount.value; // 切换棋手
    checkExpansion();
    checkBalanceBreaker(dataY, dataX);
    checkGameOver(dataY, dataX);
  }
  else {
    console.log(`Position (${dataX}, ${dataY}) already occupied by player ${boardData[dataY][dataX]}`);
    return; // 位置已被占用
  }
  console.log(`Clicked at (${visibleX}, ${visibleY}) - Data position: (${dataX}, ${dataY})`, "Data", boardData[dataY][dataX]);

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
  console.log(`Checking game over conditions for player ${boardData[Y][X]} at (${Y}, ${X})`);
  if ( boardData[Y][X] === boardData[Y][X - 1] && // 横向五连中
       boardData[Y][X] === boardData[Y][X + 1] && 
       boardData[Y][X] === boardData[Y][X - 2] && 
    boardData[Y][X] === boardData[Y][X + 2] || 
       boardData[Y][X] === boardData[Y - 1][X] && // 竖向五连中
       boardData[Y][X] === boardData[Y + 1][X] && 
       boardData[Y][X] === boardData[Y - 2][X] &&
       boardData[Y][X] === boardData[Y + 2][X] || 
       boardData[Y][X] === boardData[Y - 1][X - 1] && // 左斜五连中
       boardData[Y][X] === boardData[Y + 1][X + 1] &&
       boardData[Y][X] === boardData[Y - 2][X - 2] &&
       boardData[Y][X] === boardData[Y + 2][X + 2] || 
    boardData[Y][X] === boardData[Y - 1][X + 1] && // 右斜五连中
    boardData[Y][X] === boardData[Y + 1][X - 1] &&
       boardData[Y][X] === boardData[Y - 2][X + 2] &&
    boardData[Y][X] === boardData[Y + 2][X - 2] || 
    boardData[Y][X] === boardData[Y][X - 1] && // 横向五连右1
      boardData[Y][X] === boardData[Y][X + 1] &&
      boardData[Y][X] === boardData[Y][X - 2] &&
    boardData[Y][X] === boardData[Y][X - 3] || 
    boardData[Y][X] === boardData[Y][X - 1] && // 横向五连左1
      boardData[Y][X] === boardData[Y][X + 1] &&
      boardData[Y][X] === boardData[Y][X + 2] &&
    boardData[Y][X] === boardData[Y][X + 3] || 
    boardData[Y][X] === boardData[Y - 1][X] && // 竖向五连下1
    boardData[Y][X] === boardData[Y + 1][X] &&
    boardData[Y][X] === boardData[Y + 2][X] &&
    boardData[Y][X] === boardData[Y + 3][X] ||
    boardData[Y][X] === boardData[Y - 1][X] && // 竖向五连上1
      boardData[Y][X] === boardData[Y + 1][X] &&
  boardData[Y][X] === boardData[Y - 2][X] &&
    boardData[Y][X] === boardData[Y - 3][X] || 
    boardData[Y][X] === boardData[Y - 1][X - 1] && // 左斜五连右1
    boardData[Y][X] === boardData[Y + 1][X + 1] &&
    boardData[Y][X] === boardData[Y - 2][X - 2] &&
    boardData[Y][X] === boardData[Y - 3][X - 3] ||
    boardData[Y][X] === boardData[Y - 1][X - 1] && // 左斜五连左1
    boardData[Y][X] === boardData[Y + 1][X + 1] &&
    boardData[Y][X] === boardData[Y + 2][X + 2] &&
    boardData[Y][X] === boardData[Y + 3][X + 3] ||
    boardData[Y][X] === boardData[Y - 1][X + 1] && // 右斜五连右1
    boardData[Y][X] === boardData[Y + 1][X - 1] &&
    boardData[Y][X] === boardData[Y - 2][X + 2] &&
    boardData[Y][X] === boardData[Y - 3][X + 3] ||
    boardData[Y][X] === boardData[Y - 1][X + 1] && // 右斜五连左1
    boardData[Y][X] === boardData[Y + 1][X - 1] &&
    boardData[Y][X] === boardData[Y + 2][X - 2] &&
    boardData[Y][X] === boardData[Y + 3][X - 3] ||
    boardData[Y][X] === boardData[Y][X - 1] && // 横向五连右2
    boardData[Y][X] === boardData[Y][X - 2] &&
  boardData[Y][X] === boardData[Y][X - 3] &&
    boardData[Y][X] === boardData[Y][X - 4] ||
    boardData[Y][X] === boardData[Y][X + 1] && // 横向五连左2
    boardData[Y][X] === boardData[Y][X + 2] &&
    boardData[Y][X] === boardData[Y][X + 3] &&
    boardData[Y][X] === boardData[Y][X + 4] ||
    boardData[Y][X] === boardData[Y + 1][X] && // 竖向五连下2
    boardData[Y][X] === boardData[Y + 2][X] &&
    boardData[Y][X] === boardData[Y + 3][X] &&
    boardData[Y][X] === boardData[Y + 4][X] ||
    boardData[Y][X] === boardData[Y - 1][X] && // 竖向五连上2
    boardData[Y][X] === boardData[Y - 2][X] &&
    boardData[Y][X] === boardData[Y - 3][X] &&
    boardData[Y][X] === boardData[Y - 4][X] ||
    boardData[Y][X] === boardData[Y - 1][X - 1] && // 左斜五连右2
    boardData[Y][X] === boardData[Y - 2][X - 2] &&
    boardData[Y][X] === boardData[Y - 3][X - 3] &&
    boardData[Y][X] === boardData[Y - 4][X - 4] ||
    boardData[Y][X] === boardData[Y + 1][X + 1] && // 左斜五连左2
    boardData[Y][X] === boardData[Y + 2][X + 2] &&
    boardData[Y][X] === boardData[Y + 3][X + 3] &&
    boardData[Y][X] === boardData[Y + 4][X + 4] ||
    boardData[Y][X] === boardData[Y - 1][X + 1] && // 右斜五连右2
    boardData[Y][X] === boardData[Y - 2][X + 2] &&
    boardData[Y][X] === boardData[Y - 3][X + 3] &&
    boardData[Y][X] === boardData[Y - 4][X + 4] ||
    boardData[Y][X] === boardData[Y + 1][X - 1] && // 右斜五连左2
    boardData[Y][X] === boardData[Y + 2][X - 2] &&
    boardData[Y][X] === boardData[Y + 3][X - 3] &&
    boardData[Y][X] === boardData[Y + 4][X - 4]
) {
        Winner.value = boardData[Y][X];
        GameOver.value = true;
        console.log(`Player ${Winner.value} wins!`);
    return; // 检查到胜利条件，游戏结束
  }
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

.player-1 {
  fill: #111;
  stroke: #555;
}

.player-2 {
  fill: #eee;
  stroke: #bbb;
}

</style>