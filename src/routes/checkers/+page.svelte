<script lang="ts">
	import Crown from '$lib/assets/Crown.svelte';

	const mask_l3 = 0x0e0e0e0e; // 0000_1110_0000_1110_0000_1110_0000_1110; << 3
	const mask_l5 = 0x00707070; // 0000_0000_0111_0000_0111_0000_0111_0000; << 5
	const mask_r3 = 0x70707070; // 0111_0000_0111_0000_0111_0000_0111_0000; >> 3
	const mask_r5 = 0x0e0e0e00; // 0000_1110_0000_1110_0000_1110_0000_0000; >> 5

	let bbb = $state(0x00000fff);
	let wbb = $state(0xfff00000);
	let kbb = $state(0x00000000);
	let abb = $derived(bbb | wbb);
	let fbb = $derived(~abb);

	let turn = $state(false);

	let white_movers = $derived.by(() => {
		if (!turn) return 0;

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

	// let white_jumpers = $derived.by(() => {
	// 	if (!turn) return 0;

	// 	const wkbb = wbb & kbb;
	// 	let movers = 0;
	// 	let tmp = (fbb << 4) & bbb;
	// 	if (tmp) movers |= (((tmp & mask_l3) << 3) | ((tmp & mask_l5) << 5)) & wbb;
	// 	tmp = (((fbb & mask_l3) << 3) | ((fbb & mask_l5) << 5)) & bbb;
	// 	movers |= (tmp << 4) & wbb;
	// 	if (wkbb) {
	// 		let tmp = (fbb >> 4) & bbb;
	// 		if (tmp) movers |= (((tmp & mask_r3) >> 3) | ((tmp & mask_r5) >> 5)) & wkbb;
	// 		tmp = (((fbb & mask_r3) >> 3) | ((fbb & mask_r5) >> 5)) & bbb;
	// 		movers |= (tmp >> 4) & wkbb;
	// 	}
	// 	return movers;
	// });

	// let white_movable = $derived(white_jumpers | white_movers);

	let black_movers = $derived.by(() => {
		if (turn) return 0;

		const bkbb = bbb & kbb;
		let movers = 0;
		let tmp = (fbb << 4) & wbb;
		if (tmp) movers |= (((tmp & mask_l3) << 3) | ((tmp & mask_l5) << 5)) & bbb;
		tmp = (((fbb & mask_l3) << 3) | ((fbb & mask_l5) << 5)) & wbb;
		movers |= (tmp << 4) & bbb;
		if (bkbb) {
			let tmp = (fbb >> 4) & wbb;
			if (tmp) movers |= (((tmp & mask_r3) >> 3) | ((tmp & mask_r5) >> 5)) & bkbb;
			tmp = (((fbb & mask_r3) >> 3) | ((fbb & mask_r5) >> 5)) & wbb;
			movers |= (tmp >> 4) & bkbb;
		}
		if (movers != 0) return movers;

		movers = (fbb << 4) & bbb;
		movers |= ((fbb & mask_l3) << 3) & bbb;
		movers |= ((fbb & mask_l5) << 5) & bbb;
		if (bkbb) {
			movers |= (fbb >> 4) & bkbb;
			movers |= ((fbb & mask_r3) >> 3) & bkbb;
			movers |= ((fbb & mask_r5) >> 5) & bkbb;
		}
		return movers;
	});

	// let black_jumpers = $derived.by(() => {
	// 	if (turn) return 0;

	// 	const bkbb = bbb & kbb;
	// 	let movers = 0;
	// 	let tmp = (fbb << 4) & wbb;
	// 	if (tmp) movers |= (((tmp & mask_l3) << 3) | ((tmp & mask_l5) << 5)) & bbb;
	// 	tmp = (((fbb & mask_l3) << 3) | ((fbb & mask_l5) << 5)) & wbb;
	// 	movers |= (tmp << 4) & bbb;
	// 	if (bkbb) {
	// 		let tmp = (fbb >> 4) & wbb;
	// 		if (tmp) movers |= (((tmp & mask_r3) >> 3) | ((tmp & mask_r5) >> 5)) & bkbb;
	// 		tmp = (((fbb & mask_r3) >> 3) | ((fbb & mask_r5) >> 5)) & wbb;
	// 		movers |= (tmp >> 4) & bkbb;
	// 	}
	// 	return movers;
	// });

	// let black_movable = $derived(black_jumpers | black_movers);

	let all_bb = $derived(bbb | wbb);
	let whitePossibleMoves = $state(0);
	let blackPossibleMoves = $state(0);
	let selected = 0;

	function isOccupied(bb: number, row: number, col: number): boolean {
		const mask = 1 << (4 * row + col);
		return (mask & bb) != 0;
	}

	function showMovesWhite(row: number, col: number) {
		selected = 1 << (4 * row + col);
		whitePossibleMoves = (selected >> 4) & fbb;
		if (selected & mask_r3) whitePossibleMoves |= (selected >> 3) & fbb;
		if (selected & mask_r5) whitePossibleMoves |= (selected >> 5) & fbb;
		if ((selected & kbb) == 0) return;
		whitePossibleMoves |= (selected << 4) & fbb;
		if (selected & mask_l3) whitePossibleMoves |= (selected << 3) & fbb;
		if (selected & mask_l5) whitePossibleMoves |= (selected << 5) & fbb;
	}

	function showMovesBlack(row: number, col: number) {
		selected = 1 << (4 * row + col);
		blackPossibleMoves = (selected << 4) & fbb;
		if (selected & mask_l3) blackPossibleMoves |= (selected << 3) & fbb;
		if (selected & mask_l5) blackPossibleMoves |= (selected << 5) & fbb;
		if ((selected & kbb) == 0) return;
		blackPossibleMoves |= (selected >> 4) & fbb;
		if (selected & mask_r3) blackPossibleMoves |= (selected >> 3) & fbb;
		if (selected & mask_r5) blackPossibleMoves |= (selected >> 5) & fbb;
	}

	function move(bb: number, row: number, col: number): number {
		bb ^= selected;
		const mask = 1 << (4 * row + col);
		bb |= mask;
		if ((selected & kbb) != 0) {
			kbb ^= selected;
			kbb |= mask;
		}
		whitePossibleMoves = 0;
		blackPossibleMoves = 0;
		selected ^= 0;
		selected = 0;
		turn = !turn;
		return bb;
	}
</script>

<div class="bg-amber-900 p-4 w-fit">
	{#key all_bb}
		<div class="flex flex-col-reverse">
			{#each { length: 8 } as _, i}
				<div class="flex flex-row">
					{#each { length: 4 } as _, j}
						{@const isEmpty = !isOccupied(all_bb, i, j)}
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
									} else if (isPossibleWhite) {
										wbb = move(wbb, i, j);
									} else whitePossibleMoves = 0;
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
								</div>
							</button>
							<button
								class="bg-amber-100 w-10 h-10"
								class:order-1={i % 2 == 1}
								aria-label="fake-cell"
								onclick={() => {
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
