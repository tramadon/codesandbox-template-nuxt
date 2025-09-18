<template>
  <div class="tetris-shell">
    <div class="tetris-background"></div>
    <div class="tetris-card">
      <header class="tetris-header">
        <h1>Tetris</h1>
        <p class="subtitle">Stack the neon blocks and chase a personal best.</p>
      </header>
      <div class="tetris-layout">
        <aside class="sidebar">
          <section class="stat-block">
            <h2>Score</h2>
            <p>{{ score }}</p>
          </section>
          <section class="stat-row">
            <div class="stat-block compact">
              <h3>Level</h3>
              <p>{{ level }}</p>
            </div>
            <div class="stat-block compact">
              <h3>Lines</h3>
              <p>{{ linesCleared }}</p>
            </div>
          </section>
          <section class="next-piece">
            <h2>Next</h2>
            <div class="mini-grid">
              <div
                v-for="(row, rowIndex) in nextPreview"
                :key="`preview-row-${rowIndex}`"
                class="mini-row"
              >
                <div
                  v-for="(cell, cellIndex) in row"
                  :key="`preview-cell-${rowIndex}-${cellIndex}`"
                  class="mini-cell"
                  :class="{ filled: cell }"
                />
              </div>
            </div>
          </section>
          <section class="controls">
            <button class="primary" @click="startGame">
              {{ gameOver ? 'Restart' : isRunning ? 'Restart' : 'Start Game' }}
            </button>
            <button
              class="ghost"
              :disabled="!isRunning || gameOver"
              @click="togglePause"
            >
              {{ isPaused ? 'Resume' : 'Pause' }}
            </button>
            <button
              class="ghost"
              :disabled="!isRunning || isPaused || gameOver"
              @click="hardDrop"
            >
              Hard Drop
            </button>
          </section>
          <section class="legend">
            <h2>Controls</h2>
            <ul>
              <li><span>← / →</span> Move</li>
              <li><span>↑</span> Rotate</li>
              <li><span>↓</span> Soft drop</li>
              <li><span>Space</span> Hard drop</li>
              <li><span>P</span> Pause</li>
              <li><span>R</span> Restart</li>
            </ul>
          </section>
        </aside>
        <main class="board-wrapper">
          <div class="board" :class="{ paused: isPaused && !gameOver }">
            <div
              v-for="(row, rowIndex) in renderedBoard"
              :key="`row-${rowIndex}`"
              class="row"
            >
              <div
                v-for="(cell, cellIndex) in row"
                :key="`cell-${rowIndex}-${cellIndex}`"
                class="cell"
                :class="{
                  filled: Boolean(cell),
                  active: cell && cell.state === 'active',
                  ghost: cell && cell.state === 'ghost',
                }"
                :style="cell ? { '--cell-color': cell.color } : undefined"
              >
                <span v-if="cell && cell.state === 'locked'" class="shine" />
              </div>
            </div>
            <div v-if="gameOver" class="overlay">
              <div class="overlay-card">
                <h2>Game Over</h2>
                <p>
                  You stacked {{ linesCleared }} lines with {{ score }} points.
                </p>
                <button class="primary" @click="startGame">Play again</button>
              </div>
            </div>
            <div v-else-if="isPaused" class="overlay">
              <div class="overlay-card">
                <h2>Paused</h2>
                <p>Press resume or hit <strong>P</strong> to keep playing.</p>
              </div>
            </div>
          </div>
        </main>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'

type RotationState = Array<{ x: number; y: number }>

type PieceState = 'locked' | 'active' | 'ghost'

interface RenderCell {
  color: string
  state: PieceState
}

type PieceKey = 'I' | 'J' | 'L' | 'O' | 'S' | 'T' | 'Z'

interface BoardCell {
  color: string
  name: PieceKey
}

interface Tetromino {
  color: string
  rotations: RotationState[]
}

const BOARD_WIDTH = 10
const BOARD_HEIGHT = 20

