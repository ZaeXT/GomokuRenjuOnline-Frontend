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

      <!-- 5. 渲染棋子 -->
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
    <!-- <div class="info">
      <p>当前棋手: {{ CurrentPlayer }}</p>
      <p>棋子总数: {{ PieceCount }}</p>
      <p>棋盘大小: {{ GoBoardVisibleSize }}x{{ GoBoardVisibleSize }}</p>
      <p>数据修改版本: {{ DataModificationVersion }}</p>
      <p>棋手数量: {{ PlayerCount }}</p>
      <p>棋盘数据位置: {{ visibleBoardDataPos.start }} - {{ visibleBoardDataPos.end }}</p>
      <p>棋盘数据大小: {{ DATA_GRID_SIZE }}x{{ DATA_GRID_SIZE }}</p>
      <p>棋盘总大小: {{ boardSize }}px</p>
      <p>棋盘格子大小: {{ CELL_SIZE }}px</p>
      <p>棋盘边距: {{ PADDING }}px</p>
      <p>棋盘星位: {{ starPoints.map(p => `(${p.x}, ${p.y})`).join(', ') }}</p>
      <p>棋盘可见棋子: {{ visiblePieces.length }}</p>
      <p>棋盘可见棋子位置: {{ visiblePieces.map(p => `(${p.dataX}, ${p.dataY})`).join(', ') }}</p>
      <p>棋盘可见棋子坐标: {{ visiblePieces.map(p => `(${p.cx}, ${p.cy})`).join(', ') }}</p>
      <p>棋盘可见棋子玩家: {{ visiblePieces.map(p => p.player).join(', ') }}</p>
      <p>棋盘可见棋子数据: {{ visiblePieces }}</p>
      <p>棋盘可见棋子数据位置: {{ visibleBoardDataPos }}</p>
      <p>棋盘可见棋子数据大小: {{ GoBoardVisibleSize * GoBoardVisibleSize }}</p>
      <p>棋盘可见棋子数据修改版本: {{ DataModificationVersion }}</p>
      <p>棋盘可见棋子数据修改次数: {{ PieceCount }}</p>
      <p>棋盘可见棋子数据修改进度: {{ (PieceCount / (GoBoardVisibleSize * GoBoardVisibleSize) * 100).toFixed(2) }}%</p>
      <p>棋盘可见棋子数据修改阈值: {{ (GoBoardVisibleSize * GoBoardVisibleSize * 0.8).toFixed(2) }}</p>
      <p>棋盘可见棋子数据修改触发: {{ PieceCount >= (GoBoardVisibleSize * GoBoardVisibleSize * 0.8) ? '是' : '否' }}</p>
      <p>棋盘可见棋子数据修改后棋盘大小: {{ GoBoardVisibleSize }}</p>
      <p>棋盘可见棋子数据修改后棋盘大小: {{ GoBoardVisibleSize * GoBoardVisibleSize }}</p>
      <p>棋盘可见棋子数据修改后棋盘大小: {{ GoBoardVisibleSize * CELL_SIZE + PADDING * 2 }}px</p>
    </div> -->
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

/* 给SVG加个阴影，看起来更有立体感 */
svg {
  box-shadow: 5px 5px 15px rgba(0, 0, 0, 0.5);
  border-radius: 4px; /* 轻微的圆角让阴影更好看 */
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

.game-info {
  margin-top: 20px;
  padding: 10px;
  background-color: #f0f0f0;
  border-radius: 8px;
  box-shadow: inset 0 0 5px rgba(0,0,0,0.1);
}

.black-piece-text {
  font-weight: bold;
  color: black;
}
.white-piece-text {
  font-weight: bold;
  color: #555;
}

</style>