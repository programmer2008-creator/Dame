<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

type Player = 'A' | 'B'

type Piece = {
  id: number
  player: Player
  king: boolean
}

type Cell = Piece | null

type Mode = 'menu' | 'playing' | 'gameover'

type GameMode = 'pvp' | 'cpu'

const MAX_CPU_LEVEL = 10
const CPU_LEVEL_LABELS = [
  'Beginner', 'Novice', 'Casual', 'Learner', 'Standard',
  'Skilled', 'Tough', 'Expert', 'Master', 'Champion',
] as const

type Move = {
  from: number
  to: number
  captures: number[] // list of captured piece indices (may be multi)
  promote: boolean
}

const N = 10
const BOARD_SIZE = N * N
const DARK_SQUARE = (r: number, c: number) => (r + c) % 2 === 1

const humanPlayer: Player = 'A'
const cpuPlayer: Player = 'B'
let nextPieceId = 0

const mode = ref<Mode>('menu')
const gameMode = ref<GameMode>('pvp')
const cpuLevel = ref(1)

// Board: index = r*N + c
const board = ref<Cell[]>(Array.from({ length: BOARD_SIZE }, () => null))
const turn = ref<Player>('A')
const winner = ref<Player | null>(null)

// Selection & move
const selected = ref<number | null>(null)
const legalMovesForSelected = ref<Move[]>([])
const forcedCaptures = ref<Move[]>([])
const isThinking = ref(false)
const dragging = ref<{ from: number; startX: number; startY: number; ghostX: number; ghostY: number } | null>(null)
const dragMoved = ref(false)
const isDraggingMode = ref(false)

// Game tracking
const lastMove = ref<{ from: number; to: number } | null>(null)
const moveCount = ref(0)
const capturedByA = ref(0)
const capturedByB = ref(0)
const pieceCounts = computed(() => {
  let a = 0, b = 0, ak = 0, bk = 0
  for (const cell of board.value) {
    if (!cell) continue
    if (cell.player === 'A') { a++; if (cell.king) ak++ }
    else { b++; if (cell.king) bk++ }
  }
  return { a, b, ak, bk }
})

// Animation/background canvas
const bgCanvas = ref<HTMLCanvasElement | null>(null)
let bgCtx: CanvasRenderingContext2D | null = null
let bgRaf = 0
let bgW = 0
let bgH = 0
let particles: {
  x: number
  y: number
  vx: number
  vy: number
  r: number
  a: number
}[] = []

// Mouse interaction for background
const mouseX = ref(0)
const mouseY = ref(0)
const mouseActive = ref(false)

function cpuDepthForLevel(level: number): number {
  return 2 + Math.floor(((level - 1) * 4) / (MAX_CPU_LEVEL - 1))
}

function cpuMobilityWeight(level: number): number {
  return 0.04 + level * 0.008
}

const cpuLevelLabel = computed(() => CPU_LEVEL_LABELS[cpuLevel.value - 1] ?? `Level ${cpuLevel.value}`)

const nextCpuLevelLabel = computed(() => CPU_LEVEL_LABELS[cpuLevel.value] ?? '')

function rand(min: number, max: number) {
  return min + Math.random() * (max - min)
}

function resetMatchStats() {
  selected.value = null
  legalMovesForSelected.value = []
  forcedCaptures.value = []
  isThinking.value = false
  lastMove.value = null
  moveCount.value = 0
  capturedByA.value = 0
  capturedByB.value = 0
}

function beginRound() {
  winner.value = null
  resetMatchStats()
  setupBoard()
  updateForcedCaptures()
  if (gameMode.value === 'cpu' && turn.value === cpuPlayer) void cpuTurn()
}

function setupBoard() {
  nextPieceId = 0
  board.value = Array.from({ length: BOARD_SIZE }, () => null)

  for (let r = 0; r < N; r++) {
    for (let c = 0; c < N; c++) {
      if (!DARK_SQUARE(r, c)) continue
      const idx = r * N + c
      if (r <= 3) board.value[idx] = { id: nextPieceId++, player: 'A', king: false }
      else if (r >= 6) board.value[idx] = { id: nextPieceId++, player: 'B', king: false }
    }
  }

  turn.value = 'A'
  forcedCaptures.value = getAllLegalMoves('both').filter((m) => m.captures.length > 0)
  mode.value = 'playing'
}

function idxToRC(idx: number) {
  return { r: Math.floor(idx / N), c: idx % N }
}

function rcToIdx(r: number, c: number) {
  return r * N + c
}

function inBounds(r: number, c: number) {
  return r >= 0 && r < N && c >= 0 && c < N
}

