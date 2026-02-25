<script lang="ts">
	import { onMount } from 'svelte';

	type Player = 'X' | 'O';
	type Cell = Player | null;

	let board: Cell[] = $state(Array(9).fill(null));
	let currentPlayer: Player = $state('X');
	let winner: Player | null = $state(null);
	let isDraw: boolean = $state(false);
	let winningLine: number[] = $state([]);
	let scores = $state({ X: 0, O: 0, draws: 0 });
	let isAI = $state(true);
	let aiThinking = $state(false);
	let moveHistory: number[] = $state([]);
	let animatingCell: number | null = $state(null);

	const winPatterns = [
		[0, 1, 2], [3, 4, 5], [6, 7, 8],
		[0, 3, 6], [1, 4, 7], [2, 5, 8],
		[0, 4, 8], [2, 4, 6]
	];

	function checkWinner(b: Cell[]): { winner: Player | null; line: number[] } {
		for (const pattern of winPatterns) {
			const [a, bIdx, c] = pattern;
			if (b[a] && b[a] === b[bIdx] && b[a] === b[c]) {
				return { winner: b[a] as Player, line: pattern };
			}
		}
		return { winner: null, line: [] };
	}

	function minimax(b: Cell[], depth: number, isMaximizing: boolean, alpha: number, beta: number): number {
		const result = checkWinner(b);
		if (result.winner === 'O') return 10 - depth;
		if (result.winner === 'X') return depth - 10;
		if (b.every((cell) => cell !== null)) return 0;

		if (isMaximizing) {
			let best = -Infinity;
			for (let i = 0; i < 9; i++) {
				if (b[i] === null) {
					b[i] = 'O';
					best = Math.max(best, minimax(b, depth + 1, false, alpha, beta));
					b[i] = null;
					alpha = Math.max(alpha, best);
					if (beta <= alpha) break;
				}
			}
			return best;
		} else {
			let best = Infinity;
			for (let i = 0; i < 9; i++) {
				if (b[i] === null) {
					b[i] = 'X';
					best = Math.min(best, minimax(b, depth + 1, true, alpha, beta));
					b[i] = null;
					beta = Math.min(beta, best);
					if (beta <= alpha) break;
				}
			}
			return best;
		}
	}

	function getBestMove(): number {
		let bestScore = -Infinity;
		let bestMove = -1;
		const b = [...board];
		for (let i = 0; i < 9; i++) {
			if (b[i] === null) {
				b[i] = 'O';
				const score = minimax(b, 0, false, -Infinity, Infinity);
				b[i] = null;
				if (score > bestScore) {
					bestScore = score;
					bestMove = i;
				}
			}
		}
		return bestMove;
	}

	async function makeMove(index: number) {
		if (board[index] || winner || isDraw || aiThinking) return;

		animatingCell = index;
		board[index] = currentPlayer;
		moveHistory = [...moveHistory, index];

		const result = checkWinner(board);
		if (result.winner) {
			winner = result.winner;
			winningLine = result.line;
			scores[result.winner]++;
			return;
		}

		if (board.every((cell) => cell !== null)) {
			isDraw = true;
			scores.draws++;
			return;
		}

		currentPlayer = currentPlayer === 'X' ? 'O' : 'X';

		if (isAI && currentPlayer === 'O') {
			aiThinking = true;
			await new Promise((r) => setTimeout(r, 400));
			const aiMove = getBestMove();
			if (aiMove !== -1) {
				animatingCell = aiMove;
				board[aiMove] = 'O';
				moveHistory = [...moveHistory, aiMove];

				const aiResult = checkWinner(board);
				if (aiResult.winner) {
					winner = aiResult.winner;
					winningLine = aiResult.line;
					scores[aiResult.winner]++;
				} else if (board.every((cell) => cell !== null)) {
					isDraw = true;
					scores.draws++;
				} else {
					currentPlayer = 'X';
				}
			}
			aiThinking = false;
		}
	}

	function resetGame() {
		board = Array(9).fill(null);
		currentPlayer = 'X';
		winner = null;
		isDraw = false;
		winningLine = [];
		moveHistory = [];
		animatingCell = null;
		aiThinking = false;
	}

	function resetScores() {
		scores = { X: 0, O: 0, draws: 0 };
		resetGame();
	}

	function toggleMode() {
		isAI = !isAI;
		resetScores();
	}

	function getStatus(): string {
		if (winner) return `${winner} wins! 🎉`;
		if (isDraw) return "It's a draw! 🤝";
		if (aiThinking) return 'AI is thinking...';
		return `${currentPlayer}'s turn`;
	}

	function getCellClass(index: number): string {
		let cls = 'cell';
		if (board[index] === 'X') cls += ' x';
		if (board[index] === 'O') cls += ' o';
		if (winningLine.includes(index)) cls += ' winning';
		if (!board[index] && !winner && !isDraw) cls += ' hoverable';
		return cls;
	}
