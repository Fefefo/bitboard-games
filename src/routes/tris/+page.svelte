<script lang="ts">
	import * as Dialog from '$lib/components/ui/alert-dialog/index.js';
	import { Button } from '$lib/components/ui/button/index.js';
	let circle_bitboard = $state(0);
	let cross_bitboard = $state(0);
	let both_bitboard = $derived(circle_bitboard | cross_bitboard);

	let WIDTH = 3;
	let HEIGHT = 4;
	let turn = $state(true);
	let win = $state(false);

	function isOccupied(bb: number, col: number, row: number): boolean {
		const mask = 1 << (HEIGHT * col + row);
		return (mask & bb) != 0;
	}

	function columnMask(col: number): number {
		return ((1 << 7) - 1) << (col * HEIGHT);
	}

	function bottomMask(col: number): number {
		return 1 << (col * HEIGHT);
	}

	function checkWin(bb: number): boolean {
		// vertical check
		let mask = bb & (bb >> 1);
		if (mask & (mask >> 1)) return true;

		// horizontal check
		mask = bb & (bb >> HEIGHT);
		if (mask & (mask >> HEIGHT)) return true;

		// diagonal / check
		mask = bb & (bb >> (HEIGHT + 1));
		if (mask & (mask >> (HEIGHT + 1))) return true;

		// diagonal \ check
		mask = bb & (bb >> (HEIGHT - 1));
		if (mask & (mask >> (HEIGHT + -1))) return true;

		return false;
	}

	function putPiece(col: number, row: number) {
		const move = 1 << (HEIGHT * col + row);
		if (turn) {
			cross_bitboard |= move;
			win = checkWin(cross_bitboard);
			if (win) return;
		} else {
			circle_bitboard |= move;
			win = checkWin(circle_bitboard);
			if (win) return;
		}
		turn = !turn;
	}
</script>

<div class="bg-white p-1 w-fit">
	<div class="flex flex-col-reverse">
		{#key both_bitboard}
			{#each { length: 3 } as _, i}
				<div class="flex flex-row">
					{#each { length: 3 } as _, j}
						{@const isEmpty = !isOccupied(both_bitboard, j, i)}
						{@const isCircle = isOccupied(circle_bitboard, j, i)}
						<button
							aria-label="cell"
							class="flex w-10 h-10 text-black bg-white border-black justify-center items-center"
							onclick={() => {
								if (isEmpty) putPiece(j, i);
							}}
							class:border-t={i != 2}
							class:border-r={j != 2}
							>{isEmpty ? '' : isCircle ? 'o' : 'x'}
						</button>
					{/each}
				</div>
			{/each}
		{/key}
	</div>
</div>

<Dialog.Root open={win}>
	<Dialog.Content>
		<Dialog.Header>
			<Dialog.Title>{turn == true ? 'Cross' : 'Circle'} player won!</Dialog.Title>
		</Dialog.Header>
		<Button
			onclick={() => {
				win = false;
				cross_bitboard = 0;
				circle_bitboard = 0;
			}}>Play again!</Button
		>
	</Dialog.Content>
</Dialog.Root>