function getPieceMovesFromBoard(b: Cell[], from: number): Move[] {
  const p = b[from]
  if (!p) return []

  const { r, c } = idxToRC(from)

  const moves: Move[] = []

  // International draughts: ALL pieces can capture in all 4 diagonal directions
  const captureDirs = [-1, 1]

  const captureResults: Move[] = []

  // Generate multi-jumps (DFS)
  function dfs(curBoard: Cell[], curFrom: number, curPiece: Piece, pathCaptures: number[]) {
    const { r: cr, c: cc } = idxToRC(curFrom)

    // Check if current position is a promotion square — if so, promote & end sequence
    if (!curPiece.king) {
      const onKingRow = curPiece.player === 'A' ? cr === N - 1 : cr === 0
      if (onKingRow) {
        if (pathCaptures.length > 0) {
          captureResults.push({ from, to: curFrom, captures: pathCaptures, promote: true })
        }
        return
      }
    }

    let found = false
    for (const d of captureDirs) {
      for (const dc of [-1, 1]) {
        const midR = cr + d
        const midC = cc + dc
        const landR = cr + 2 * d
        const landC = cc + 2 * dc
        if (!inBounds(midR, midC) || !inBounds(landR, landC)) continue
        const midIdx = rcToIdx(midR, midC)
        const landIdx = rcToIdx(landR, landC)
        const midPiece = curBoard[midIdx]
        if (!midPiece || midPiece.player === curPiece.player) continue
        if (curBoard[landIdx]) continue

        found = true

        const nextBoard = curBoard.slice()
        nextBoard[curFrom] = null
        nextBoard[midIdx] = null
        nextBoard[landIdx] = { ...curPiece }

        const nextCaptures = pathCaptures.concat([midIdx])
        dfs(nextBoard, landIdx, curPiece, nextCaptures)
      }
    }

    if (!found && pathCaptures.length > 0) {
      const { r: fr } = idxToRC(curFrom)
      const promote = !curPiece.king && (curPiece.player === 'A' ? fr === N - 1 : fr === 0)
      captureResults.push({ from, to: curFrom, captures: pathCaptures, promote })
    }
  }

  dfs(b.slice(), from, p, [])

  moves.push(...captureResults)

  // If captures exist, normal moves are not allowed.
  if (moves.some((m) => m.captures.length > 0)) return moves

  // Normal (non-capture) moves
  const stepDirs = p.king ? [-1, 1] : [p.player === 'A' ? 1 : -1]
  for (const d of stepDirs) {
    for (const dc of [-1, 1]) {
      const nr = r + d
      const nc = c + dc
      if (!inBounds(nr, nc)) continue
      const to = rcToIdx(nr, nc)
      if (!DARK_SQUARE(nr, nc)) continue
      if (b[to]) continue

      const { r: pr } = idxToRC(to)
      const promote = !p.king && (p.player === 'A' ? pr === N - 1 : pr === 0)
      moves.push({ from, to, captures: [], promote })
    }
  }

  return moves
}

function getPieceMovesFrom(from: number): Move[] {
  return getPieceMovesFromBoard(board.value, from)
}

function applyMoveOnBoard(srcBoard: Cell[], move: Move, currentTurn: Player) {
  const next = srcBoard.slice()
  const piece = next[move.from]
  if (!piece) return next

  next[move.from] = null

  // remove captured pieces
  for (const capIdx of move.captures) next[capIdx] = null

  let placed: Piece = { ...piece }

  // Promote if needed after the sequence.
  if (move.promote) placed = { player: piece.player, king: true }

  next[move.to] = placed
  return next
}

function getAllLegalMovesFromBoard(b: Cell[], forSide: Player | 'both'): Move[] {
  const sides: Player[] = forSide === 'both' ? ['A', 'B'] : [forSide]
  const all: Move[] = []

  for (const s of sides) {
    for (let i = 0; i < BOARD_SIZE; i++) {
      const p = b[i]
      if (!p || p.player !== s) continue
      all.push(...getPieceMovesFromBoard(b, i))
    }
  }

  const anyCaptures = all.some((m) => m.captures.length > 0)
  if (anyCaptures) return all.filter((m) => m.captures.length > 0)
  return all
}

function getAllLegalMoves(forSide: Player | 'both'): Move[] {
  return getAllLegalMovesFromBoard(board.value, forSide)
}

function updateForcedCaptures() {
  const captures = getAllLegalMoves(turn.value).filter((m) => m.captures.length > 0)
  forcedCaptures.value = captures
}

function refreshSelectionMoves() {
  if (selected.value == null) {
    legalMovesForSelected.value = []
    return
  }
  const from = selected.value
  const piece = board.value[from]
  if (!piece || piece.player !== turn.value) {
    selected.value = null
    legalMovesForSelected.value = []
    return
  }

  const all = getAllLegalMoves(turn.value)
  legalMovesForSelected.value = all.filter((m) => m.from === from)
}

function onCellPointerDown(idx: number, e: PointerEvent) {
  if (mode.value !== 'playing' || isThinking.value) return
  if (gameMode.value === 'cpu' && turn.value !== humanPlayer) return
  const p = board.value[idx]
  if (!p || p.player !== turn.value) return
  const allForTurn = getAllLegalMoves(turn.value)
  const hasAnyCaptures = allForTurn.some((m) => m.captures.length > 0)
  const pieceMoves = allForTurn.filter((m) => m.from === idx)
  if (hasAnyCaptures && !pieceMoves.some((m) => m.captures.length > 0)) return
  dragging.value = { from: idx, startX: e.clientX, startY: e.clientY, ghostX: 0, ghostY: 0 }
  dragMoved.value = false
  isDraggingMode.value = false
  document.addEventListener('pointermove', onDocPointerMove)
  document.addEventListener('pointerup', onDocPointerUp)
}

function onDocPointerMove(e: PointerEvent) {
  if (!dragging.value) return
  const dx = e.clientX - dragging.value.startX
  const dy = e.clientY - dragging.value.startY
  dragging.value.ghostX = dx
  dragging.value.ghostY = dy
  if (!isDraggingMode.value && (Math.abs(dx) > 4 || Math.abs(dy) > 4)) {
    isDraggingMode.value = true
    dragMoved.value = true
    const allForTurn = getAllLegalMoves(turn.value)
    const hasAnyCaptures = allForTurn.some((m) => m.captures.length > 0)
    const pieceMoves = allForTurn.filter((m) => m.from === dragging.value!.from)
    if (hasAnyCaptures && !pieceMoves.some((m) => m.captures.length > 0)) {
      isDraggingMode.value = false
      dragging.value = null
      document.removeEventListener('pointermove', onDocPointerMove)
      document.removeEventListener('pointerup', onDocPointerUp)
      return
    }
    selected.value = dragging.value.from
    refreshSelectionMoves()
  }
  e.preventDefault()
}

