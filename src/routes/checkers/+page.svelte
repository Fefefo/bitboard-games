<script lang="ts">
	import Crown from '$lib/assets/Crown.svelte';
	import { tick } from 'svelte';

	const mask_l3 = 0x0e0e0e0e; // 0000_1110_0000_1110_0000_1110_0000_1110; << 3
	const mask_l5 = 0x00707070; // 0000_0000_0111_0000_0111_0000_0111_0000; << 5
	const mask_r3 = 0x70707070; // 0111_0000_0111_0000_0111_0000_0111_0000; >> 3
	const mask_r5 = 0x0e0e0e00; // 0000_1110_0000_1110_0000_1110_0000_0000; >> 5

	const first_row = 0xf0000000;
	const last_row = 0x0000000f;

	let bbb = $state(0x00000fff);
	let wbb = $state(0xfff00000);
	let kbb = $state(0x00000000);

	let whitePossibleMoves = $state(0);
	let blackPossibleMoves = $state(0);

	let abb = $derived(bbb | wbb);
	let fbb = $derived(~abb);

	let turn = $state(true);
	let selected = 0;

	let toKill = new Map<number, number>();

	let white_movers = $derived.by(() => {
		if (turn) return 0;

		const wkbb = wbb & kbb;
		let movers = 0;
		let tmp = (fbb << 4) & bbb;
		if (tmp) movers |= (((tmp & mask_l3) << 3) | ((tmp & mask_l5) << 5)) & wbb;
		tmp = (((fbb & mask_l3) << 3) | ((fbb & mask_l5) << 5)) & bbb;
		movers |= (tmp << 4) & wbb;
		if (wkbb) {
			let tmp = (fbb >> 4) & bbb;
			if (tmp) movers |= (((tmp & mask_r3) >> 3) | ((tmp & mask_r5) >> 5)) & wkbb;
			tmp = (((fbb & mask_r3) >> 3) | ((fbb & mask_r5) >> 5)) & bbb;
			movers |= (tmp >> 4) & wkbb;
		}
		if (movers != 0) return movers;

		movers = (fbb << 4) & wbb;
		movers |= ((fbb & mask_l3) << 3) & wbb;
		movers |= ((fbb & mask_l5) << 5) & wbb;
		if (wkbb) {
			movers |= (fbb >> 4) & wkbb;
			movers |= ((fbb & mask_r3) >> 3) & wkbb;
			movers |= ((fbb & mask_r5) >> 5) & wkbb;
		}
		return movers;
	});

	let black_movers = $derived.by(() => {
		if (!turn) return 0;

		const bkbb = bbb & kbb;
		let movers = 0;
		let tmp = (fbb >> 4) & wbb;
		if (tmp) movers |= (((tmp & mask_r3) >> 3) | ((tmp & mask_r5) >> 5)) & bbb;
		tmp = (((fbb & mask_r3) >> 3) | ((fbb & mask_r5) >> 5)) & wbb;
		movers |= (tmp >> 4) & bbb;
		if (bkbb) {
			let tmp = (fbb << 4) & wbb;
			if (tmp) movers |= (((tmp & mask_l3) << 3) | ((tmp & mask_l5) << 5)) & bkbb;
			tmp = (((fbb & mask_l3) << 3) | ((fbb & mask_l5) << 5)) & wbb;
			movers |= (tmp >> 4) & bkbb;
		}
		if (movers != 0) return movers;

		movers = (fbb >> 4) & bbb;
		movers |= ((fbb & mask_r3) >> 3) & bbb;
		movers |= ((fbb & mask_r5) >> 5) & bbb;
		if (bkbb) {
			movers |= (fbb << 4) & bkbb;
			movers |= ((fbb & mask_l3) << 3) & bkbb;
			movers |= ((fbb & mask_l5) << 5) & bkbb;
		}
		return movers;
	});

	function isOccupied(bb: number, row: number, col: number): boolean {
		const mask = 1 << (4 * row + col);
		return (mask & bb) != 0;
	}

	function showMovesWhite(row: number, col: number) {
		selected = 1 << (4 * row + col);
		const isK = (selected & kbb) != 0;
		whitePossibleMoves = 0;
		toKill.clear();

		let opponent = (selected >> 4) & bbb;
		if (opponent) {
			whitePossibleMoves = (((opponent & mask_r3) >> 3) | ((opponent & mask_r5) >> 5)) & fbb;
			if (whitePossibleMoves) {
				toKill.set(whitePossibleMoves, opponent);
			}
		}
		opponent = (((selected & mask_r3) >> 3) | ((selected & mask_r5) >> 5)) & bbb;
		let move = (opponent >> 4) & fbb;
		if (move) {
			whitePossibleMoves |= move;
			toKill.set(move, opponent);
		}
		if (isK) {
			opponent = (selected << 4) & bbb;
			if (opponent) {
				move = (((opponent & mask_l3) << 3) | ((opponent & mask_l5) << 5)) & fbb;
				whitePossibleMoves |= move;
				if (move) {
					toKill.set(move, opponent);
				}
			}
			opponent = (((selected & mask_l3) << 3) | ((selected & mask_l5) << 5)) & bbb;
			move = (opponent << 4) & fbb;
			if (move) {
				whitePossibleMoves |= move;
				toKill.set(move, opponent);
			}
		}

		if (whitePossibleMoves != 0) return;

		whitePossibleMoves = (selected >> 4) & fbb;
		if (selected & mask_r3) whitePossibleMoves |= (selected >> 3) & fbb;
		if (selected & mask_r5) whitePossibleMoves |= (selected >> 5) & fbb;
		if (!isK) return;
		whitePossibleMoves |= (selected << 4) & fbb;
		if (selected & mask_l3) whitePossibleMoves |= (selected << 3) & fbb;
		if (selected & mask_l5) whitePossibleMoves |= (selected << 5) & fbb;
	}

	function showMovesBlack(row: number, col: number) {
		selected = 1 << (4 * row + col);
		const isK = (selected & kbb) != 0;
		blackPossibleMoves = 0;
		toKill.clear();

		let opponent = (selected << 4) & wbb;
		if (opponent) {
			blackPossibleMoves = (((opponent & mask_l3) << 3) | ((opponent & mask_l5) << 5)) & fbb;
			if (blackPossibleMoves) {
				toKill.set(blackPossibleMoves, opponent);
			}
		}
		opponent = (((selected & mask_l3) << 3) | ((selected & mask_l5) << 5)) & wbb;
		let move = (opponent << 4) & fbb;
		if (move) {
			blackPossibleMoves |= move;
			toKill.set(move, opponent);
		}
		if (isK) {
			opponent = (selected >> 4) & wbb;
			if (opponent) {
				move = (((opponent & mask_r3) >> 3) | ((opponent & mask_r5) >> 5)) & fbb;
				blackPossibleMoves = move;
				toKill.set(move, opponent);
			}
			opponent = (((selected & mask_r3) >> 3) | ((selected & mask_r5) >> 5)) & wbb;
			move = (opponent >> 4) & fbb;
			if (move) {
				blackPossibleMoves |= move;
				toKill.set(move, opponent);
			}
		}
		if (blackPossibleMoves != 0) return;

		blackPossibleMoves = (selected << 4) & fbb;
		if (selected & mask_l3) blackPossibleMoves |= (selected << 3) & fbb;
		if (selected & mask_l5) blackPossibleMoves |= (selected << 5) & fbb;
		if (!isK) return;
		blackPossibleMoves |= (selected >> 4) & fbb;
		if (selected & mask_r3) blackPossibleMoves |= (selected >> 3) & fbb;
		if (selected & mask_r5) blackPossibleMoves |= (selected >> 5) & fbb;
	}

	function move(bb: number, row: number, col: number): number {
		// bb ^= selected;
		const mask = (1 << (4 * row + col)) | selected;
		bb ^= mask;
		if ((selected & kbb) != 0) {
			// kbb ^= selected;
			// kbb |= mask;
			kbb ^= mask;
		}
		whitePossibleMoves = 0;
		blackPossibleMoves = 0;
		selected = 0;
		turn = !turn;
		return bb;
	}
