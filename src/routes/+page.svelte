<script>
	import Button from '$lib/components/ui/button/button.svelte';
	import Text from '$lib/components/ui/text/text.svelte';
</script>

<div class="flex justify-center pt-8 absolute w-full">
	<Text as="h3">Some game implemented with bitboard.</Text>
</div>
<div class="flex flex-col h-screen items-center justify-center gap-4">
	<Button href="/connect4">Connect 4</Button>
	<Button href="/tris">Tris</Button>
	<Button href="/checkers">Checkers</Button>
</div>
