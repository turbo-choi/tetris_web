<template>
  <div class="min-h-screen bg-gradient-to-br from-purple-900 via-blue-900 to-indigo-900 flex items-center justify-center p-4">
    <div class="bg-white/10 backdrop-blur-lg rounded-2xl shadow-2xl p-6 max-w-4xl w-full">
      <!-- 게임 제목 -->
      <div class="text-center mb-6">
        <h1 class="text-4xl font-bold text-white mb-2">TETRIS</h1>
        <p class="text-gray-300">클래식 테트리스 게임</p>
      </div>

      <div class="flex flex-col lg:flex-row gap-6 items-start justify-center">
        <!-- 게임 보드 -->
        <div class="flex-shrink-0">
          <div class="bg-black/50 p-4 rounded-xl border-2 border-gray-600">
            <div class="grid grid-cols-10 gap-0.5 bg-gray-800 p-2 rounded-lg" style="width: 300px; height: 600px;">
              <div
                v-for="(cell, index) in board.flat()"
                :key="index"
                :class="[
                  'w-7 h-7 border border-gray-700 rounded-sm transition-all duration-150',
                  getCellClass(cell)
                ]"
              ></div>
            </div>
          </div>
        </div>

        <!-- 게임 정보 패널 -->
        <div class="flex-1 space-y-4">
          <!-- 다음 블록 -->
          <div class="bg-white/10 p-4 rounded-xl">
            <h3 class="text-white font-semibold mb-3">다음 블록</h3>
            <div class="bg-black/30 p-3 rounded-lg">
              <div class="grid grid-cols-4 gap-0.5 w-fit mx-auto">
                <div
                  v-for="(cell, index) in getNextPieceDisplay()"
                  :key="index"
                  :class="[
                    'w-6 h-6 rounded-sm',
                    cell ? getPieceColor(nextPiece.type) : 'bg-transparent'
                  ]"
                ></div>
              </div>
            </div>
          </div>

          <!-- 게임 통계 -->
          <div class="bg-white/10 p-4 rounded-xl space-y-3">
            <div class="flex justify-between text-white">
              <span>점수:</span>
              <span class="font-bold">{{ score.toLocaleString() }}</span>
            </div>
            <div class="flex justify-between text-white">
              <span>레벨:</span>
              <span class="font-bold">{{ level }}</span>
            </div>
            <div class="flex justify-between text-white">
              <span>라인:</span>
              <span class="font-bold">{{ lines }}</span>
            </div>
          </div>

          <!-- 게임 컨트롤 -->
          <div class="space-y-3">
            <button
              @click="toggleGame"
              :class="[
                'w-full py-3 px-6 rounded-xl font-semibold text-white transition-all duration-200',
                gameStarted && !gameOver ? 'bg-yellow-600 hover:bg-yellow-700' : 'bg-green-600 hover:bg-green-700'
              ]"
            >
              {{ gameStarted && !gameOver ? (isPaused ? '계속하기' : '일시정지') : '게임 시작' }}
            </button>
            
            <button
              @click="resetGame"
              class="w-full py-3 px-6 rounded-xl font-semibold text-white bg-red-600 hover:bg-red-700 transition-all duration-200"
            >
              다시 시작
            </button>
          </div>

          <!-- 모바일 컨트롤 -->
          <div class="lg:hidden bg-white/10 p-4 rounded-xl">
            <h3 class="text-white font-semibold mb-3">컨트롤</h3>
            <div class="grid grid-cols-3 gap-2">
              <button @click="rotatePiece" class="col-start-2 py-2 px-4 bg-blue-600 hover:bg-blue-700 text-white rounded-lg font-semibold">회전</button>
              <button @click="movePiece(-1, 0)" class="py-2 px-4 bg-gray-600 hover:bg-gray-700 text-white rounded-lg font-semibold">←</button>
              <button @click="movePiece(0, 1)" class="py-2 px-4 bg-gray-600 hover:bg-gray-700 text-white rounded-lg font-semibold">↓</button>
              <button @click="movePiece(1, 0)" class="py-2 px-4 bg-gray-600 hover:bg-gray-700 text-white rounded-lg font-semibold">→</button>
            </div>
          </div>

          <!-- 조작법 안내 -->
          <div class="hidden lg:block bg-white/10 p-4 rounded-xl">
            <h3 class="text-white font-semibold mb-3">조작법</h3>
            <div class="text-gray-300 text-sm space-y-1">
              <div>← → : 좌우 이동</div>
              <div>↓ : 빠른 낙하</div>
              <div>↑ : 회전</div>
              <div>스페이스 : 하드 드롭</div>
              <div>P : 일시정지</div>
            </div>
          </div>
        </div>
      </div>

      <!-- 게임 오버 모달 -->
      <div v-if="gameOver" class="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
        <div class="bg-white p-8 rounded-2xl text-center max-w-sm mx-4">
          <h2 class="text-2xl font-bold text-gray-800 mb-4">게임 오버!</h2>
          <p class="text-gray-600 mb-2">최종 점수: {{ score.toLocaleString() }}</p>
          <p class="text-gray-600 mb-6">클리어한 라인: {{ lines }}</p>
          <button
            @click="resetGame"
            class="w-full py-3 px-6 bg-blue-600 hover:bg-blue-700 text-white rounded-xl font-semibold transition-all duration-200"
          >
            다시 시작
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, computed } from 'vue'