</script>

<div class="bg-amber-900 p-4 w-fit">
	{#key abb}
		<div class="flex flex-col-reverse">
			{#each { length: 8 } as _, i}
				<div class="flex flex-row">
					{#each { length: 4 } as _, j}
						{@const isEmpty = !isOccupied(abb, i, j)}
						{@const isBlack = isOccupied(bbb, i, j)}
						{@const isPossibleWhite = isOccupied(whitePossibleMoves, i, j)}
						{@const isPossibleBlack = isOccupied(blackPossibleMoves, i, j)}
						{@const isPossible = isPossibleWhite || isPossibleBlack}
						{@const isKing = isOccupied(kbb, i, j)}
						<div class="w-20 h-10 flex">
							<button
								aria-label="real-cell"
								class="w-10 h-10 bg-amber-600 p-1"
								onclick={() => {
									if (isOccupied(white_movers, i, j)) {
										showMovesWhite(i, j);
									} else if (isOccupied(black_movers, i, j)) {
										showMovesBlack(i, j);
									} else if (isPossibleBlack) {
										bbb = move(bbb, i, j);
										kbb |= bbb & first_row;
										if (white_movers == 0 && !turn) {
											alert('black won');
										}
										let dead = toKill.get(1 << (4 * i + j)) ?? 0;
										wbb ^= dead;
										if (kbb & dead) kbb ^= dead;
									} else if (isPossibleWhite) {
										wbb = move(wbb, i, j);
										kbb |= wbb & last_row;
										if (black_movers == 0 && turn) {
											alert('white won');
										}
										let dead = toKill.get(1 << (4 * i + j)) ?? 0;
										bbb ^= dead;
										if (kbb & dead) kbb ^= dead;
									} else {
										selected = 0;
										blackPossibleMoves = 0;
										whitePossibleMoves = 0;
									}
								}}
								class:order-2={i % 2 == 1}
							>
								<div
									class="flex w-8 h-8 rounded-full justify-center items-center"
									class:bg-black={isBlack}
									class:bg-white={!isEmpty && !isBlack}
									class:bg-gray-700={isPossibleBlack}
									class:bg-gray-200={isPossibleWhite}
									class:opacity-70={isPossible}
									class:border-king={isKing}
								>
									{#if isKing}
										<Crown></Crown>
									{/if}
									<!-- {isOccupied(mask_l3, i, j) ? 'l3' : ''}
									{isOccupied(mask_r3, i, j) ? 'r3' : ''}
									{isOccupied(mask_l5, i, j) ? 'l5' : ''}
									{isOccupied(mask_r5, i, j) ? 'r5' : ''} -->
								</div>
							</button>
							<button
								class="bg-amber-100 w-10 h-10"
								class:order-1={i % 2 == 1}
								aria-label="fake-cell"
								onclick={() => {
									selected = 0;
									blackPossibleMoves = 0;
									whitePossibleMoves = 0;
								}}
							>
							</button>
						</div>
					{/each}
				</div>
			{/each}
		</div>
	{/key}
</div>

<style>
	.border-king {
		border: 2px solid #e6ae3d !important;
	}
</style>
