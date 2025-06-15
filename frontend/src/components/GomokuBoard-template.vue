<template>
  <div class="board-container">
    <svg 
      :width="boardSize" 
      :height="boardSize" 
      :viewBox="`0 0 ${boardSize} ${boardSize}`"
      @mousemove="handleMouseMove"
      @mouseleave="handleMouseLeave"
      @click="handleBoardClick"
    >
      <!-- 1. 棋盘背景 -->
      <rect :width="boardSize" :height="boardSize" class="board-bg" />

      <!-- 2. 绘制棋盘网格线 -->
      <g class="grid-lines">
        <!-- 横线 -->
        <line
          v-for="i in gridSize"
          :key="'h' + i"
          :x1="padding"
          :y1="padding + (i - 1) * cellSize"
          :x2="boardSize - padding"
          :y2="padding + (i - 1) * cellSize"
        />
        <!-- 竖线 -->
        <line
          v-for="i in gridSize"
          :key="'v' + i"
          :x1="padding + (i - 1) * cellSize"
          :y1="padding"
          :x2="padding + (i - 1) * cellSize"
          :y2="boardSize - padding"
        />
      </g>
      
      <!-- 3. 绘制星位 (天元和另外8个关键点) -->
      <g class="star-points">
        <circle 
          v-for="(point, index) in starPoints" 
          :key="'star' + index"
          :cx="point.x" 
          :cy="point.y" 
          r="4" 
        />
      </g>

      <!-- 4. 绘制棋子 -->
      <g class="stones">
        <circle
          v-for="(stone, index) in placedStones"
          :key="'stone' + index"
          :cx="getCoord(stone.x)"
          :cy="getCoord(stone.y)"
          :r="stoneRadius"
          :class="stone.player === 1 ? 'black-stone' : 'white-stone'"
        />
      </g>

      <!-- 5. 鼠标悬停时的 "幽灵" 棋子，提升用户体验 -->
      <circle
        v-if="ghostStone.visible"
        :cx="getCoord(ghostStone.x)"
        :cy="getCoord(ghostStone.y)"
        :r="stoneRadius"
        class="ghost-stone"
        :class="currentPlayer === 1 ? 'black-stone' : 'white-stone'"
      />
    </svg>
  </div>
</template>

<script setup>
import { ref, reactive, computed } from 'vue';

// --- 棋盘配置 ---
const gridSize = 15; // 15x15的棋盘
const cellSize = 40; // 每个格子的像素大小
const padding = 20; // 棋盘边缘的内边距
const boardSize = computed(() => (gridSize - 1) * cellSize + padding * 2); // SVG视窗总大小
const stoneRadius = computed(() => cellSize * 0.45); // 棋子半径

// --- 棋盘状态 ---
// 创建一个15x15的二维数组，0: 空, 1: 黑子, 2: 白子
const boardState = ref(Array(gridSize).fill(0).map(() => Array(gridSize).fill(0)));
const currentPlayer = ref(1); // 1: 黑方, 2: 白方

// --- 交互状态 ---
const ghostStone = reactive({
  visible: false,
  x: 0,
  y: 0,
});

// --- 计算属性 ---
// 星位坐标 (天元, 角上4个, 边上4个)
const starPoints = computed(() => {
  const center = Math.floor(gridSize / 2); // 7
  const points = [
    { x: center, y: center }, // 天元
    { x: center - 4, y: center - 4 }, // 左上角
    { x: center - 4, y: center + 4 }, // 左下角
    { x: center + 4, y: center - 4 }, // 右上角
    { x: center + 4, y: center + 4 } // 右下角
  ];
  return points.map(p => ({ x: getCoord(p.x), y: getCoord(p.y) }));
});
// 将二维数组的棋盘状态转换为一维数组，方便v-for渲染
const placedStones = computed(() => {
  const stones = [];
  for (let y = 0; y < gridSize; y++) {
    for (let x = 0; x < gridSize; x++) {
      if (boardState.value[y][x] !== 0) {
        stones.push({ x, y, player: boardState.value[y][x] });
      }
    }
  }
  return stones;
});

// --- 工具函数 ---
// 将网格坐标(0-14)转换为SVG像素坐标
const getCoord = (gridIndex) => {
  return padding + gridIndex * cellSize;
};

console.log(starPoints.value);
// --- 事件处理 ---
const handleMouseMove = (event) => {
  // 从事件中获取相对于SVG左上角的坐标
  const rect = event.currentTarget.getBoundingClientRect();
  const x = event.clientX - rect.left;
  const y = event.clientY - rect.top;

  // 将像素坐标转换为最近的网格坐标
  const gridX = Math.round((x - padding) / cellSize);
  const gridY = Math.round((y - padding) / cellSize);
  
  // 检查坐标是否在棋盘内且该位置为空
  if (gridX >= 0 && gridX < gridSize && gridY >= 0 && gridY < gridSize) {
    ghostStone.visible = true;
    ghostStone.x = gridX;
    ghostStone.y = gridY;
  } else {
    ghostStone.visible = false;
  }
};

const handleMouseLeave = () => {
  ghostStone.visible = false;
};

const handleBoardClick = () => {
  if (!ghostStone.visible) return; // 如果鼠标在棋盘外，不处理点击

  const { x, y } = ghostStone;
  
  // 如果该位置已经有棋子，则不处理
  if (boardState.value[y][x] !== 0) {
    console.log('此位置已有棋子!');
    return;
  }

  // 更新棋盘状态
  boardState.value[y][x] = currentPlayer.value;

  // 切换玩家
  currentPlayer.value = currentPlayer.value === 1 ? 2 : 1;
};

</script>

<style scoped>
.board-container {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
}

.board-bg {
  fill: #e3c16f; /* 经典棋盘木色 */
}

.grid-lines line {
  stroke: #000;
  stroke-width: 1;
}

.star-points circle {
  fill: #000;
}

.black-stone {
  fill: #000;
}

.white-stone {
  fill: #fff;
  stroke: #bbb; /* 给白棋一个描边，使其在背景上更清晰 */
  stroke-width: 1;
}

.ghost-stone {
  opacity: 0.5;
  pointer-events: none; /* 让幽灵棋子不影响鼠标事件 */
}
</style>