function onDocPointerUp(e: PointerEvent) {
  if (!dragging.value) return
  if (isDraggingMode.value) {
    const el = document.elementFromPoint(e.clientX, e.clientY)
    if (el) {
      const cellEl = (el as HTMLElement).closest('[data-idx]')
      if (cellEl) {
        const toIdx = parseInt(cellEl.getAttribute('data-idx')!, 10)
        const move = legalMovesForSelected.value.find((m) => m.to === toIdx)
        if (move) {
          const curBoard = board.value.slice()
          const nextBoard = applyMoveOnBoard(curBoard, move, turn.value)
          const caps = move.captures.length
          if (caps > 0) {
            if (turn.value === 'A') capturedByA.value += caps
            else capturedByB.value += caps
          }
          board.value = nextBoard
          lastMove.value = { from: move.from, to: move.to }
          moveCount.value++
          selected.value = null
          legalMovesForSelected.value = []
          const nextTurn: Player = turn.value === 'A' ? 'B' : 'A'
          turn.value = nextTurn
          updateForcedCaptures()
          const movesNext = getAllLegalMoves(nextTurn)
          if (movesNext.length === 0) {
            winner.value = nextTurn === 'A' ? 'B' : 'A'
            mode.value = 'gameover'
          } else if (gameMode.value === 'cpu' && nextTurn === cpuPlayer) {
            void cpuTurn()
          }
        }
      }
    }
  }
  dragging.value = null
  document.removeEventListener('pointermove', onDocPointerMove)
  document.removeEventListener('pointerup', onDocPointerUp)
}

function handleCellClick(idx: number) {
  if (dragMoved.value) {
    dragMoved.value = false
    return
  }
  if (mode.value !== 'playing') return
  if (isThinking.value) return

  // vs CPU: only allow selecting/moving for human
  if (gameMode.value === 'cpu' && turn.value !== humanPlayer) return

  const p = board.value[idx]

  // If a capture is forced, only allow selecting pieces that have capture moves.
  const allForTurn = getAllLegalMoves(turn.value)
  const hasAnyCaptures = allForTurn.some((m) => m.captures.length > 0)

  if (selected.value == null) {
    if (!p || p.player !== turn.value) return
    const pieceMoves = allForTurn.filter((m) => m.from === idx)
    if (hasAnyCaptures && !pieceMoves.some((m) => m.captures.length > 0)) return
    selected.value = idx
    refreshSelectionMoves()
    return
  }

  // Clicking selected piece again clears selection.
  if (selected.value === idx) {
    selected.value = null
    legalMovesForSelected.value = []
    return
  }

  // If clicking another own piece, switch selection.
  if (p && p.player === turn.value) {
    const pieceMoves = allForTurn.filter((m) => m.from === idx)
    if (hasAnyCaptures && !pieceMoves.some((m) => m.captures.length > 0)) return
    selected.value = idx
    refreshSelectionMoves()
    return
  }

  // Attempt move to empty/occupied destination (destination must be legal and from selected).
  const move = legalMovesForSelected.value.find((m) => m.to === idx)
  if (!move) return

  const curBoard = board.value.slice()
  const nextBoard = applyMoveOnBoard(curBoard, move, turn.value)

  // Track captures
  const caps = move.captures.length
  if (caps > 0) {
    if (turn.value === 'A') capturedByA.value += caps
    else capturedByB.value += caps
  }

  // Commit
  board.value = nextBoard
  lastMove.value = { from: move.from, to: move.to }
  moveCount.value++
  selected.value = null
  legalMovesForSelected.value = []

  // Switch turn
  const nextTurn: Player = turn.value === 'A' ? 'B' : 'A'
  turn.value = nextTurn

  // Update forced captures for UI
  updateForcedCaptures()

  // check end
  const movesNext = getAllLegalMoves(nextTurn)
  if (movesNext.length === 0) {
    winner.value = nextTurn === 'A' ? 'B' : 'A'
    mode.value = 'gameover'
    return
  }

  // CPU move if needed
  if (gameMode.value === 'cpu' && nextTurn === cpuPlayer) {
    void cpuTurn()
  }
}

function heuristic(b: Cell[], side: Player) {
  // Simple heuristic: piece values + king bonus + mobility
  let score = 0
  for (let i = 0; i < b.length; i++) {
    const p = b[i]
    if (!p) continue
    const mult = p.player === side ? 1 : -1
    const val = p.king ? 3.2 : 1
    score += mult * val
  }

  const myMoves = getAllLegalMovesFromBoard(b, side)
  const oppMoves = getAllLegalMovesFromBoard(b, side === 'A' ? 'B' : 'A')
  const mobilityWeight = gameMode.value === 'cpu' ? cpuMobilityWeight(cpuLevel.value) : 0.05
  score += mobilityWeight * (myMoves.length - oppMoves.length)
  return score
}

function listAllMovesForBoard(b: Cell[], side: Player): Move[] {
  return getAllLegalMovesFromBoard(b, side)
}

function cpuMinimax(b: Cell[], side: Player, depth: number, alpha: number, beta: number): { score: number; best?: Move } {
  const nextSide: Player = side === 'A' ? 'B' : 'A'
  const moves = listAllMovesForBoard(b, side)

  // terminal
  if (depth === 0 || moves.length === 0) {
    if (moves.length === 0) {
      // side cannot move -> losing for side
      const loseScore = side === humanPlayer ? -9999 : 9999
      return { score: loseScore }
    }
    return { score: heuristic(b, cpuPlayer) }
  }

  let bestScore = -Infinity
  let bestMove: Move | undefined

  for (const m of moves) {
    const nextB = applyMoveOnBoard(b, m, side)
    const res = cpuMinimax(nextB, nextSide, depth - 1, -beta, -alpha)
    const score = -res.score

    if (score > bestScore) {
      bestScore = score
      bestMove = m
    }
    alpha = Math.max(alpha, score)
    if (alpha >= beta) break
  }

  return { score: bestScore, best: bestMove }
}

