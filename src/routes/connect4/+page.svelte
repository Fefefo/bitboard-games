<script lang="ts">
	import * as Dialog from '$lib/components/ui/alert-dialog/index.js';
	import { Button } from '$lib/components/ui/button/index.js';

	let red_bitboard: bigint = $state(0n);
	let yellow_bitboard: bigint = $state(0n);
	let both_bitboard: bigint = $derived(red_bitboard | yellow_bitboard);

	// const WIDTH = 7;
	const HEIGTH = 7; // one more than actual height of the board
	const BIG_HEIGHT = BigInt(HEIGTH);

	let turn = $state(true);
	let win = $state(false);

	function isOccupied(bb: bigint, col: number, row: number): boolean {
		const mask = 1n << BigInt(HEIGTH * col + row);
		return (mask & bb) != 0n;
	}

	// function isPlayable(bb: bigint, col: number): boolean {
	// 	return !isOccupied(bb, col, HEIGTH - 1);
	// }

	function columnMask(col: number): bigint {
		return ((1n << 7n) - 1n) << BigInt(col * 7);
	}

	// function currentBitboard(turn: boolean): bigint {
	// 	if (turn) return red_bitboard;
	// 	return yellow_bitboard;
	// }

	function bottomMask(col: number): bigint {
		return 1n << BigInt(col * 7);
	}

	function putPiece(col: number) {
		const move = (both_bitboard + bottomMask(col)) & columnMask(col);
		if (turn) {
			red_bitboard |= move;
			win = checkWin(red_bitboard);
			if (win) return;
		} else {
			yellow_bitboard |= move;
			win = checkWin(yellow_bitboard);
			if (win) return;
		}
		turn = !turn;
	}

	function checkWin(bb: bigint): boolean {
		// vertical check
		let mask = bb & (bb >> 1n);
		if (mask & (mask >> 2n)) return true;

		// horizontal check
		mask = bb & (bb >> BIG_HEIGHT);
		if (mask & (mask >> (2n * BIG_HEIGHT))) return true;

		// diagonal / check
		mask = bb & (bb >> (BIG_HEIGHT + 1n));
		if (mask & (mask >> (2n * (BIG_HEIGHT + 1n)))) return true;

		// diagonal \ check
		mask = bb & (bb >> (BIG_HEIGHT - 1n));
		if (mask & (mask >> (2n * (BIG_HEIGHT + -1n)))) return true;

		return false;
	}
</script>

<div class="bg-blue-500 p-4 w-fit">
	<div class="flex gap-4 flex-col-reverse">
		{#key both_bitboard}
			{#each { length: 6 } as _, i}
				<div class="flex flex-row gap-4">
					{#each { length: 7 } as _, j}
						{@const isEmpty = !isOccupied(both_bitboard, j, i)}
						{@const isRed = isOccupied(red_bitboard, j, i)}
						<button
							aria-label="cell"
							class="flex w-10 h-10 border rounded-full text-black"
							onclick={() => {
								if (isEmpty) putPiece(j);
							}}
							class:bg-white={isEmpty}
							class:bg-red-500={isRed}
							class:bg-yellow-500={!isEmpty && !isRed}
						></button>
					{/each}
				</div>
			{/each}
		{/key}
	</div>
</div>

<Dialog.Root open={win}>
	<Dialog.Content>
		<Dialog.Header>
			<Dialog.Title>{turn == true ? 'Red' : 'Yellow'} player won!</Dialog.Title>
		</Dialog.Header>
		<Button
			onclick={() => {
				win = false;
				red_bitboard = 0n;
				yellow_bitboard = 0n;
			}}>Play again!</Button
		>
	</Dialog.Content>
</Dialog.Root>
