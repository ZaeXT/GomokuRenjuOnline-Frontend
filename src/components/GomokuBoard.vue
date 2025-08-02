<template>
  <div class="main-container">
    <div class="game-status">
      <p v-if="!isConnected">Connecting to server...</p>
      <p v-else-if="serverState.isGameOver">
        Game Over! {{serverState.winner ? `Player ${serverState.winner} Wins!` : 'Its a Draw!'}}
      </p>
      <p v-else-if="isMyTurn">
        <span class="your-turn">●</span> Your Turn
      </p>
      <p v-else>
        <span class="opponent-turn">○</span> Waiting for Opponent...
      </p>
      <p class="player-info">(You are Player {{ serverState.yourPlayerId }})</p>
    </div>  
      <div class="board-container">
      <svg 
        :viewBox="`0 0 ${boardSize} ${boardSize}`"
        xmlns="http://www.w3.org/2000/svg"
        preserveAspectRatio="xMidYMid meet"
        @click="handleClick"
        @mousemove="handleMovement"
        @mouseleave="ghostPiece.visible = false"
        @mouseenter="ghostPiece.visible = true;handleMovement($event)"
        :class="{ 'my-turn-cursor': isMyTurn && !serverState.isGameOver }"
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
            :key="`${piece.x}-${piece.y}`"
            :cx="piece.cx" 
            :cy="piece.cy" 
            r="18" 
            :class="`player-${piece.player-1}`"
          />
        </g>

        <!-- 6. 绘制幽灵棋子 -->
        <g class="ghost-piece" v-if="ghostPiece.visible">
          <circle 
            :cx="ghostPiece.cx" 
            :cy="ghostPiece.cy" 
            r="18" 
            :class="`player-${serverState.currentPlayer - 1}`"
          />
        </g>

      </svg>
      <div v-if="serverState.isGameOver" class="victory-overlay">
        <div class="victory-content">
          <p>GameOver</p>
          <h2>Player {{ serverState.winner ? `Player ${serverState.winner} Wins!` : 'Draw!' }}</h2>
          <button @click="resetGame">Restart Game</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted} from 'vue';

const socket = ref(null);
const isConnected = ref(false);

const serverState = ref({
  visibleSize: 15,
  pieces: [],
  currentPlayer: 1,
  isGameOver: false,
  winner: null,
  yourPlayerId: null,
});



const CELL_SIZE = 40;
const DATA_CENTER_POS = 2048;
const PADDING = 20;

const GoBoardVisibleSize = computed(() =>
serverState.value.visibleSize || 15);
const boardSize = computed(() => {
  return (GoBoardVisibleSize.value - 1) * CELL_SIZE + PADDING * 2;
});
const isMyTurn = computed(() => isConnected.value && serverState.value.currentPlayer === serverState.value.yourPlayerId);
// 提取棋子数据
const visiblePieces = computed(() => {
  return serverState.value.pieces.map(p => ({
    ...p,
    cx: svgPos(toSvgCoord(p.x)),
    cy: svgPos(toSvgCoord(p.y)),
  }));
});

const occupiedPositionsSet = computed(() => {
  return new Set(
    serverState.value.pieces.map(p => `${p.x},${p.y}`)
  );
});