async function cpuTurn() {
  if (mode.value !== 'playing') return
  if (gameMode.value !== 'cpu') return
  if (turn.value !== cpuPlayer) return

  isThinking.value = true
  await Promise.resolve()

  const thinkMs = 3000
  const thinkStart = performance.now()

  const depth = cpuDepthForLevel(cpuLevel.value)
  const b = board.value.slice()

  const { best } = cpuMinimax(b, cpuPlayer, depth, -Infinity, Infinity)

  const thinkRemaining = thinkMs - (performance.now() - thinkStart)
  if (thinkRemaining > 0) await new Promise((r) => setTimeout(r, thinkRemaining))
  if (!best) {
    isThinking.value = false
    return
  }

  const caps = best.captures.length
  if (caps > 0) {
    if (cpuPlayer === 'A') capturedByA.value += caps
    else capturedByB.value += caps
  }
  const nextB = applyMoveOnBoard(b, best, cpuPlayer)
  board.value = nextB
  lastMove.value = { from: best.from, to: best.to }
  moveCount.value++
  selected.value = null
  legalMovesForSelected.value = []

  const nextTurn: Player = cpuPlayer === 'A' ? 'B' : 'A'
  turn.value = nextTurn
  updateForcedCaptures()

  // End game?
  const movesNext = getAllLegalMoves(nextTurn)
  if (movesNext.length === 0) {
    winner.value = nextTurn === 'A' ? 'B' : 'A'
    mode.value = 'gameover'
    isThinking.value = false
    return
  }

  isThinking.value = false
}

function cellClass(idx: number) {
  const { r, c } = idxToRC(idx)
  const isDark = DARK_SQUARE(r, c)
  const p = board.value[idx]
  const isSel = selected.value === idx
  const legalTo = legalMovesForSelected.value.some((m) => m.to === idx)
  const isLastFrom = lastMove.value?.from === idx
  const isLastTo = lastMove.value?.to === idx

  let cls = 'cell'
  cls += isDark ? ' cell--dark' : ' cell--light'
  if (p) cls += p.player === 'A' ? (p.king ? ' piece piece--a piece--king' : ' piece piece--a') : (p.king ? ' piece piece--b piece--king' : ' piece piece--b')
  if (isSel) cls += ' cell--selected'
  if (legalTo) cls += ' cell--legal'
  if (isLastFrom) cls += ' cell--last-from'
  if (isLastTo) cls += ' cell--last-to'
  return cls
}

function pieceDotStyle(idx: number) {
  const p = board.value[idx]
  if (!p) return {}
  if (!p.king) return {}
  // subtle inner highlight for kings
  return {
    background: p.player === 'A'
      ? 'radial-gradient(circle at 30% 30%, #ffe8a3 0%, #ffcf3a 30%, #d19b00 70%, #8a5b00 100%)'
      : 'radial-gradient(circle at 30% 30%, #c7f0ff 0%, #55d8ff 30%, #1f8fb8 70%, #0c5f88 100%)'
  }
}

const livePieces = computed(() => {
  const result: { id: number; player: Player; king: boolean; idx: number }[] = []
  for (let i = 0; i < BOARD_SIZE; i++) {
    const p = board.value[i]
    if (p) result.push({ ...p, idx: i })
  }
  return result
})

function pieceOverlayStyle(idx: number) {
  const { r, c } = idxToRC(idx)
  return {
    left: `${(c / N) * 100}%`,
    top: `${(r / N) * 100}%`,
    width: `${100 / N}%`,
    height: `${100 / N}%`,
  }
}

function pieceOverlayClass(p: { id: number; player: Player; king: boolean; idx: number }) {
  const isSel = selected.value === p.idx
  const isHidden = dragging.value?.from === p.idx && isDraggingMode.value
  return {
    'piece-overlay': true,
    'piece-overlay--a': p.player === 'A',
    'piece-overlay--b': p.player === 'B',
    'piece-overlay--king': p.king,
    'piece-overlay--sel': isSel,
    'piece-overlay--hidden': isHidden,
  }
}

const dragGhostStyle = computed(() => {
  if (!dragging.value || !isDraggingMode.value) return {} as Record<string, string | number>
  const d = dragging.value
  const piece = board.value[d.from]
  let bg = ''
  if (piece) {
    if (piece.king) {
      bg = piece.player === 'A'
        ? 'radial-gradient(circle at 30% 30%, #ffe8a3 0%, #ffcf3a 30%, #d19b00 70%, #8a5b00 100%)'
        : 'radial-gradient(circle at 30% 30%, #c7f0ff 0%, #55d8ff 30%, #1f8fb8 70%, #0c5f88 100%)'
    } else {
      bg = piece.player === 'A'
        ? 'radial-gradient(circle at 30% 25%, #ffffff 0%, #ffdb7a 18%, #ffb300 45%, #c98200 100%)'
        : 'radial-gradient(circle at 30% 25%, #ffffff 0%, #9ff4ff 18%, #49d6ff 45%, #1c88b6 100%)'
    }
  }
  return {
    position: 'fixed' as const,
    left: (d.startX + d.ghostX - 25) + 'px',
    top: (d.startY + d.ghostY - 25) + 'px',
    width: '50px',
    height: '50px',
    borderRadius: '999px',
    pointerEvents: 'none' as const,
    zIndex: 1000,
    opacity: 0.85,
    boxShadow: '0 8px 30px rgba(0,0,0,0.5)',
    background: bg,
  }
})

const statusText = computed(() => {
  if (mode.value === 'menu') return ''
  if (mode.value === 'gameover') {
    if (gameMode.value === 'cpu') {
      if (winner.value === humanPlayer) {
        if (cpuLevel.value >= MAX_CPU_LEVEL) return 'You conquered all 10 levels!'
        return `Level ${cpuLevel.value} complete — advance to Level ${cpuLevel.value + 1}!`
      }
      return `Computer wins Level ${cpuLevel.value}. Try again!`
    }
    return `${winner.value === 'A' ? 'Player 1 (A)' : 'Player 2 (B)'} wins!`
  }
  const who = turn.value === 'A' ? 'Player 1 (A)' : 'Player 2 (B)'
  if (gameMode.value === 'cpu') {
    const lvl = `Level ${cpuLevel.value}/${MAX_CPU_LEVEL} (${cpuLevelLabel.value})`
    if (turn.value === humanPlayer) return `${lvl} · Your turn · Captured: ${capturedByB}`
    return isThinking.value ? `${lvl} · Computer thinking…` : `${lvl} · Computer turn`
  }
  return `Turn: ${who} · A:${capturedByA} B:${capturedByB}`
})