// 게임 상태 타입 정의
interface Position {
  x: number
  y: number
}

interface Piece {
  type: number
  rotation: number
  position: Position
}

// 테트로미노 정의 (7가지 블록)
const PIECES = [
  // I 블록
  [
    [[0,0,0,0],[1,1,1,1],[0,0,0,0],[0,0,0,0]],
    [[0,0,1,0],[0,0,1,0],[0,0,1,0],[0,0,1,0]],
    [[0,0,0,0],[0,0,0,0],[1,1,1,1],[0,0,0,0]],
    [[0,1,0,0],[0,1,0,0],[0,1,0,0],[0,1,0,0]]
  ],
  // O 블록
  [
    [[0,0,0,0],[0,1,1,0],[0,1,1,0],[0,0,0,0]],
    [[0,0,0,0],[0,1,1,0],[0,1,1,0],[0,0,0,0]],
    [[0,0,0,0],[0,1,1,0],[0,1,1,0],[0,0,0,0]],
    [[0,0,0,0],[0,1,1,0],[0,1,1,0],[0,0,0,0]]
  ],
  // T 블록
  [
    [[0,0,0,0],[0,1,0,0],[1,1,1,0],[0,0,0,0]],
    [[0,0,0,0],[0,1,0,0],[0,1,1,0],[0,1,0,0]],
    [[0,0,0,0],[0,0,0,0],[1,1,1,0],[0,1,0,0]],
    [[0,0,0,0],[0,1,0,0],[1,1,0,0],[0,1,0,0]]
  ],
  // S 블록
  [
    [[0,0,0,0],[0,1,1,0],[1,1,0,0],[0,0,0,0]],
    [[0,0,0,0],[0,1,0,0],[0,1,1,0],[0,0,1,0]],
    [[0,0,0,0],[0,0,0,0],[0,1,1,0],[1,1,0,0]],
    [[0,0,0,0],[1,0,0,0],[1,1,0,0],[0,1,0,0]]
  ],
  // Z 블록
  [
    [[0,0,0,0],[1,1,0,0],[0,1,1,0],[0,0,0,0]],
    [[0,0,0,0],[0,0,1,0],[0,1,1,0],[0,1,0,0]],
    [[0,0,0,0],[0,0,0,0],[1,1,0,0],[0,1,1,0]],
    [[0,0,0,0],[0,1,0,0],[1,1,0,0],[1,0,0,0]]
  ],
  // J 블록
  [
    [[0,0,0,0],[1,0,0,0],[1,1,1,0],[0,0,0,0]],
    [[0,0,0,0],[0,1,1,0],[0,1,0,0],[0,1,0,0]],
    [[0,0,0,0],[0,0,0,0],[1,1,1,0],[0,0,1,0]],
    [[0,0,0,0],[0,1,0,0],[0,1,0,0],[1,1,0,0]]
  ],
  // L 블록
  [
    [[0,0,0,0],[0,0,1,0],[1,1,1,0],[0,0,0,0]],
    [[0,0,0,0],[0,1,0,0],[0,1,0,0],[0,1,1,0]],
    [[0,0,0,0],[0,0,0,0],[1,1,1,0],[1,0,0,0]],
    [[0,0,0,0],[1,1,0,0],[0,1,0,0],[0,1,0,0]]
  ]
]

// 블록 색상 정의
const PIECE_COLORS = [
  'bg-cyan-400',    // I
  'bg-yellow-400',  // O
  'bg-purple-400',  // T
  'bg-green-400',   // S
  'bg-red-400',     // Z
  'bg-blue-400',    // J
  'bg-orange-400'   // L
]

// 게임 상태
const board = ref<number[][]>(Array(20).fill(null).map(() => Array(10).fill(0)))
const currentPiece = ref<Piece>({ type: 0, rotation: 0, position: { x: 4, y: 0 } })
const nextPiece = ref<Piece>({ type: 0, rotation: 0, position: { x: 0, y: 0 } })
const score = ref(0)
const level = ref(1)
const lines = ref(0)
const gameOver = ref(false)
const gameStarted = ref(false)
const isPaused = ref(false)

