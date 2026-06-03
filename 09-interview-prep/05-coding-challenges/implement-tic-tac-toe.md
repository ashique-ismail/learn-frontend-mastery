# Implement Tic-Tac-Toe

## Core Logic

```ts
type Player = 'X' | 'O';
type Cell = Player | null;
type Board = Cell[]; // 9 cells, indices 0-8

const WINNING_COMBINATIONS = [
  [0, 1, 2], // top row
  [3, 4, 5], // middle row
  [6, 7, 8], // bottom row
  [0, 3, 6], // left col
  [1, 4, 7], // center col
  [2, 5, 8], // right col
  [0, 4, 8], // diagonal
  [2, 4, 6], // anti-diagonal
];

function calculateWinner(board: Board): { player: Player; line: number[] } | null {
  for (const [a, b, c] of WINNING_COMBINATIONS) {
    if (board[a] && board[a] === board[b] && board[a] === board[c]) {
      return { player: board[a] as Player, line: [a, b, c] };
    }
  }
  return null;
}
```

---

## Game State with `useReducer`

```ts
interface GameState {
  board: Board;
  currentPlayer: Player;
  winner: { player: Player; line: number[] } | null;
  isDraw: boolean;
  moveCount: number;
  history: Board[]; // for undo
}

type GameAction =
  | { type: 'MOVE'; index: number }
  | { type: 'RESET' }
  | { type: 'UNDO' };

const initialState: GameState = {
  board: Array(9).fill(null),
  currentPlayer: 'X',
  winner: null,
  isDraw: false,
  moveCount: 0,
  history: [],
};

function gameReducer(state: GameState, action: GameAction): GameState {
  switch (action.type) {
    case 'MOVE': {
      const { board, currentPlayer, winner, isDraw, history } = state;
      if (winner || isDraw || board[action.index]) return state; // invalid move

      const newBoard = [...board];
      newBoard[action.index] = currentPlayer;

      const newWinner = calculateWinner(newBoard);
      const newMoveCount = state.moveCount + 1;
      const newIsDraw = !newWinner && newMoveCount === 9;

      return {
        board: newBoard,
        currentPlayer: currentPlayer === 'X' ? 'O' : 'X',
        winner: newWinner,
        isDraw: newIsDraw,
        moveCount: newMoveCount,
        history: [...history, board],
      };
    }
    case 'RESET':
      return initialState;
    case 'UNDO': {
      if (state.history.length === 0) return state;
      const previousBoard = state.history[state.history.length - 1];
      return {
        ...state,
        board: previousBoard,
        currentPlayer: state.currentPlayer === 'X' ? 'O' : 'X',
        winner: calculateWinner(previousBoard),
        isDraw: false,
        moveCount: state.moveCount - 1,
        history: state.history.slice(0, -1),
      };
    }
    default:
      return state;
  }
}
```

---

## React Component

```tsx
function TicTacToe() {
  const [state, dispatch] = useReducer(gameReducer, initialState);
  const { board, currentPlayer, winner, isDraw } = state;

  const winningLine = winner?.line ?? [];

  function getStatus() {
    if (winner) return `Player ${winner.player} wins! 🎉`;
    if (isDraw) return "It's a draw!";
    return `Player ${currentPlayer}'s turn`;
  }

  return (
    <div className="game">
      <p className="status" aria-live="polite">{getStatus()}</p>

      {/* Board */}
      <div
        className="board"
        role="grid"
        aria-label="Tic Tac Toe board"
      >
        {board.map((cell, index) => (
          <button
            key={index}
            role="gridcell"
            className={`cell ${winningLine.includes(index) ? 'winning' : ''}`}
            onClick={() => dispatch({ type: 'MOVE', index })}
            disabled={!!cell || !!winner || isDraw}
            aria-label={`Cell ${index + 1}: ${cell ?? 'empty'}`}
          >
            {cell}
          </button>
        ))}
      </div>

      {/* Controls */}
      <div className="controls">
        <button onClick={() => dispatch({ type: 'UNDO' })} disabled={state.history.length === 0}>
          Undo
        </button>
        <button onClick={() => dispatch({ type: 'RESET' })}>
          New Game
        </button>
      </div>
    </div>
  );
}
```

---

## CSS

```css
.board {
  display: grid;
  grid-template-columns: repeat(3, 100px);
  gap: 4px;
}

.cell {
  width: 100px;
  height: 100px;
  font-size: 2.5rem;
  font-weight: bold;
  background: #f8fafc;
  border: 2px solid #e2e8f0;
  cursor: pointer;
  transition: background 0.1s;
}

.cell:hover:not(:disabled) {
  background: #e0e7ff;
}

.cell.winning {
  background: #bbf7d0;
  border-color: #16a34a;
}

.cell:disabled {
  cursor: default;
}
```

---

## Extension: Minimax AI

```ts
function minimax(board: Board, isMaximizing: boolean, depth: number = 0): number {
  const winner = calculateWinner(board);
  if (winner?.player === 'X') return 10 - depth;
  if (winner?.player === 'O') return depth - 10;
  if (board.every(Boolean)) return 0; // draw

  const available = board.map((c, i) => (c ? null : i)).filter(i => i !== null) as number[];

  if (isMaximizing) {
    let best = -Infinity;
    for (const move of available) {
      const newBoard = [...board];
      newBoard[move] = 'X';
      best = Math.max(best, minimax(newBoard, false, depth + 1));
    }
    return best;
  } else {
    let best = Infinity;
    for (const move of available) {
      const newBoard = [...board];
      newBoard[move] = 'O';
      best = Math.min(best, minimax(newBoard, true, depth + 1));
    }
    return best;
  }
}

function getBestMove(board: Board): number {
  let bestScore = -Infinity;
  let bestMove = -1;
  board.forEach((cell, index) => {
    if (!cell) {
      const newBoard = [...board];
      newBoard[index] = 'X';
      const score = minimax(newBoard, false);
      if (score > bestScore) { bestScore = score; bestMove = index; }
    }
  });
  return bestMove;
}
```

---

## Common Interview Questions

**Q: Why use `useReducer` instead of multiple `useState` calls?**
Game state has closely related fields (board, current player, winner, draw status) that change together. `useReducer` keeps updates atomic — you can't have a stale board with a correct winner. It also makes the logic testable independently from the UI.

**Q: How does minimax work?**
It recursively simulates all possible games from the current position, assigning scores (+10 for X win, -10 for O win, 0 for draw). X maximizes the score, O minimizes it. The AI picks the move that leads to the best guaranteed outcome.

**Q: How would you add score tracking across multiple games?**
Store `{ X: 0, O: 0, draws: 0 }` in a separate state. Update it when a game ends in the `RESET` action (check winner before resetting) or a separate `RECORD_RESULT` action.