function startGame(choice: GameMode) {
  gameMode.value = choice
  if (choice === 'cpu') cpuLevel.value = 1
  mode.value = 'playing'
  beginRound()
}

function continueToNextLevel() {
  if (cpuLevel.value < MAX_CPU_LEVEL) cpuLevel.value++
  beginRound()
}

function retryCpuLevel() {
  beginRound()
}

function backToMenu() {
  mode.value = 'menu'
  winner.value = null
  isThinking.value = false
}

function restart() {
  if (gameMode.value === 'cpu') retryCpuLevel()
  else startGame(gameMode.value)
}

// Background
function resizeBg() {
  const canvas = bgCanvas.value
  if (!canvas) return
  const parent = canvas.parentElement
  if (!parent) return
  const rect = parent.getBoundingClientRect()
  bgW = Math.max(320, Math.floor(rect.width))
  bgH = Math.max(320, Math.floor(rect.height))
  canvas.width = bgW * devicePixelRatio
  canvas.height = bgH * devicePixelRatio
  canvas.style.width = `${bgW}px`
  canvas.style.height = `${bgH}px`
  bgCtx = canvas.getContext('2d')
  if (bgCtx) {
    bgCtx.setTransform(devicePixelRatio, 0, 0, devicePixelRatio, 0, 0)
  }
}

function initParticles() {
  const count = Math.floor((bgW * bgH) / 12000)
  particles = Array.from({ length: Math.max(40, count) }, () => ({
    x: rand(0, bgW),
    y: rand(0, bgH),
    vx: rand(-0.4, 0.4),
    vy: rand(-0.4, 0.4),
    r: rand(1, 3),
    a: rand(0.2, 0.8)
  }))
}

function drawBg(t: number) {
  if (!bgCtx) return
  const time = t * 0.001

  bgCtx.clearRect(0, 0, bgW, bgH)

  // animated gradient backdrop
  const hueShift = Math.sin(time * 0.03) * 20
  const g = bgCtx.createLinearGradient(0, 0, bgW, bgH)
  g.addColorStop(0, `hsla(${220 + hueShift}, 80%, 50%, 0.15)`)
  g.addColorStop(0.5, `hsla(${270 + hueShift}, 70%, 50%, 0.12)`)
  g.addColorStop(1, `hsla(${170 + hueShift}, 80%, 50%, 0.10)`)
  bgCtx.fillStyle = g
  bgCtx.fillRect(0, 0, bgW, bgH)

  // mouse glow
  if (mouseActive.value) {
    const mg = bgCtx.createRadialGradient(mouseX.value, mouseY.value, 0, mouseX.value, mouseY.value, 200)
    mg.addColorStop(0, 'rgba(180, 230, 255, 0.12)')
    mg.addColorStop(0.5, 'rgba(140, 180, 255, 0.06)')
    mg.addColorStop(1, 'rgba(0, 0, 0, 0)')
    bgCtx.fillStyle = mg
    bgCtx.fillRect(0, 0, bgW, bgH)
  }

  // warp grid
  bgCtx.globalAlpha = 0.3
  bgCtx.strokeStyle = 'rgba(255,255,255,0.06)'
  bgCtx.lineWidth = 1
  const spacing = 28
  for (let x = 0; x < bgW; x += spacing) {
    bgCtx.beginPath()
    for (let y = 0; y <= bgH; y += 14) {
      const yy = y + Math.sin(time * 0.6 + x * 0.018 + y * 0.012) * 2.5
      if (y === 0) bgCtx.moveTo(x, yy)
      else bgCtx.lineTo(x, yy)
    }
    bgCtx.stroke()
  }
  bgCtx.globalAlpha = 1

  // particles — mouse attraction/repulsion
  const mx = mouseActive.value ? mouseX.value : -9999
  const my = mouseActive.value ? mouseY.value : -9999

  for (const p of particles) {
    // mouse interaction
    if (mouseActive.value) {
      const dx = mx - p.x
      const dy = my - p.y
      const dist = Math.sqrt(dx * dx + dy * dy)
      if (dist < 150 && dist > 1) {
        const force = (150 - dist) / 150 * 0.03
        p.vx += (dx / dist) * force
        p.vy += (dy / dist) * force
      }
    }

    // damping
    p.vx *= 0.99
    p.vy *= 0.99

    p.x += p.vx
    p.y += p.vy

    if (p.x < -20) p.x = bgW + 20
    if (p.x > bgW + 20) p.x = -20
    if (p.y < -20) p.y = bgH + 20
    if (p.y > bgH + 20) p.y = -20
  }

  // connections
  const maxDist = 120
  for (let i = 0; i < particles.length; i++) {
    const pa = particles[i]!
    for (let j = i + 1; j < particles.length; j++) {
      const pb = particles[j]!
      const dx = pa.x - pb.x
      const dy = pa.y - pb.y
      const dist2 = dx * dx + dy * dy
      if (dist2 < maxDist * maxDist) {
        const k = 1 - Math.sqrt(dist2) / maxDist
        bgCtx.strokeStyle = `rgba(180,220,255,${0.07 * k})`
        bgCtx.lineWidth = 1
        bgCtx.beginPath()
        bgCtx.moveTo(pa.x, pa.y)
        bgCtx.lineTo(pb.x, pb.y)
        bgCtx.stroke()
      }
    }
  }

  // draw particles
  for (const p of particles) {
    bgCtx.beginPath()
    const alpha = mouseActive.value
      ? Math.min(1, p.a * (1 + 0.4 * (1 - Math.sqrt((p.x - mx) ** 2 + (p.y - my) ** 2) / 300)))
      : p.a
    bgCtx.fillStyle = `rgba(200,230,255,${alpha})`
    bgCtx.arc(p.x, p.y, p.r, 0, Math.PI * 2)
    bgCtx.fill()
  }

  bgRaf = requestAnimationFrame(drawBg)
}