</script>

<div class="app">
	<div class="container">
		<h1>Tic Tac Toe</h1>

		<div class="mode-toggle">
			<button class="mode-btn" class:active={isAI} onclick={toggleMode}>
				{isAI ? '🤖 vs AI' : '👥 2 Players'}
			</button>
		</div>

		<div class="scoreboard">
			<div class="score x-score">
				<span class="score-label">X</span>
				<span class="score-value">{scores.X}</span>
			</div>
			<div class="score draw-score">
				<span class="score-label">Draw</span>
				<span class="score-value">{scores.draws}</span>
			</div>
			<div class="score o-score">
				<span class="score-label">O</span>
				<span class="score-value">{scores.O}</span>
			</div>
		</div>

		<div class="status" class:winner-status={!!winner} class:draw-status={isDraw}>
			{getStatus()}
		</div>

		<div class="board">
			{#each board as cell, i}
				<button
					class={getCellClass(i)}
					onclick={() => makeMove(i)}
					disabled={!!board[i] || !!winner || isDraw || aiThinking}
					aria-label={`Cell ${i + 1}`}
				>
					{#if cell === 'X'}
						<svg class="mark" viewBox="0 0 100 100">
							<line x1="20" y1="20" x2="80" y2="80" />
							<line x1="80" y1="20" x2="20" y2="80" />
						</svg>
					{:else if cell === 'O'}
						<svg class="mark" viewBox="0 0 100 100">
							<circle cx="50" cy="50" r="30" />
						</svg>
					{/if}
				</button>
			{/each}
		</div>

		<div class="actions">
			<button class="btn btn-primary" onclick={resetGame}>New Game</button>
			<button class="btn btn-secondary" onclick={resetScores}>Reset All</button>
		</div>
	</div>
</div>

<style>
	@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800&display=swap');

	:global(body) {
		margin: 0;
		padding: 0;
		font-family: 'Inter', system-ui, sans-serif;
		background: #0f0f1a;
		color: #e0e0e0;
		min-height: 100vh;
		overflow: hidden;
	}

	.app {
		display: flex;
		justify-content: center;
		align-items: center;
		min-height: 100vh;
		background: radial-gradient(ellipse at top, #1a1a2e 0%, #0f0f1a 70%);
	}

	.container {
		text-align: center;
		padding: 2rem;
	}

	h1 {
		font-size: 2.5rem;
		font-weight: 800;
		margin: 0 0 1.5rem;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		-webkit-background-clip: text;
		-webkit-text-fill-color: transparent;
		background-clip: text;
		letter-spacing: -0.02em;
	}

	.mode-toggle {
		margin-bottom: 1.5rem;
	}

	.mode-btn {
		background: rgba(255, 255, 255, 0.05);
		border: 1px solid rgba(255, 255, 255, 0.1);
		color: #e0e0e0;
		padding: 0.6rem 1.5rem;
		border-radius: 50px;
		font-size: 0.95rem;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.2s ease;
		font-family: inherit;
	}

	.mode-btn:hover {
		background: rgba(255, 255, 255, 0.1);
		border-color: rgba(255, 255, 255, 0.2);
	}

	.scoreboard {
		display: flex;
		gap: 1.5rem;
		justify-content: center;
		margin-bottom: 1.5rem;
	}

	.score {
		display: flex;
		flex-direction: column;
		align-items: center;
		padding: 0.6rem 1.5rem;
		border-radius: 12px;
		background: rgba(255, 255, 255, 0.03);
		border: 1px solid rgba(255, 255, 255, 0.06);
		min-width: 70px;
	}

	.score-label {
		font-size: 0.8rem;
		font-weight: 600;
		text-transform: uppercase;
		letter-spacing: 0.05em;
		opacity: 0.6;
	}

	.score-value {
		font-size: 1.5rem;
		font-weight: 800;
	}

	.x-score .score-value { color: #667eea; }
	.o-score .score-value { color: #f093fb; }
	.draw-score .score-value { color: #888; }

	.status {
		font-size: 1.2rem;
		font-weight: 600;
		margin-bottom: 1.5rem;
		padding: 0.5rem 1rem;
		border-radius: 8px;
		transition: all 0.3s ease;
	}

	.winner-status {
		color: #ffd700;
		animation: pulse 1s ease-in-out infinite;
	}

	.draw-status {
		color: #aaa;
	}

	@keyframes pulse {
		0%, 100% { opacity: 1; }
		50% { opacity: 0.7; }
	}

	.board {
		display: grid;
		grid-template-columns: repeat(3, 1fr);
		gap: 8px;
		max-width: 360px;
		margin: 0 auto 2rem;
		padding: 12px;
		background: rgba(255, 255, 255, 0.03);
		border-radius: 20px;
		border: 1px solid rgba(255, 255, 255, 0.06);
	}

	.cell {
		aspect-ratio: 1;
		background: rgba(255, 255, 255, 0.04);
		border: 1px solid rgba(255, 255, 255, 0.08);
		border-radius: 12px;
		cursor: pointer;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 15px;
		transition: all 0.2s ease;
		position: relative;
		overflow: hidden;
	}

	.cell:disabled {
		cursor: default;
	}

	.cell.hoverable:hover {
		background: rgba(255, 255, 255, 0.08);
		border-color: rgba(255, 255, 255, 0.15);
		transform: scale(1.02);
	}

	.cell.winning {
		background: rgba(255, 215, 0, 0.1);
		border-color: rgba(255, 215, 0, 0.3);
		animation: winGlow 1s ease-in-out infinite;
	}

	@keyframes winGlow {
		0%, 100% { box-shadow: 0 0 10px rgba(255, 215, 0, 0.1); }
		50% { box-shadow: 0 0 25px rgba(255, 215, 0, 0.2); }
	}

	.mark {
		width: 100%;
		height: 100%;
		fill: none;
		stroke-linecap: round;
		stroke-linejoin: round;
	}

	.cell.x .mark {
		stroke: #667eea;
		stroke-width: 8;
		animation: drawX 0.3s ease-out forwards;
	}

	.cell.o .mark {
		stroke: #f093fb;
		stroke-width: 8;
		animation: drawO 0.4s ease-out forwards;
	}

	.cell.winning.x .mark { stroke: #ffd700; }
	.cell.winning.o .mark { stroke: #ffd700; }

	@keyframes drawX {
		from {
			stroke-dasharray: 100;
			stroke-dashoffset: 100;
		}
		to {
			stroke-dashoffset: 0;
		}
	}

	@keyframes drawO {
		from {
			stroke-dasharray: 200;
			stroke-dashoffset: 200;
		}
		to {
			stroke-dashoffset: 0;
		}
	}

	.actions {
		display: flex;
		gap: 1rem;
		justify-content: center;
	}

	.btn {
		padding: 0.7rem 1.8rem;
		border-radius: 10px;
		font-size: 0.95rem;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.2s ease;
		border: none;
		font-family: inherit;
	}

	.btn-primary {
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		color: white;
	}

	.btn-primary:hover {
		transform: translateY(-2px);
		box-shadow: 0 8px 25px rgba(102, 126, 234, 0.3);
	}

	.btn-secondary {
		background: rgba(255, 255, 255, 0.06);
		color: #aaa;
		border: 1px solid rgba(255, 255, 255, 0.1);
	}

	.btn-secondary:hover {
		background: rgba(255, 255, 255, 0.1);
		color: #e0e0e0;
	}

	@media (max-width: 480px) {
		.container { padding: 1rem; }
		h1 { font-size: 2rem; }
		.board { max-width: 300px; gap: 6px; padding: 8px; }
		.scoreboard { gap: 0.8rem; }
		.score { padding: 0.5rem 1rem; min-width: 60px; }
	}
</style>