const TETROMINOS: Record<PieceKey, Tetromino> = {
  I: {
    color: '#4de2f8',
    rotations: [
      [
        { x: 0, y: 1 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
        { x: 3, y: 1 },
      ],
      [
        { x: 2, y: 0 },
        { x: 2, y: 1 },
        { x: 2, y: 2 },
        { x: 2, y: 3 },
      ],
      [
        { x: 0, y: 2 },
        { x: 1, y: 2 },
        { x: 2, y: 2 },
        { x: 3, y: 2 },
      ],
      [
        { x: 1, y: 0 },
        { x: 1, y: 1 },
        { x: 1, y: 2 },
        { x: 1, y: 3 },
      ],
    ],
  },
  J: {
    color: '#5d5fe9',
    rotations: [
      [
        { x: 0, y: 0 },
        { x: 0, y: 1 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
      ],
      [
        { x: 1, y: 0 },
        { x: 2, y: 0 },
        { x: 1, y: 1 },
        { x: 1, y: 2 },
      ],
      [
        { x: 0, y: 1 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
        { x: 2, y: 2 },
      ],
      [
        { x: 1, y: 0 },
        { x: 1, y: 1 },
        { x: 0, y: 2 },
        { x: 1, y: 2 },
      ],
    ],
  },
  L: {
    color: '#ff8f3f',
    rotations: [
      [
        { x: 2, y: 0 },
        { x: 0, y: 1 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
      ],
      [
        { x: 1, y: 0 },
        { x: 1, y: 1 },
        { x: 1, y: 2 },
        { x: 2, y: 2 },
      ],
      [
        { x: 0, y: 1 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
        { x: 0, y: 2 },
      ],
      [
        { x: 0, y: 0 },
        { x: 1, y: 0 },
        { x: 1, y: 1 },
        { x: 1, y: 2 },
      ],
    ],
  },
  O: {
    color: '#f9d423',
    rotations: [
      [
        { x: 1, y: 0 },
        { x: 2, y: 0 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
      ],
      [
        { x: 1, y: 0 },
        { x: 2, y: 0 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
      ],
      [
        { x: 1, y: 0 },
        { x: 2, y: 0 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
      ],
      [
        { x: 1, y: 0 },
        { x: 2, y: 0 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
      ],
    ],
  },
  S: {
    color: '#72e06a',
    rotations: [
      [
        { x: 1, y: 0 },
        { x: 2, y: 0 },
        { x: 0, y: 1 },
        { x: 1, y: 1 },
      ],
      [
        { x: 1, y: 0 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
        { x: 2, y: 2 },
      ],
      [
        { x: 1, y: 1 },
        { x: 2, y: 1 },
        { x: 0, y: 2 },
        { x: 1, y: 2 },
      ],
      [
        { x: 0, y: 0 },
        { x: 0, y: 1 },
        { x: 1, y: 1 },
        { x: 1, y: 2 },
      ],
    ],
  },
  T: {
    color: '#b379ff',
    rotations: [
      [
        { x: 1, y: 0 },
        { x: 0, y: 1 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
      ],
      [
        { x: 1, y: 0 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
        { x: 1, y: 2 },
      ],
      [
        { x: 0, y: 1 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
        { x: 1, y: 2 },
      ],
      [
        { x: 1, y: 0 },
        { x: 0, y: 1 },
        { x: 1, y: 1 },
        { x: 1, y: 2 },
      ],
    ],
  },
  Z: {
    color: '#ff4f81',
    rotations: [
      [
        { x: 0, y: 0 },
        { x: 1, y: 0 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
      ],
      [
        { x: 2, y: 0 },
        { x: 1, y: 1 },
        { x: 2, y: 1 },
        { x: 1, y: 2 },
      ],
      [
        { x: 0, y: 1 },
        { x: 1, y: 1 },
        { x: 1, y: 2 },
        { x: 2, y: 2 },
      ],
      [
        { x: 1, y: 0 },
        { x: 0, y: 1 },
        { x: 1, y: 1 },
        { x: 0, y: 2 },
      ],
    ],
  },
}

function randomPiece(): PieceKey {
  const keys = Object.keys(TETROMINOS) as PieceKey[]
  const index = Math.floor(Math.random() * keys.length)
  return keys[index]
}

export default Vue.extend({
  name: 'TetrisGame',
  data() {
    return {
      board: this.createEmptyBoard() as (BoardCell | null)[][],
      currentPieceKey: null as PieceKey | null,
      currentRotation: 0,
      currentPosition: { x: 3, y: -2 },
      nextPieceKey: randomPiece() as PieceKey,
      loopId: null as number | null,
      score: 0,
      linesCleared: 0,
      level: 1,
      isRunning: false,
      isPaused: false,
      gameOver: false,
      lastDropTime: 0,
      dropInterval: 800,
    }
  },
  computed: {
    renderedBoard(): Array<Array<RenderCell | null>> {
      const grid: Array<Array<RenderCell | null>> = this.board.map((row) =>
        row.map((cell) =>
          cell ? { color: cell.color, state: 'locked' as PieceState } : null
        )
      )

      if (!this.currentPieceKey) {
        return grid
      }

      const piece = TETROMINOS[this.currentPieceKey]
      const color = piece.color

      const ghostPosition = this.getGhostPosition()
      const ghostBlocks = this.getBlocks(
        this.currentPieceKey,
        this.currentRotation,
        ghostPosition
      )
      ghostBlocks.forEach(({ x, y }) => {
        if (y < 0 || y >= BOARD_HEIGHT || x < 0 || x >= BOARD_WIDTH) {
          return
        }
        if (!grid[y][x]) {
          grid[y][x] = { color, state: 'ghost' }
        }
      })

      const activeBlocks = this.getBlocks(
        this.currentPieceKey,
        this.currentRotation,
        this.currentPosition
      )
      activeBlocks.forEach(({ x, y }) => {
        if (y < 0 || y >= BOARD_HEIGHT || x < 0 || x >= BOARD_WIDTH) {
          return
        }
        grid[y][x] = { color, state: 'active' }
      })

      return grid
    },
    nextPreview(): boolean[][] {
      const size = 4
      const grid = Array.from({ length: size }, () => Array(size).fill(false))
      const nextKey = this.nextPieceKey
      if (!nextKey) {
        return grid
      }
      const piece = TETROMINOS[nextKey]
      const blocks = piece.rotations[0]
      blocks.forEach(({ x, y }) => {
        const col = x - 1
        const row = y
        if (row >= 0 && row < size && col >= 0 && col < size) {
          grid[row][col] = true
        }
      })
      return grid
    },
  },
  mounted() {
    if (typeof window !== 'undefined') {
      window.addEventListener('keydown', this.handleKeydown)
    }
  },
  beforeDestroy() {
    if (typeof window !== 'undefined') {
      window.removeEventListener('keydown', this.handleKeydown)
      this.stopLoop()
    }
  },
  methods: {
    createEmptyBoard(): (BoardCell | null)[][] {
      return Array.from({ length: BOARD_HEIGHT }, () =>
        Array(BOARD_WIDTH).fill(null)
      )
    },
    startGame(): void {
      this.stopLoop()
      this.board = this.createEmptyBoard()
      this.score = 0
      this.linesCleared = 0
      this.level = 1
      this.dropInterval = 800
      this.isRunning = true
      this.isPaused = false
      this.gameOver = false
      this.currentPieceKey = null
      this.currentRotation = 0
      this.currentPosition = { x: 3, y: -2 }
      this.nextPieceKey = randomPiece()
      this.spawnPiece()
      this.lastDropTime = performance.now()
      this.startLoop()
    },
    startLoop(): void {
      if (typeof window === 'undefined') {
        return
      }
      this.stopLoop()
      this.loopId = window.setInterval(() => {
        this.tick()
      }, 16)
    },
    stopLoop(): void {
      if (this.loopId !== null) {
        window.clearInterval(this.loopId)
        this.loopId = null
      }
    },
    tick(): void {
      if (
        !this.isRunning ||
        this.isPaused ||
        this.gameOver ||
        !this.currentPieceKey
      ) {
        return
      }
      const now = performance.now()
      if (now - this.lastDropTime >= this.dropInterval) {
        this.softDrop()
        this.lastDropTime = now
      }
    },
    handleKeydown(event: KeyboardEvent): void {
      const keys = [
        'ArrowLeft',
        'ArrowRight',
        'ArrowDown',
        'ArrowUp',
        ' ',
        'Spacebar',
        'p',
        'P',
        'r',
        'R',
      ]
      if (keys.includes(event.key)) {
        event.preventDefault()
      }

      if (event.key === 'r' || event.key === 'R') {
        this.startGame()
        return
      }

      if (!this.isRunning || this.gameOver) {
        if (
          event.key === ' ' ||
          event.key === 'Spacebar' ||
          event.key === 'ArrowUp'
        ) {
          this.startGame()
        }
        return
      }

      if (event.key === 'p' || event.key === 'P') {
        this.togglePause()
        return
      }

      if (this.isPaused) {
        return
      }

      switch (event.key) {
        case 'ArrowLeft':
          this.move(-1)
          break
        case 'ArrowRight':
          this.move(1)
          break
        case 'ArrowDown':
          this.softDrop()
          break
        case 'ArrowUp':
          this.rotate()
          break
        case ' ':
        case 'Spacebar':
          this.hardDrop()
          break
        default:
      }
    },
    togglePause(): void {
      if (!this.isRunning || this.gameOver) {
        return
      }
      this.isPaused = !this.isPaused
      if (!this.isPaused) {
        this.lastDropTime = performance.now()
      }
    },
    spawnPiece(): void {
      const pieceKey = this.nextPieceKey || randomPiece()
      this.currentPieceKey = pieceKey
      this.currentRotation = 0
      this.currentPosition = { x: 3, y: -2 }
      this.nextPieceKey = randomPiece()

      if (!this.canPlace(this.currentPosition, this.currentRotation)) {
        this.endGame()
      }
    },
    canPlace(
      position: { x: number; y: number },
      rotationIndex: number
    ): boolean {
      if (!this.currentPieceKey) {
        return false
      }
      const blocks = this.getBlocks(
        this.currentPieceKey,
        rotationIndex,
        position
      )
      return blocks.every(({ x, y }) => {
        if (x < 0 || x >= BOARD_WIDTH) {
          return false
        }
        if (y >= BOARD_HEIGHT) {
          return false
        }
        if (y < 0) {
          return true
        }
        return !this.board[y][x]
      })
    },
    getBlocks(
      pieceKey: PieceKey,
      rotationIndex: number,
      position: { x: number; y: number }
    ): Array<{ x: number; y: number }> {
      const piece = TETROMINOS[pieceKey]
      const rotation = piece.rotations[rotationIndex % piece.rotations.length]
      return rotation.map((block) => ({
        x: position.x + block.x,
        y: position.y + block.y,
      }))
    },
    move(offsetX: number): void {
      if (!this.currentPieceKey) {
        return
      }
      const nextPosition = {
        x: this.currentPosition.x + offsetX,
        y: this.currentPosition.y,
      }
      if (this.canPlace(nextPosition, this.currentRotation)) {
        this.currentPosition = nextPosition
      }
    },
    softDrop(): void {
      if (!this.currentPieceKey) {
        return
      }
      const nextPosition = {
        x: this.currentPosition.x,
        y: this.currentPosition.y + 1,
      }
      if (this.canPlace(nextPosition, this.currentRotation)) {
        this.currentPosition = nextPosition
      } else {
        this.lockPiece()
      }
    },
    hardDrop(): void {
      if (
        !this.currentPieceKey ||
        !this.isRunning ||
        this.isPaused ||
        this.gameOver
      ) {
        return
      }
      let distance = 0
      while (
        this.canPlace(
          {
            x: this.currentPosition.x,
            y: this.currentPosition.y + distance + 1,
          },
          this.currentRotation
        )
      ) {
        distance += 1
      }
      this.currentPosition = {
        x: this.currentPosition.x,
        y: this.currentPosition.y + distance,
      }
      this.lockPiece()
    },
    rotate(): void {
      if (!this.currentPieceKey) {
        return
      }
      const nextRotation =
        (this.currentRotation + 1) %
        TETROMINOS[this.currentPieceKey].rotations.length
      const kicks = [
        { x: 0, y: 0 },
        { x: -1, y: 0 },
        { x: 1, y: 0 },
        { x: 0, y: -1 },
        { x: -2, y: 0 },
        { x: 2, y: 0 },
      ]
      for (const kick of kicks) {
        const testPosition = {
          x: this.currentPosition.x + kick.x,
          y: this.currentPosition.y + kick.y,
        }
        if (this.canPlace(testPosition, nextRotation)) {
          this.currentRotation = nextRotation
          this.currentPosition = testPosition
          return
        }
      }
    },
    lockPiece(): void {
      if (!this.currentPieceKey) {
        return
      }
      const pieceKey = this.currentPieceKey
      const color = TETROMINOS[pieceKey].color
      const blocks = this.getBlocks(
        pieceKey,
        this.currentRotation,
        this.currentPosition
      )
      blocks.forEach(({ x, y }) => {
        if (y < 0) {
          return
        }
        if (y >= BOARD_HEIGHT || x < 0 || x >= BOARD_WIDTH) {
          return
        }
        this.$set(this.board[y], x, { color, name: pieceKey })
      })
      const cleared = this.clearLines()
      this.updateScore(cleared)
      this.spawnPiece()
    },
    clearLines(): number {
      const remaining: (BoardCell | null)[][] = []
      let cleared = 0
      for (let row = 0; row < BOARD_HEIGHT; row += 1) {
        const isFull = this.board[row].every((cell) => cell)
        if (isFull) {
          cleared += 1
        } else {
          remaining.push(this.board[row])
        }
      }
      while (remaining.length < BOARD_HEIGHT) {
        remaining.unshift(Array(BOARD_WIDTH).fill(null))
      }
      this.board = remaining
      this.linesCleared += cleared
      return cleared
    },
    updateScore(lines: number): void {
      if (lines === 0) {
        return
      }
      const lineValues = [0, 100, 300, 500, 800]
      this.score += lineValues[lines] * this.level
      const newLevel = Math.floor(this.linesCleared / 10) + 1
      if (newLevel !== this.level) {
        this.level = newLevel
        this.dropInterval = Math.max(120, 800 - (this.level - 1) * 70)
      }
    },
    getGhostPosition(): { x: number; y: number } {
      if (!this.currentPieceKey) {
        return this.currentPosition
      }
      let offset = 0
      while (
        this.canPlace(
          { x: this.currentPosition.x, y: this.currentPosition.y + offset + 1 },
          this.currentRotation
        )
      ) {
        offset += 1
      }
      return { x: this.currentPosition.x, y: this.currentPosition.y + offset }
    },
    endGame(): void {
      this.gameOver = true
      this.isRunning = false
      this.stopLoop()
    },
  },
})
</script>

<style scoped>
.tetris-shell {
  position: relative;
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2rem;
  background: radial-gradient(
      circle at top,
      rgba(77, 226, 248, 0.25),
      transparent 60%
    ),
    radial-gradient(
      circle at bottom,
      rgba(182, 121, 255, 0.25),
      transparent 55%
    ),
    #050810;
  color: #f6f7fb;
  overflow: hidden;
}

.tetris-background::before,
.tetris-background::after {
  content: '';
  position: absolute;
  width: 30rem;
  height: 30rem;
  background: radial-gradient(circle, rgba(77, 226, 248, 0.4), transparent 60%);
  filter: blur(60px);
  z-index: 0;
}

.tetris-background::after {
  top: -10rem;
  right: -12rem;
}

.tetris-background::before {
  bottom: -12rem;
  left: -10rem;
}

.tetris-card {
  position: relative;
  z-index: 1;
  backdrop-filter: blur(18px);
  background: rgba(12, 16, 27, 0.7);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 32px;
  padding: 2.5rem;
  box-shadow: 0 25px 60px rgba(5, 8, 16, 0.6);
  max-width: 1100px;
  width: 100%;
}

.tetris-header {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-bottom: 2rem;
}

.tetris-header h1 {
  font-size: 2.5rem;
  font-weight: 700;
  letter-spacing: 0.15rem;
  text-transform: uppercase;
  color: #ffffff;
}

.subtitle {
  color: rgba(255, 255, 255, 0.6);
  font-size: 1rem;
}

.tetris-layout {
  display: grid;
  grid-template-columns: minmax(240px, 1fr) minmax(320px, 420px);
  gap: 2.5rem;
  align-items: stretch;
}

.sidebar {
  display: flex;
  flex-direction: column;
  gap: 1.75rem;
}

.stat-block {
  background: linear-gradient(
    135deg,
    rgba(77, 226, 248, 0.16),
    rgba(182, 121, 255, 0.1)
  );
  border-radius: 20px;
  padding: 1.4rem 1.6rem;
  border: 1px solid rgba(255, 255, 255, 0.08);
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.04);
}

.stat-block h2,
.stat-block h3 {
  text-transform: uppercase;
  letter-spacing: 0.12rem;
  font-size: 0.8rem;
  color: rgba(255, 255, 255, 0.55);
}

.stat-block p {
  font-size: 2rem;
  font-weight: 600;
  color: #ffffff;
}

.stat-block.compact {
  padding: 1rem 1.2rem;
  align-items: center;
  text-align: center;
}

.stat-block.compact p {
  font-size: 1.6rem;
}

.stat-row {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1rem;
}

.next-piece {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.mini-grid {
  background: rgba(8, 10, 18, 0.8);
  border-radius: 18px;
  border: 1px solid rgba(255, 255, 255, 0.07);
  padding: 0.75rem;
  display: grid;
  gap: 0.35rem;
}

.mini-row {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 0.35rem;
}

.mini-cell {
  width: 2.4rem;
  aspect-ratio: 1;
  border-radius: 10px;
  background: rgba(255, 255, 255, 0.06);
  box-shadow: inset 0 1px 3px rgba(255, 255, 255, 0.05);
}

.mini-cell.filled {
  background: linear-gradient(
    135deg,
    rgba(255, 255, 255, 0.6),
    rgba(255, 255, 255, 0.15)
  );
}

.controls {
  display: grid;
  gap: 0.75rem;
}

button {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI',
    sans-serif;
  font-size: 0.95rem;
  border-radius: 16px;
  padding: 0.85rem 1.2rem;
  border: none;
  cursor: pointer;
  transition: transform 0.15s ease, box-shadow 0.2s ease;
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}

button.primary {
  background: linear-gradient(135deg, #4de2f8, #b679ff);
  color: #050810;
  font-weight: 600;
  box-shadow: 0 12px 24px rgba(77, 226, 248, 0.25);
}

button.primary:hover:not(:disabled) {
  transform: translateY(-1px);
  box-shadow: 0 16px 30px rgba(77, 226, 248, 0.35);
}

button.ghost {
  background: rgba(255, 255, 255, 0.08);
  color: #f6f7fb;
  border: 1px solid rgba(255, 255, 255, 0.08);
}

button.ghost:hover:not(:disabled) {
  transform: translateY(-1px);
  background: rgba(255, 255, 255, 0.12);
}

.legend ul {
  list-style: none;
  padding: 0;
  margin: 0;
  display: grid;
  gap: 0.35rem;
}

.legend li {
  display: flex;
  justify-content: space-between;
  color: rgba(255, 255, 255, 0.65);
  font-size: 0.95rem;
}

.legend span {
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
  color: rgba(255, 255, 255, 0.9);
}

.board-wrapper {
  position: relative;
  border-radius: 30px;
  padding: 1.2rem;
  background: linear-gradient(
    135deg,
    rgba(77, 226, 248, 0.15),
    rgba(255, 79, 129, 0.12)
  );
  border: 1px solid rgba(255, 255, 255, 0.08);
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.04);
}

.board {
  position: relative;
  width: min(60vw, 22rem);
  aspect-ratio: 10 / 20;
  display: grid;
  grid-template-rows: repeat(20, 1fr);
  background: rgba(5, 8, 16, 0.85);
  border-radius: 20px;
  overflow: hidden;
  border: 1px solid rgba(255, 255, 255, 0.08);
}

.row {
  display: grid;
  grid-template-columns: repeat(10, 1fr);
}

.cell {
  border: 1px solid rgba(255, 255, 255, 0.03);
  position: relative;
  background: rgba(10, 15, 24, 0.6);
}

.cell.filled {
  background: var(--cell-color);
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.12),
    0 4px 14px rgba(0, 0, 0, 0.35);
}

.cell.active {
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.35),
    0 8px 18px rgba(77, 226, 248, 0.4);
}

.cell.ghost {
  background: rgba(255, 255, 255, 0.05);
  border: 1px dashed rgba(255, 255, 255, 0.2);
  box-shadow: none;
}

.cell .shine {
  content: '';
  position: absolute;
  top: 15%;
  left: 10%;
  width: 60%;
  height: 40%;
  background: rgba(255, 255, 255, 0.25);
  border-radius: 50%;
  filter: blur(6px);
}

.overlay {
  position: absolute;
  inset: 0;
  background: rgba(5, 8, 16, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
}

.overlay-card {
  background: rgba(12, 16, 27, 0.9);
  border-radius: 20px;
  padding: 2rem 2.5rem;
  text-align: center;
  border: 1px solid rgba(255, 255, 255, 0.08);
  box-shadow: 0 20px 45px rgba(5, 8, 16, 0.6);
}

.overlay-card h2 {
  font-size: 1.75rem;
  margin-bottom: 0.75rem;
}

.overlay-card p {
  color: rgba(255, 255, 255, 0.7);
  margin-bottom: 1.5rem;
}

@media (max-width: 1024px) {
  .tetris-layout {
    grid-template-columns: 1fr;
  }

  .board {
    width: min(80vw, 22rem);
  }
}

@media (max-width: 640px) {
  .tetris-card {
    padding: 1.75rem;
  }

  .tetris-shell {
    padding: 1.5rem;
  }

  .board {
    width: 100%;
  }

  .mini-cell {
    width: 1.8rem;
  }
}
</style>