function stopBg() {
  if (bgRaf) cancelAnimationFrame(bgRaf)
}

onMounted(() => {
  resizeBg()
  initParticles()
  bgRaf = requestAnimationFrame(drawBg)

  const onResize = () => {
    resizeBg()
    initParticles()
  }
  window.addEventListener('resize', onResize)

  const onMouseMove = (e: MouseEvent) => {
    mouseX.value = e.clientX
    mouseY.value = e.clientY
    mouseActive.value = true
  }
  const onMouseLeave = () => { mouseActive.value = false }
  document.addEventListener('mousemove', onMouseMove)
  document.addEventListener('mouseleave', onMouseLeave)

  onBeforeUnmount(() => {
    window.removeEventListener('resize', onResize)
    document.removeEventListener('mousemove', onMouseMove)
    document.removeEventListener('mouseleave', onMouseLeave)
    document.removeEventListener('pointermove', onDocPointerMove)
    document.removeEventListener('pointerup', onDocPointerUp)
    stopBg()
  })
})

// Make sure initial forced captures computed
updateForcedCaptures()
</script>

<template>
  <div class="app">
    <header class="topbar">
      <div class="brand">
        <div class="logo">♟︎</div>
        <div>
          <div class="title">Dame</div>
          <div class="subtitle">Vue + Vite • 2 Players / vs Computer</div>
        </div>
      </div>

      <div class="status">
        {{ statusText }}
        <span v-if="mode === 'playing'" class="status__hint">
          · Click to select, then click dest · Or drag & drop
        </span>
      </div>

      <div class="actions">
        <button class="btn btn--ghost" @click="restart" v-if="mode !== 'menu'">
          Restart
        </button>
      </div>
    </header>

    <section class="hero">
      <div class="bg" aria-hidden="true">
        <canvas ref="bgCanvas" class="bg__canvas"></canvas>
        <div class="bg__overlay" />
      </div>

      <div class="hero__content">
        <div class="menu" v-if="mode === 'menu'">
          <h2 class="menu__h2">Choose mode</h2>

          <div class="cards">
            <button class="card" @click="startGame('pvp')">
              <div class="card__title">2 Players</div>
              <div class="card__desc">Local hot-seat: Player A vs Player B.</div>
              <div class="card__meta">No AI</div>
            </button>

            <button class="card" @click="startGame('cpu')">
              <div class="card__title">1 Player vs Computer</div>
              <div class="card__desc">Beat 10 levels — each win unlocks a smarter, tougher computer.</div>
              <div class="card__meta">10 levels · Starts at Level 1 (Beginner)</div>
            </button>
          </div>

          <div class="menu__footer">
            <div class="rules">
              <div class="rule">• Mandatory captures (if available)</div>
              <div class="rule">• Multi-jumps supported</div>
              <div class="rule">• King promotion at the last row</div>
            </div>
          </div>
        </div>

        <div class="game" v-else>
          <div class="layout">
            <div class="boardWrap">
              <div class="board" role="application" aria-label="Dame board">
                <div
                  v-for="idx in BOARD_SIZE"
                  :key="idx - 1"
                  :data-idx="idx - 1"
                  :class="cellClass(idx - 1)"
                  @click="handleCellClick(idx - 1)"
                  @pointerdown="onCellPointerDown(idx - 1, $event)"
                >
                  <div v-if="board[idx - 1]" class="pieceDot" :style="pieceDotStyle(idx - 1)"></div>
                </div>
              </div>

              <div class="controls">
                <div class="pill" v-if="gameMode === 'cpu'">Level {{ cpuLevel }}/{{ MAX_CPU_LEVEL }}</div>
                <div class="pill">Move {{ moveCount }}</div>
                <div class="pill pill--a">A: {{ pieceCounts.a }} ({{ capturedByA }} taken)</div>
                <div class="pill pill--b">B: {{ pieceCounts.b }} ({{ capturedByB }} taken)</div>
                <div class="pill" v-if="selected != null">Sel: {{ selected }}</div>
              </div>
            </div>

            <aside class="side">
              <div class="panel" v-if="gameMode === 'cpu'">
                <div class="panel__title">Campaign</div>
                <div class="panel__text">
                  Level <b>{{ cpuLevel }}</b> of {{ MAX_CPU_LEVEL }} — {{ cpuLevelLabel }}
                </div>
                <div class="levelBar">
                  <div class="levelBar__fill" :style="{ width: `${(cpuLevel / MAX_CPU_LEVEL) * 100}%` }"></div>
                </div>
                <div class="panel__text">Win to unlock the next level.</div>
              </div>

              <div class="panel">
                <div class="panel__title">Move helper</div>
                <div class="panel__text">
                  {{ selected == null ? 'Pick a piece.' : 'Pick a highlighted destination.' }}
                </div>
                <div class="panel__text" v-if="selected != null && legalMovesForSelected.length > 0">
                  Legal moves from selected: <b>{{ legalMovesForSelected.length }}</b>
                </div>
                <div class="panel__text" v-if="selected != null && legalMovesForSelected.length === 0">
                  No legal moves for selected.
                </div>
              </div>

              <div class="panel panel--tips">
                <div class="panel__title">Tips</div>
                <ul class="tips">
                  <li>When captures are available, you must capture.</li>
                  <li>Multi-jumps are generated automatically.</li>
                  <li>Kings can move in both directions.</li>
                </ul>
              </div>

              <div class="panel panel--bottom" v-if="mode === 'gameover'">
                <div class="panel__title">Game Over</div>
                <div class="gameover">
                  <template v-if="gameMode === 'cpu'">
                    <div class="winner" v-if="winner === 'A' && cpuLevel >= MAX_CPU_LEVEL">
                      You conquered all {{ MAX_CPU_LEVEL }} levels!
                    </div>
                    <div class="winner" v-else-if="winner === 'A'">
                      Level {{ cpuLevel }} complete!
                    </div>
                    <div class="winner" v-else>
                      Computer wins Level {{ cpuLevel }}
                    </div>
                    <div class="panel__text" v-if="winner === 'A' && cpuLevel < MAX_CPU_LEVEL">
                      Next up: Level {{ cpuLevel + 1 }} ({{ nextCpuLevelLabel }})
                    </div>
                    <div class="panel__text" v-else-if="winner === 'B'">
                      Retry Level {{ cpuLevel }} or return to the menu.
                    </div>
                  </template>
                  <div class="winner" v-else>
                    {{ winner === 'A' ? 'Player 1 (A)' : 'Player 2 (B)' }} wins!
                  </div>
                  <div class="gameover__stats">
                    <div class="gameover__stat" v-if="gameMode === 'cpu'">Level: <b>{{ cpuLevel }}/{{ MAX_CPU_LEVEL }}</b></div>
                    <div class="gameover__stat">Moves: <b>{{ moveCount }}</b></div>
                    <div class="gameover__stat">A captured: <b>{{ capturedByA }}</b></div>
                    <div class="gameover__stat">B captured: <b>{{ capturedByB }}</b></div>
                    <div class="gameover__stat">A pieces left: <b>{{ pieceCounts.a }}</b></div>
                    <div class="gameover__stat">B pieces left: <b>{{ pieceCounts.b }}</b></div>
                  </div>
                  <div class="gameover__actions">
                    <button
                      v-if="gameMode === 'cpu' && winner === 'A' && cpuLevel < MAX_CPU_LEVEL"
                      class="btn"
                      @click="continueToNextLevel"
                    >
                      Continue to Level {{ cpuLevel + 1 }}
                    </button>
                    <button
                      v-if="gameMode === 'cpu' && winner === 'B'"
                      class="btn"
                      @click="retryCpuLevel"
                    >
                      Retry Level {{ cpuLevel }}
                    </button>
                    <button
                      v-if="gameMode === 'cpu' && winner === 'A' && cpuLevel >= MAX_CPU_LEVEL"
                      class="btn"
                      @click="startGame('cpu')"
                    >
                      Play again from Level 1
                    </button>
                    <button v-if="gameMode !== 'cpu'" class="btn" @click="restart">Play again</button>
                    <button v-if="gameMode === 'cpu'" class="btn btn--ghost" @click="backToMenu">Main menu</button>
                  </div>
                </div>
              </div>
            </aside>
          </div>
        </div>
      </div>
      <div v-if="dragging && isDraggingMode" :style="dragGhostStyle" class="drag-ghost"></div>
    </section>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background: radial-gradient(circle at 20% 10%, rgba(80,150,255,0.12), transparent 40%),
    radial-gradient(circle at 80% 20%, rgba(160,80,255,0.10), transparent 42%),
    #060712;
  color: #eaf2ff;
}

