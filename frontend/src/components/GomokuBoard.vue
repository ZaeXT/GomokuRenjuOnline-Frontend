<template>
  <div class="board-container">
    <svg 
      :width="boardSize" 
      :height="boardSize" 
      :viewBox="`0 0 ${boardSize} ${boardSize}`"
      xmlns="http://www.w3.org/2000/svg"
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
          v-for="i in GoBoardSize" 
          :key="'h' + i"
          :x1="PADDING" 
          :y1="LinePos(i - 1)" 
          :x2="boardSize - PADDING" 
          :y2="LinePos(i - 1)"
        />
      </g>

      <!-- 3. 绘制所有竖线 -->
      <g class="board-lines">
        <line 
          v-for="i in GoBoardSize" 
          :key="'v' + i"
          :x1="LinePos(i - 1)" 
          :y1="PADDING" 
          :x2="LinePos(i - 1)" 
          :y2="boardSize - PADDING"
        />
      </g>
      
      <!-- 4. 绘制星位 -->
      <g class="star-points">
        <circle 
          v-for="(point, index) in starPoints"
          :key="'star' + index"
          :cx="point.x"
          :cy="point.y"
          r="4" 
        />
      </g>

      <!-- 5. 之后棋子也会被渲染在这里 -->

    </svg>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';


//棋盘大小
const GoBoardSize = ref(25);
//格子大小
const CELL_SIZE = 40;
//棋盘边距
const PADDING = 20;
//棋盘总大小
const boardSize = computed(() => {
  return (GoBoardSize.value - 1) * CELL_SIZE + PADDING * 2;
});
//棋盘数据
const board = computed(() => {
  return Array(GoBoardSize.value).fill(0).map(() => Array(GoBoardSize.value).fill(0));
});
//棋盘星位
const starPoints = computed(() => {
  const center = Math.floor(GoBoardSize.value / 2); // 7
  const points = [
    { x: center, y: center }, // 天元
    { x: center - 4, y: center - 4 }, // 左上角
    { x: center - 4, y: center + 4 }, // 左下角
    { x: center + 4, y: center - 4 }, // 右上角
    { x: center + 4, y: center + 4 } // 右下角
  ];
  return points.map(p => ({ x: LinePos(p.x), y: LinePos(p.y) }));
});
//线条计算
const LinePos = (index) => PADDING + index * CELL_SIZE;
window.changeBoardSize = (newSize) => {
  GoBoardSize.value = newSize;
  console.log('Board size changed to:', newSize);
  console.log('New SVG size:', boardSize.value);
  console.log('New board model:', board.value);
};

console.log(board.value);

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

</style>