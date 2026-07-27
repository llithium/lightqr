<script lang="ts">
	import encodeQR, { type ErrorCorrection } from 'qr';
	import * as Card from '$lib/components/ui/card/index.js';
	import { Input } from '$lib/components/ui/input/index.js';
	import { Label } from '$lib/components/ui/label/index.js';
	import { Button } from '$lib/components/ui/button/index.js';
	import Download from 'lucide-svelte/icons/download';
	import * as Select from '$lib/components/ui/select/index.js';
	import Slider from './ui/slider/slider.svelte';
	import { page } from '$app/state';
	import { goto } from '$app/navigation';
	import { svgToPng } from 'qr/dom.js';
	import debounce from 'debounce';

	let text = $state('');
	let ecc: ErrorCorrection = $state(
		(page.url.searchParams.get('ec') as ErrorCorrection) || 'medium'
	);
	let type = $state(page.url.searchParams.get('type') || 'png');
	let sliderValue = $state(300);
	let qrCode = $derived(encodeQR(text || 'LightQR', 'svg', { ecc: ecc, border: 1, scale: 1 }));

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
	const ecNote = new Map([
		['low', '7%'],
		['medium', '15%'],
		['quartile', '25%'],
		['high', '30%']
	]);
</script>

<div class="flex flex-col md:flex-row w-full gap-4">
	<Card.Root class="md:w-1/2">
		<Card.Header>
			<Card.Title>Content</Card.Title>
		</Card.Header>
		<Card.Content class="h-full flex flex-col gap-4">
			<div class="space-y-2">
				<!-- <Label for="text">What should it link to?</Label> -->
				<Input class="h-10" autofocus bind:value={text} type="text" id="text" maxlength={1000} />
			</div>
			<div class="grid grid-cols-2 gap-4">
				<div class="space-y-2">
					<Label for="type">Format</Label>
					<Select.Root onValueChange={onTypeChange} bind:value={type} name="type" type="single">
						<Select.Trigger class="w-full h-10">{typeLabels.get(type)}</Select.Trigger>
						<Select.Content>
							<Select.Item value="png">PNG</Select.Item>
							<Select.Item value="svg">SVG</Select.Item>
						</Select.Content>
					</Select.Root>
				</div>
				<div class="space-y-2">
					<Label for="errorCorrection">Quality</Label>
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
			</div>
			{#if type === 'png'}
				<div class="space-y-2">
					<div class="flex items-center justify-between">
						<Label for="size">Size</Label>
						<span
							class="rounded-full bg-primary/15 px-2.5 py-0.5 text-xs font-bold text-primary tabular-nums"
							>{sliderValue} px</span
						>
					</div>
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
		</Card.Content>
	</Card.Root>
	<Card.Root
		class="md:w-1/2 flex flex-col justify-between items-center bg-gradient-to-b from-card to-muted/30"
	>
		<Card.Header>
			<Card.Title class="text-center">Your QR code</Card.Title>
		</Card.Header>
		<Card.Content class="w-full h-full flex flex-col justify-center items-center gap-5">
			<div
				class="rounded-2xl bg-white p-4 shadow-lg shadow-black/5 transition-transform duration-200 hover:-translate-y-1"
			>
				<div class="flex size-48 items-center justify-center">
					{#if type === 'svg'}
						<div class="*:h-full *:w-full h-full w-full">
							{@html qrCode}
						</div>
					{:else if type === 'png'}
						<img
							src={pngData}
							alt={text}
							class="h-full w-full object-contain [image-rendering:pixelated]"
						/>
					{/if}
				</div>
			</div>
			<div class="flex flex-wrap justify-center gap-2">
				<span
					class="rounded-full border border-border bg-muted px-3 py-1 text-xs font-bold text-muted-foreground"
					>{typeLabels.get(type)}</span
				>
				{#if type === 'png'}
					<span
						class="rounded-full border border-border bg-muted px-3 py-1 text-xs font-bold text-muted-foreground"
						>{sliderValue}×{sliderValue}</span
					>
				{/if}
				<span
					class="rounded-full border border-border bg-muted px-3 py-1 text-xs font-bold text-muted-foreground"
					>EC {ecNote.get(ecc)}</span
				>
			</div>
		</Card.Content>
		<Card.Footer class="w-full">
			<Button download="QR Code" href={blob || pngData} class="w-full">
				<Download class="mr-2 size-4" />
				Download {typeLabels.get(type)}
			</Button>
		</Card.Footer>
	</Card.Root>
</div>

<style>
</style>