.topbar {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 14px 18px;
}

.brand {
  display: flex;
  gap: 12px;
  align-items: center;
  min-width: 260px;
}

.logo {
  width: 42px;
  height: 42px;
  border-radius: 14px;
  display: grid;
  place-items: center;
  background: rgba(255,255,255,0.06);
  border: 1px solid rgba(255,255,255,0.10);
  box-shadow: 0 18px 45px rgba(0, 0, 0, 0.35);
}

.title {
  font-weight: 800;
  letter-spacing: 0.2px;
  font-size: 16px;
}

.subtitle {
  font-size: 12px;
  opacity: 0.8;
  margin-top: 2px;
}

.status {
  flex: 1;
  opacity: 0.95;
  font-size: 14px;
}

.status__hint {
  opacity: 0.75;
  font-weight: 500;
}

.actions {
  min-width: 140px;
  display: flex;
  justify-content: flex-end;
}

.btn {
  border: 1px solid rgba(255,255,255,0.16);
  background: linear-gradient(180deg, rgba(255,255,255,0.08), rgba(255,255,255,0.03));
  color: #eff5ff;
  border-radius: 12px;
  padding: 10px 14px;
  font-weight: 700;
  cursor: pointer;
  transition: transform 0.08s ease, border-color 0.2s ease;
}

.btn:active {
  transform: translateY(1px);
}

.btn--ghost {
  background: transparent;
}

.hero {
  position: relative;
  flex: 1;
  display: flex;
}

.bg {
  position: absolute;
  inset: 0;
  overflow: hidden;
}

.bg__canvas {
  display: block;
  width: 100%;
  height: 100%;
  filter: saturate(1.15) contrast(1.05);
}

.bg__overlay {
  position: absolute;
  inset: 0;
  background:
    radial-gradient(circle at 30% 25%, rgba(255,255,255,0.08), transparent 40%),
    radial-gradient(circle at 70% 60%, rgba(0,255,209,0.06), transparent 48%),
    linear-gradient(180deg, rgba(0,0,0,0.55), rgba(0,0,0,0.75));
}

.hero__content {
  position: relative;
  width: 100%;
  display: flex;
  justify-content: center;
  padding: 20px;
}

.menu {
  width: min(1040px, 100%);
  padding: 18px;
}

.menu__h2 {
  margin: 0 0 14px;
  font-size: 22px;
  letter-spacing: 0.2px;
}

.cards {
  display: grid;
  grid-template-columns: repeat(2, minmax(280px, 1fr));
  gap: 16px;
}

