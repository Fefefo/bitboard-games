<script lang="ts">
	import * as Dialog from '$lib/components/ui/dialog/index.js';
	import { Button } from '$lib/components/ui/button/index.js';
	const CELL = {
		EMPTY: 0,
		RED: 1,
		YELLOW: 2
	};

	const directionsForWin = [
		[0, 1],
		[1, 0],
		[1, 1],
		[1, -1]
	];

	let boardLogic: number[][] = [];
	let boardButtons: HTMLButtonElement[][] = [];
	let turn = 1;
	let win = false;

	function bindBoard(node: HTMLButtonElement, params: { r: number; c: number }) {
		const { r, c } = params;
		if (!boardButtons[r]) boardButtons[r] = [];
		boardButtons[r][c] = node as HTMLButtonElement;
	}

	function putPiece(x: number, y: number) {
		if (boardLogic[x][y] != CELL.EMPTY) return;
		while (true) {
			if (boardLogic[x][y + 1] != undefined && boardLogic[x][y + 1] == CELL.EMPTY) y += 1;
			else break;
		}
		boardLogic[x][y] = turn;
		win = checkWin(boardLogic, turn);
		if (win) return;
		turn = turn == CELL.RED ? CELL.YELLOW : CELL.RED;
	}

	function checkWin(board: number[][], turn: number): boolean {
		for (let direction of directionsForWin) {
			for (let i = 0; i < board.length - direction[0] * 3; i++) {
				for (
					let j = 0 + (direction[1] == -1 ? 3 : 0);
					j < board[i].length - (direction[1] != -1 ? direction[1] : 0) * 3;
					j++
				) {
					if (
						board[i][j] == turn &&
						board[i + direction[0] * 1][j + direction[1] * 1] == turn &&
						board[i + direction[0] * 2][j + direction[1] * 2] == turn &&
						board[i + direction[0] * 3][j + direction[1] * 3] == turn
					) {
						return true;
					}
				}
			}
		}
		return false;
	}

	function setupBoard() {
		boardLogic = [];
		for (let i = 0; i < 7; i++) {
			let arrayRow: number[] = [];
			for (let j = 0; j < 6; j++) {
				arrayRow.push(CELL.EMPTY);
			}
			boardLogic.push(arrayRow);
		}
	}

	function assignScore(board: number[][], turn: number): number[] {
		let scores = [0, 0, 0, 4, 0, 0, 0]; // let's assign 4 score points to the column 4
		return scores;
	}

	setupBoard();
</script>

<div class="bg-blue-500 p-4 w-fit">
	<div class="flex flex-row gap-4">
		{#each boardLogic as rowLogic, i}
			<div class="flex flex-col gap-4">
				{#each rowLogic as cellLogic, j}
					<button
						aria-label="cell-[${i}]-[${j}]"
						class="flex w-10 h-10 border bg-white rounded-full"
						use:bindBoard={{ r: i, c: j }}
						onclick={() => {
							putPiece(i, j);
						}}
						class:bg-white={cellLogic == CELL.EMPTY}
						class:bg-red-500={cellLogic == CELL.RED}
						class:bg-yellow-500={cellLogic == CELL.YELLOW}
					></button>
				{/each}
			</div>
		{/each}
	</div>
</div>

<Dialog.Root open={win}>
	<Dialog.Content>
		<Dialog.Header>
			<Dialog.Title>{turn == CELL.RED ? 'Red' : 'Yellow'} player won!</Dialog.Title>
		</Dialog.Header>
		<Button
			onclick={() => {
				win = false;
				setupBoard();
			}}>Play again!</Button
		>
	</Dialog.Content>
</Dialog.Root>
