<script lang="ts">
	import type { Snippet } from 'svelte';
	import '../app.css';
	let { children }: { children: Snippet } = $props();
</script>

<svelte:head>
	<title>LightQR</title>
	<meta name="description" content="QR Code generator" />
</svelte:head>
<div>
	{@render children()}
</div>

<style>
</style>