// 棋盘星位
const starPoints = computed(() => {
  const center = Math.floor(GoBoardVisibleSize.value / 2);
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
onMounted(() => {
  connectWebSocket();
});

function connectWebSocket() {
  socket.value = new WebSocket("ws://192.168.1.8:8080/ws");
  socket.value.onopen = () => {
    console.log("WebSocket connection established.");
    isConnected.value = true;
    sendMessage("JOIN_ROOM",{roomID:"room1"});
  };
  socket.value.onmessage = (event) => {
    const message = JSON.parse(event.data);
    console.log("Received:", message);
    if (message.type === "GAME_STATE_UPDATE") {
      serverState.value = message.payload;
    } else if (message.type === "ERROR") {
      alert(`Error: ${message.payload.message}`);
    }
  };

  socket.value.onclose = () => {
    console.log("WebSocket connection closed.");
    isConnected.value = false;
  };
  socket.value.onerror = (error) => {
    console.error("WebSocket error:", error);
    isConnected.value = false;
  };
}

function sendMessage(type, payload) {
  if (socket.value && socket.value.readyState === WebSocket.OPEN) {
    socket.value.send(JSON.stringify({ type, payload }));
  } else {
    console.error("WebSocket is not open. Cannot send message.");
  }
}

// 坐标计算
function svgPos(index) {
  return PADDING + index * CELL_SIZE;
}
function toSvgCoord(absoluteCoord){
  const halfVisible = Math.floor(GoBoardVisibleSize.value / 2);
  const visibleStart = DATA_CENTER_POS - halfVisible;
  return absoluteCoord - visibleStart;
}
// 点击事件处理
function handleClick(event) {
  if (!isMyTurn.value || serverState.value.isGameOver) return;
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
  const halfVisible = Math.floor(GoBoardVisibleSize.value / 2);
  const dataX = (DATA_CENTER_POS - halfVisible) + visibleX;
  const dataY = (DATA_CENTER_POS - halfVisible) + visibleY;
  if (occupiedPositionsSet.value.has(`${dataX},${dataY}`)) {
    return; // 该位置已被占用
  }
  console.log(`Sending MAKE_MOVE for absolute coords:(${dataX}, ${dataY})`);
  sendMessage("MAKE_MOVE", { x: dataX, y: dataY });
  ghostPiece.value.visible = false; // 点击后隐藏幽灵棋子
}



function handleMovement(event) {
  if (!isMyTurn.value || serverState.value.isGameOver) {
    ghostPiece.value.visible = false; // 非我方回合或游戏结束时隐藏幽灵棋子
    return;
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
    ghostPiece.value.visible = false; // 鼠标移出棋盘区域
    return;
  }
  const halfVisible = Math.floor(GoBoardVisibleSize.value / 2);
  const dataX = (DATA_CENTER_POS - halfVisible) + visibleX;
  const dataY = (DATA_CENTER_POS - halfVisible) + visibleY;
  const positionKey = `${dataX},${dataY}`;
  if (occupiedPositionsSet.value.has(positionKey)) {
    ghostPiece.value.visible = false; // 鼠标移到已占用位置
    return;
  }
  ghostPiece.value.cx = svgPos(visibleX);
  ghostPiece.value.cy = svgPos(visibleY);
  ghostPiece.value.visible = true;
}



function resetGame() {
  sendMessage("RESTART_GAME", {});
}

</script>

<style scoped>
.main-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
  font-family: sans-serif;
}

.game-status {
  text-align: center;
  font-size: 1.5em;
  font-weight: bold;
  color: #333;
  padding: 10px 20px;
  background-color: #f0f0f0;
  border-radius: 8px;
  min-width: 300px;
}
.game-status p {
  margin: 0;
  line-height: 1.2;
}
.game-status .player-info {
  font-size: 0.6em;
  font-weight: normal;
  color: #666;
  margin-top: 4px;
}
.your-turn { color: #42b983; }
.opponent-turn { color: #f0ad4e; }

.board-container {
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
  width: 90vw; /* 占据视口宽度的 90% */
  max-width: 800px; /* 但最大不超过 800px */
  margin-left: auto;
  margin-right: auto;
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
  /* 确保: SVG 宽度充满其容器 */
  width: 100%; 
  
  /* 保持: 强制 SVG 的盒子模型为1:1 */
  aspect-ratio: 1 / 1; 

  box-shadow: 5px 5px 15px rgba(0, 0, 0, 0.5);
  border-radius: 4px; /* 圆角 */
}

svg.my-turn-cursor {
  cursor: pointer;
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