.card {
  text-align: left;
  padding: 16px;
  border-radius: 16px;
  cursor: pointer;
  background: rgba(255,255,255,0.06);
  border: 1px solid rgba(255,255,255,0.12);
  box-shadow: 0 25px 70px rgba(0, 0, 0, 0.25);
  color: #eff5ff;
  transition: transform 0.12s ease, border-color 0.2s ease, background 0.2s ease;
}

.card:hover {
  transform: translateY(-2px);
  border-color: rgba(255,255,255,0.22);
  background: rgba(255,255,255,0.08);
}

.card__title {
  font-size: 16px;
  font-weight: 850;
  margin-bottom: 6px;
}

.card__desc {
  opacity: 0.85;
  font-size: 13px;
  line-height: 1.35;
}

.card__meta {
  margin-top: 12px;
  opacity: 0.85;
  font-size: 13px;
}

.selectWrap {
  display: flex;
  gap: 10px;
  align-items: center;
}

.select {
  background: rgba(0,0,0,0.25);
  border: 1px solid rgba(255,255,255,0.18);
  color: #eff5ff;
  border-radius: 10px;
  padding: 8px 10px;
}

.menu__footer {
  margin-top: 16px;
}

.rules {
  display: grid;
  grid-template-columns: 1fr;
  gap: 4px;
  padding: 12px 14px;
  border-radius: 14px;
  background: rgba(0,0,0,0.22);
  border: 1px solid rgba(255,255,255,0.12);
  width: fit-content;
}

.rule {
  font-size: 13px;
}

.game {
  width: min(1200px, 100%);
}

.layout {
  display: grid;
  grid-template-columns: 520px 1fr;
  gap: 18px;
  align-items: start;
}

.boardWrap {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.board {
  display: grid;
  grid-template-columns: repeat(10, 1fr);
  width: min(500px, 92vw);
  aspect-ratio: 1 / 1;
  border-radius: 18px;
  overflow: hidden;
  background: rgba(0,0,0,0.2);
  border: 1px solid rgba(255,255,255,0.12);
}

.cell {
  position: relative;
  display: grid;
  place-items: center;
  user-select: none;
  touch-action: none;
}

.cell--light {
  background: rgba(255,255,255,0.04);
}

.cell--dark {
  background: rgba(10,14,28,0.55);
}

.cell--selected {
  outline: 2px solid rgba(120,210,255,0.75);
  outline-offset: -2px;
}

.cell--legal {
  background: radial-gradient(circle at center, rgba(0,255,209,0.25), rgba(0,0,0,0));
}

.cell--last-from {
  box-shadow: inset 0 0 0 2px rgba(255,200,50,0.35);
}

.cell--last-to {
  box-shadow: inset 0 0 0 2px rgba(255,200,50,0.5);
}

.pieceDot {
  width: 70%;
  height: 70%;
  border-radius: 999px;
  box-shadow: inset 0 0 0 2px rgba(255,255,255,0.18), 0 16px 35px rgba(0,0,0,0.45);
  transition: transform 0.15s ease, box-shadow 0.2s ease;
  pointer-events: none;
}

.cell:hover .pieceDot {
  transform: scale(1.08);
}

.cell:active .pieceDot {
  transform: scale(0.95);
}

.piece--a .pieceDot {
  background: radial-gradient(circle at 30% 25%, #ffffff 0%, #ffdb7a 18%, #ffb300 45%, #c98200 100%);
}

.piece--b .pieceDot {
  background: radial-gradient(circle at 30% 25%, #ffffff 0%, #9ff4ff 18%, #49d6ff 45%, #1c88b6 100%);
}

.piece--king {
  box-shadow: 0 0 0 2px rgba(255,255,255,0.24), 0 0 22px rgba(255,207,58,0.2), inset 0 0 0 2px rgba(255,255,255,0.18);
}

.controls {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.pill {
  padding: 9px 12px;
  border-radius: 999px;
  background: rgba(0,0,0,0.25);
  border: 1px solid rgba(255,255,255,0.12);
  font-size: 12px;
  opacity: 0.9;
}

.pill--a {
  border-color: rgba(255,200,50,0.3);
  color: #ffd966;
}

.pill--b {
  border-color: rgba(80,200,255,0.3);
  color: #66d9ff;
}

.side {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.panel {
  padding: 14px 16px;
  border-radius: 16px;
  background: rgba(255,255,255,0.05);
  border: 1px solid rgba(255,255,255,0.12);
}

.panel__title {
  font-weight: 850;
  margin-bottom: 8px;
}

.panel__text {
  font-size: 13px;
  opacity: 0.9;
  line-height: 1.35;
}

.panel--tips .tips {
  margin: 0;
  padding-left: 18px;
}

.tips li {
  font-size: 13px;
  opacity: 0.9;
  margin: 6px 0;
}

.panel--bottom {
  display: grid;
  gap: 10px;
}

.gameover {
  display: grid;
  gap: 12px;
}

.gameover__actions {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.levelBar {
  height: 8px;
  border-radius: 999px;
  background: rgba(255,255,255,0.08);
  overflow: hidden;
  margin: 10px 0;
}

.levelBar__fill {
  height: 100%;
  border-radius: 999px;
  background: linear-gradient(90deg, rgba(80,200,255,0.85), rgba(0,255,209,0.85));
  transition: width 0.35s ease;
}

.gameover__stats {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 4px 16px;
  font-size: 13px;
  opacity: 0.85;
}

.gameover__stat {
  line-height: 1.4;
}

.winner {
  font-size: 22px;
  font-weight: 900;
  text-align: center;
  background: linear-gradient(135deg, #ffd700, #ff8c00);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  padding: 8px 0;
}

.drag-ghost {
  transition: none;
}

@media (max-width: 980px) {
  .layout {
    grid-template-columns: 1fr;
  }

  .board {
    width: min(540px, 92vw);
    margin: 0 auto;
  }

  .cards {
    grid-template-columns: 1fr;
  }
}
</style>