// 게임 루프 관련
let gameInterval: NodeJS.Timeout | null = null
let dropSpeed = 1000 // 밀리초

/**
 * 랜덤한 테트로미노 타입 생성
 */
function generateRandomPiece(): number {
  return Math.floor(Math.random() * 7)
}

/**
 * 새로운 블록 생성
 */
function generateNewPiece(): void {
  currentPiece.value = {
    type: nextPiece.value.type,
    rotation: 0,
    position: { x: 4, y: 0 }
  }
  nextPiece.value.type = generateRandomPiece()
}

/**
 * 블록이 이동 가능한지 체크
 */
function canMove(piece: Piece, dx: number, dy: number, rotation?: number): boolean {
  const newX = piece.position.x + dx
  const newY = piece.position.y + dy
  const pieceRotation = rotation !== undefined ? rotation : piece.rotation
  const pieceShape = PIECES[piece.type][pieceRotation]

  for (let y = 0; y < 4; y++) {
    for (let x = 0; x < 4; x++) {
      if (pieceShape[y][x]) {
        const boardX = newX + x
        const boardY = newY + y

        // 경계 체크
        if (boardX < 0 || boardX >= 10 || boardY >= 20) {
          return false
        }

        // 고정된 블록과 충돌 체크 (fixedBoard 사용)
        if (boardY >= 0 && fixedBoard.value[boardY][boardX] > 0) {
          return false
        }
      }
    }
  }
  return true
}

/**
 * 블록 이동
 */
function movePiece(dx: number, dy: number): void {
  if (!gameStarted.value || gameOver.value || isPaused.value) return

  if (canMove(currentPiece.value, dx, dy)) {
    currentPiece.value.position.x += dx
    currentPiece.value.position.y += dy
    updateBoard()
  } else if (dy > 0) {
    // 아래로 이동할 수 없으면 블록 고정
    placePiece()
  }
}

/**
 * 블록 회전
 */
function rotatePiece(): void {
  if (!gameStarted.value || gameOver.value || isPaused.value) return

  const newRotation = (currentPiece.value.rotation + 1) % 4
  if (canMove(currentPiece.value, 0, 0, newRotation)) {
    currentPiece.value.rotation = newRotation
    updateBoard()
  }
}

/**
 * 하드 드롭 (블록을 바닥까지 즉시 이동)
 */
function hardDrop(): void {
  if (!gameStarted.value || gameOver.value || isPaused.value) return

  while (canMove(currentPiece.value, 0, 1)) {
    currentPiece.value.position.y++
    score.value += 2 // 하드 드롭 보너스
  }
  placePiece()
}

/**
 * 블록을 보드에 고정
 */
function placePiece(): void {
  const piece = currentPiece.value
  const pieceShape = PIECES[piece.type][piece.rotation]

  // 블록을 고정된 보드에 배치
  for (let y = 0; y < 4; y++) {
    for (let x = 0; x < 4; x++) {
      if (pieceShape[y][x]) {
        const boardX = piece.position.x + x
        const boardY = piece.position.y + y
        if (boardY >= 0 && boardY < 20 && boardX >= 0 && boardX < 10) {
          fixedBoard.value[boardY][boardX] = piece.type + 1
        }
      }
    }
  }

  // 라인 클리어 체크
  clearLines()

  // 새로운 블록 생성
  generateNewPiece()

  // 게임 오버 체크
  if (!canMove(currentPiece.value, 0, 0)) {
    gameOver.value = true
    gameStarted.value = false
    if (gameInterval) {
      clearInterval(gameInterval)
      gameInterval = null
    }
  }

  updateBoard()
}

/**
 * 완성된 라인 제거
 */
function clearLines(): void {
  let linesCleared = 0
  
  for (let y = fixedBoard.value.length - 1; y >= 0; y--) {
    if (fixedBoard.value[y].every(cell => cell > 0)) {
      fixedBoard.value.splice(y, 1)
      fixedBoard.value.unshift(Array(10).fill(0))
      linesCleared++
      y++ // 같은 줄을 다시 체크
    }
  }

  if (linesCleared > 0) {
    lines.value += linesCleared
    
    // 점수 계산 (테트리스 표준)
    const lineScores = [0, 40, 100, 300, 1200]
    score.value += lineScores[linesCleared] * level.value

    // 레벨 업 (10라인마다)
    const newLevel = Math.floor(lines.value / 10) + 1
    if (newLevel > level.value) {
      level.value = newLevel
      dropSpeed = Math.max(50, 1000 - (level.value - 1) * 50)
      
      // 게임 루프 재시작 (속도 변경)
      if (gameInterval) {
        clearInterval(gameInterval)
        startGameLoop()
      }
    }
  }
}

