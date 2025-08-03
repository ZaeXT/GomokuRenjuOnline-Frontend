<template>
   <!-- Lobby View -->
  <div v-if="currentView === 'lobby'" class="lobby-container">
    <h1>五子棋游戏大厅</h1>
    <div class="room-creator">
      <input v-model="newRoomName" placeholder="输入新房间名称" @keyup.enter="createRoom" />
      <button @click="createRoom" :disabled="!newRoomName">创建并加入房间</button>
    </div>
    <h2>可加入房间</h2>
    <div v-if="rooms.length === 0" class="no-rooms">目前没有可加入的房间，创建一个吧！</div>
    <ul class="room-list">
      <li v-for="room in rooms" :key="room.id">
        <span>{{ room.name }} ({{ room.playerCount }}/2)</span>
        <button @click="joinRoom(room.id)">加入</button>
      </li>
    </ul>
  </div>
  <!-- Game View -->
  <div v-else-if="currentView === 'game'">
    <div class="main-container">
      <div class="game-status">
        <div v-if="serverState.isGameOver" class="game-overstatus">
          <p>
            游戏结束！ {{serverState.winner ? `玩家 ${serverState.winner} 胜利！` : '平局！'}}
          </p>
          <button @click="playAgain" class="play-again-btn">重新开始游戏</button>
        </div>
        <template v-else>
          <p v-if="!isConnected">连接中...</p>
          <p v-else-if="isMyTurn">
            <span class="your-turn">●</span> 你的回合
          </p>
          <p v-else>
            <span class="opponent-turn">○</span> 等待对手...
          </p>
        </template>
        <p class="player-info">(你是玩家 {{ serverState.yourPlayerId }} 当前房间： {{ serverState.roomName }})</p>
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
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted} from 'vue';

const currentView = ref('lobby'); // 'lobby' or 'game'
const rooms = ref([]);
const newRoomName = ref('');

const socket = ref(null);
const isConnected = ref(false);

const serverState = ref({
  roomName: '',
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
  socket.value = new WebSocket("ws://localhost:8089/ws");
  socket.value.onopen = () => {
    isConnected.value = true;
    console.log("WebSocket connected. In lobby.");
  };
  socket.value.onmessage = (event) => {
    const message = JSON.parse(event.data);
    console.log("Received:", message);
    switch (message.type) {
      case "ROOM_LIST_UPDATE":
        rooms.value = message.payload.rooms;
        break;
      case "GAME_STATE_UPDATE":
        if (currentView.value !== 'game') {
          currentView.value = 'game';
        }
        serverState.value = message.payload;
        break;
      case "ERROR":
        alert(`Error: ${message.payload.message}`);
        break;
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
  }
}

// Lobby
function createRoom() {
  if (newRoomName.value.trim()) {
    sendMessage("CREATE_ROOM", { name: newRoomName.value.trim() });
    newRoomName.value = '';
  }
}

function joinRoom(roomId) {
  sendMessage("JOIN_ROOM", { id: roomId });
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



function playAgain() {
  currentView.value = 'lobby';
}

</script>

<style scoped>
/* --- 新增 Lobby 样式 --- */
.lobby-container {
  max-width: 600px;
  margin: 40px auto;
  padding: 20px;
  background-color: #f9f9f9;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  text-align: center;
}

.room-creator {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.room-creator input {
  flex-grow: 1;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.room-creator button, .room-list button {
  padding: 10px 15px;
  border: none;
  background-color: #42b983;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s;
}
.room-creator button:disabled {
    background-color: #aaa;
    cursor: not-allowed;
}
.room-creator button:hover:not(:disabled) {
  background-color: #33a06f;
}

.room-list {
  list-style-type: none;
  padding: 0;
}

.room-list li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px;
  border-bottom: 1px solid #eee;
}
.room-list li:last-child {
  border-bottom: none;
}


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

.game-over-status {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px; /* 在文字和按钮之间增加间距 */
}

.play-again-btn {
  padding: 8px 16px;
  font-size: 0.9em;
  border: none;
  border-radius: 5px;
  background-color: #007bff; /* 蓝色，与游戏中的绿色区分 */
  color: white;
  cursor: pointer;
  transition: background-color 0.2s;
}

.play-again-btn:hover {
  background-color: #0056b3;
}
</style>