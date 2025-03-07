<script lang="ts">
	import encodeQR, { type ErrorCorrection } from '@paulmillr/qr';
	import * as Card from '$lib/components/ui/card/index.js';
	import { Input } from '$lib/components/ui/input/index.js';
	import { Label } from '$lib/components/ui/label/index.js';
	import { Button } from '$lib/components/ui/button/index.js';
	import Download from 'lucide-svelte/icons/download';
	import * as Select from '$lib/components/ui/select/index.js';
	import Slider from './ui/slider/slider.svelte';
	import { page } from '$app/state';
	import { goto } from '$app/navigation';
	import { svgToPng } from '@paulmillr/qr/dom.js';
	import debounce from 'debounce';

	let text = $state('');
	let ecc: ErrorCorrection = $state(
		(page.url.searchParams.get('ec') as ErrorCorrection) || 'medium'
	);
	let type = $state(page.url.searchParams.get('type') || 'png');
	let sliderValue = $state(300);
	let qrCode = $derived(encodeQR(text, 'svg', { ecc: ecc, border: 1, scale: 1 }));

	let pngData = $state<string | null>(null);
	let blob = $derived(
		type === 'svg' ? URL.createObjectURL(new Blob([qrCode], { type: 'image/svg+xml' })) : null
	);

	async function updatePngData() {
		if (type === 'png') {
			pngData = await svgToPng(qrCode, sliderValue, sliderValue);
		} else {
			pngData = null;
		}
	}

	$effect(() => {
		updatePngData();
	});

	function onTypeChange() {
		const newURL = new URL(page.url);
		newURL.searchParams.set('type', type);
		goto(newURL, { replaceState: true });
	}
	function onEcChange() {
		const newURL = new URL(page.url);
		newURL.searchParams.set('ec', ecc);
		goto(newURL, { replaceState: true });
	}
	function onSizeChange() {
		const newURL = new URL(page.url);
		newURL.searchParams.set('size', sliderValue.toString());
		goto(newURL, { replaceState: true });
	}
	const typeLabels = new Map([
		['svg', 'SVG'],
		['png', 'PNG']
	]);
	const ecLabel = new Map([
		['low', 'Low (7%)'],
		['medium', 'Medium (15%)'],
		['quartile', 'Quartile (25%)'],
		['high', 'High (30%)']
	]);
</script>

<div class="flex w-full gap-4">
	<Card.Root class="w-1/2">
		<Card.Header>
			<Card.Title>Generator</Card.Title>
			<!-- <Card.Description>Card Description</Card.Description> -->
		</Card.Header>
		<Card.Content class="h-full flex flex-col gap-4">
			<div class="space-y-2">
				<Label for="text">Text or URL</Label>
				<Input class="h-10" autofocus bind:value={text} type="text" id="text" maxlength={1000} />
			</div>
			<div class="space-y-2">
				<Label for="type">Type</Label>
				<Select.Root onValueChange={onTypeChange} bind:value={type} name="type" type="single">
					<Select.Trigger class="w-full h-10">{typeLabels.get(type)}</Select.Trigger>
					<Select.Content>
						<Select.Item value="png">PNG</Select.Item>
						<Select.Item value="svg">SVG</Select.Item>
					</Select.Content>
				</Select.Root>
			</div>
			{#if type === 'png'}
				<div class="space-y-2">
					<Label for="size">Size: {sliderValue}px</Label>
					<Slider
						onValueChange={debounce(onSizeChange, 300)}
						type="single"
						bind:value={sliderValue}
						min={25}
						max={1000}
						step={5}
						class="w-full"
					/>
				</div>
			{/if}
			<div class="space-y-2">
				<Label for="errorCorrection">Error Correction</Label>
				<Select.Root
					onValueChange={onEcChange}
					bind:value={ecc}
					name="errorCorrection"
					type="single"
				>
					<Select.Trigger class="w-full h-10">{ecLabel.get(ecc)}</Select.Trigger>
					<Select.Content>
						<Select.Item value="low">Low (7%)</Select.Item>
						<Select.Item value="medium">Medium (15%)</Select.Item>
						<Select.Item value="quartile">Quartile (25%)</Select.Item>
						<Select.Item value="high">High (30%)</Select.Item>
					</Select.Content>
				</Select.Root>
			</div>
		</Card.Content>
		<!-- <Card.Footer>
			<p>Card Footer</p>
		</Card.Footer> -->
	</Card.Root>
	<Card.Root class="w-1/2 flex flex-col justify-between items-center">
		<Card.Header>
			<Card.Title>Preview</Card.Title>
			<!-- <Card.Description>Card Description</Card.Description> -->
		</Card.Header>
		<Card.Content class="w-full h-full flex justify-center items-center">
			{#if type === 'svg'}
				<div class="*:h-full *:w-full h-full w-full">
					{@html qrCode}
				</div>
			{:else if type === 'png'}
				<img src={pngData} alt={text} />
			{/if}
		</Card.Content>
		<Card.Footer class="w-full">
			<Button download="QR Code" href={blob || pngData} class="w-full">
				<Download class="mr-2 size-4" />
				Downlaod
			</Button>
		</Card.Footer>
	</Card.Root>
</div>

<style>
</style>