// 실제 보드 상태 (고정된 블록만)
const fixedBoard = ref<number[][]>(Array(20).fill(null).map(() => Array(10).fill(0)))

/**
 * 보드 업데이트 (현재 블록 포함)
 */
function updateBoard(): void {
  // 고정된 블록만 복사
  const newBoard = fixedBoard.value.map(row => [...row])
  
  // 현재 블록 그리기
  const piece = currentPiece.value
  const pieceShape = PIECES[piece.type][piece.rotation]
  
  for (let y = 0; y < 4; y++) {
    for (let x = 0; x < 4; x++) {
      if (pieceShape[y][x]) {
        const boardX = piece.position.x + x
        const boardY = piece.position.y + y
        if (boardX >= 0 && boardX < 10 && boardY >= 0 && boardY < 20) {
          newBoard[boardY][boardX] = -(piece.type + 1) // 음수로 현재 블록 표시
        }
      }
    }
  }
  
  board.value = newBoard
}

/**
 * 게임 루프 시작
 */
function startGameLoop(): void {
  if (gameInterval) clearInterval(gameInterval)
  
  gameInterval = setInterval(() => {
    if (!isPaused.value && gameStarted.value && !gameOver.value) {
      movePiece(0, 1)
    }
  }, dropSpeed)
}

/**
 * 게임 시작/일시정지 토글
 */
function toggleGame(): void {
  if (gameOver.value) {
    resetGame()
    return
  }

  if (!gameStarted.value) {
    // 게임 시작
    gameStarted.value = true
    isPaused.value = false
    currentPiece.value.type = generateRandomPiece()
    nextPiece.value.type = generateRandomPiece()
    generateNewPiece()
    updateBoard()
    startGameLoop()
  } else {
    // 일시정지/재개
    isPaused.value = !isPaused.value
  }
}

/**
 * 게임 리셋
 */
function resetGame(): void {
  if (gameInterval) {
    clearInterval(gameInterval)
    gameInterval = null
  }
  
  board.value = Array(20).fill(null).map(() => Array(10).fill(0))
  fixedBoard.value = Array(20).fill(null).map(() => Array(10).fill(0))
  score.value = 0
  level.value = 1
  lines.value = 0
  gameOver.value = false
  gameStarted.value = false
  isPaused.value = false
  dropSpeed = 1000
  
  currentPiece.value = { type: 0, rotation: 0, position: { x: 4, y: 0 } }
  nextPiece.value = { type: 0, rotation: 0, position: { x: 0, y: 0 } }
}

/**
 * 셀 클래스 반환
 */
function getCellClass(cell: number): string {
  if (cell === 0) return 'bg-gray-900'
  if (cell < 0) return PIECE_COLORS[Math.abs(cell) - 1] + ' shadow-lg' // 현재 블록
  return PIECE_COLORS[cell - 1] + ' shadow-md' // 고정된 블록
}

/**
 * 블록 색상 반환
 */
function getPieceColor(type: number): string {
  return PIECE_COLORS[type]
}

/**
 * 다음 블록 표시용 배열 생성
 */
function getNextPieceDisplay(): boolean[] {
  const shape = PIECES[nextPiece.value.type][0] // 기본 회전 상태
  return shape.flat().map(cell => cell === 1)
}

/**
 * 키보드 이벤트 핸들러
 */
function handleKeyPress(event: KeyboardEvent): void {
  if (!gameStarted.value || gameOver.value) return

  switch (event.key) {
    case 'ArrowLeft':
      event.preventDefault()
      movePiece(-1, 0)
      break
    case 'ArrowRight':
      event.preventDefault()
      movePiece(1, 0)
      break
    case 'ArrowDown':
      event.preventDefault()
      movePiece(0, 1)
      break
    case 'ArrowUp':
      event.preventDefault()
      rotatePiece()
      break
    case ' ':
      event.preventDefault()
      hardDrop()
      break
    case 'p':
    case 'P':
      event.preventDefault()
      if (gameStarted.value && !gameOver.value) {
        isPaused.value = !isPaused.value
      }
      break
  }
}

// 컴포넌트 마운트/언마운트 시 이벤트 리스너 등록/해제
onMounted(() => {
  window.addEventListener('keydown', handleKeyPress)
  // 초기 블록 설정
  nextPiece.value.type = generateRandomPiece()
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyPress)
  if (gameInterval) {
    clearInterval(gameInterval)
  }
})
</script>