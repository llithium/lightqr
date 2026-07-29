<script lang="ts">
	import encodeQR, { type ErrorCorrection } from 'qr';
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

	const MIN_SIZE = 25;
	const MAX_SIZE = 1000;
	const SIZE_STEP = 5;
	const DEFAULT_SIZE = 300;

	const EC_OPTIONS = [
		{ value: 'low', label: 'Low', percent: '7%' },
		{ value: 'medium', label: 'Medium', percent: '15%' },
		{ value: 'quartile', label: 'Quartile', percent: '25%' },
		{ value: 'high', label: 'High', percent: '30%' }
	] as const satisfies readonly { value: ErrorCorrection; label: string; percent: string }[];

	const TYPE_LABELS = { png: 'PNG', svg: 'SVG' } as const;
	type FileType = keyof typeof TYPE_LABELS;

	// The URL is user-editable, so every param is validated before it reaches state.
	function initialType(): FileType {
		const raw = page.url.searchParams.get('type');
		return raw === 'svg' || raw === 'png' ? raw : 'png';
	}

	function initialEcc(): ErrorCorrection {
		const raw = page.url.searchParams.get('ec');
		return EC_OPTIONS.some((option) => option.value === raw) ? (raw as ErrorCorrection) : 'medium';
	}

	function initialSize(): number {
		const raw = Number(page.url.searchParams.get('size'));
		if (!Number.isFinite(raw) || raw <= 0) return DEFAULT_SIZE;
		const snapped = Math.round(raw / SIZE_STEP) * SIZE_STEP;
		return Math.min(MAX_SIZE, Math.max(MIN_SIZE, snapped));
	}

	let text = $state('');
	let type = $state<FileType>(initialType());
	let ecc = $state<ErrorCorrection>(initialEcc());
	// `sliderValue` tracks the thumb for the live label; `renderSize` lags behind it so a
	// drag across the range doesn't re-encode a 1000px PNG on every tick.
	let sliderValue = $state(initialSize());
	let renderSize = $state(initialSize());

	let qrCode = $derived(encodeQR(text || 'LightQR', 'svg', { ecc, border: 1, scale: 1 }));
	let eccOption = $derived(EC_OPTIONS.find((option) => option.value === ecc)!);

	let svgUrl = $state<string | null>(null);
	let pngData = $state<string | null>(null);
	let downloadUrl = $derived(type === 'svg' ? svgUrl : pngData);

	// One object URL per QR revision, revoked as soon as it is superseded.
	$effect(() => {
		if (type !== 'svg') {
			svgUrl = null;
			return;
		}
		const url = URL.createObjectURL(new Blob([qrCode], { type: 'image/svg+xml' }));
		svgUrl = url;
		return () => URL.revokeObjectURL(url);
	});

	// Rasterising is async, so a slow large render must not overwrite a newer small one.
	let renderId = 0;
	$effect(() => {
		if (type !== 'png') {
			pngData = null;
			return;
		}
		const id = ++renderId;
		const svg = qrCode;
		const size = renderSize;
		svgToPng(svg, size, size)
			.then((data) => {
				if (id === renderId) pngData = data;
			})
			.catch(() => {
				if (id === renderId) pngData = null;
			});
	});

	function setParam(key: string, value: string) {
		const newURL = new URL(page.url);
		newURL.searchParams.set(key, value);
		goto(newURL, { replaceState: true, keepFocus: true, noScroll: true });
	}

	const commitSize = debounce((value: number) => {
		renderSize = value;
		setParam('size', value.toString());
	}, 300);
</script>

<div class="grid gap-8 md:grid-cols-2 md:items-center">
	<div class="space-y-5">
		<div class="space-y-2">
			<Label for="text">Link or text to encode</Label>
			<Input
				class="h-10"
				bind:value={text}
				type="text"
				id="text"
				maxlength={1000}
				placeholder="Text or URL"
			/>
		</div>

		<div class="grid grid-cols-2 gap-4">
			<div class="space-y-2">
				<Label for="type">Format</Label>
				<Select.Root
					onValueChange={(value) => setParam('type', value)}
					bind:value={type}
					name="type"
					type="single"
				>
					<Select.Trigger class="h-10 w-full">{TYPE_LABELS[type]}</Select.Trigger>
					<Select.Content>
						<Select.Item value="png">PNG</Select.Item>
						<Select.Item value="svg">SVG</Select.Item>
					</Select.Content>
				</Select.Root>
			</div>
			<div class="space-y-2">
				<Label for="errorCorrection">Error correction</Label>
				<Select.Root
					onValueChange={(value) => setParam('ec', value)}
					bind:value={ecc}
					name="errorCorrection"
					type="single"
				>
					<Select.Trigger class="h-10 w-full">
						{eccOption.label} ({eccOption.percent})
					</Select.Trigger>
					<Select.Content>
						{#each EC_OPTIONS as option (option.value)}
							<Select.Item value={option.value}>{option.label} ({option.percent})</Select.Item>
						{/each}
					</Select.Content>
				</Select.Root>
			</div>
		</div>

		{#if type === 'png'}
			<div class="space-y-2">
				<div class="flex items-center justify-between">
					<Label for="size">Size</Label>
					<span
						class="rounded-full bg-primary/15 px-2.5 py-0.5 text-xs font-bold tabular-nums text-primary"
						>{sliderValue} px</span
					>
				</div>
				<Slider
					onValueChange={commitSize}
					type="single"
					bind:value={sliderValue}
					min={MIN_SIZE}
					max={MAX_SIZE}
					step={SIZE_STEP}
					class="w-full"
				/>
			</div>
		{/if}
	</div>

	<div class="flex flex-col items-center gap-5">
		<h2 class="text-sm font-bold uppercase tracking-wide text-muted-foreground">Your QR code</h2>

		<div
			class="rounded-2xl bg-white p-4 shadow-lg shadow-black/5 transition-transform duration-200 hover:-translate-y-1"
		>
			<!-- Scales with viewport height so tall screens get a bigger preview
			     instead of dead space, without overflowing short ones. -->
			<div class="flex size-[clamp(11rem,32vh,17rem)] items-center justify-center">
				{#if type === 'svg'}
					<div class="h-full w-full *:h-full *:w-full">
						<!-- Safe: encodeQR emits its own <svg> of rects; `text` is encoded into the
						     module matrix, never interpolated into the markup. -->
						<!-- eslint-disable-next-line svelte/no-at-html-tags -->
						{@html qrCode}
					</div>
				{:else}
					<img
						src={pngData}
						alt={text ? `QR code for ${text}` : 'QR code'}
						class="h-full w-full object-contain [image-rendering:pixelated]"
					/>
				{/if}
			</div>
		</div>

		<div class="flex flex-wrap justify-center gap-2">
			<span
				class="rounded-full border border-border bg-muted px-3 py-1 text-xs font-bold text-muted-foreground"
				>{TYPE_LABELS[type]}</span
			>
			{#if type === 'png'}
				<span
					class="rounded-full border border-border bg-muted px-3 py-1 text-xs font-bold text-muted-foreground"
					>{sliderValue}×{sliderValue}</span
				>
			{/if}
			<span
				class="rounded-full border border-border bg-muted px-3 py-1 text-xs font-bold text-muted-foreground"
				>EC {eccOption.percent}</span
			>
		</div>

		<Button
			download={`qr-code.${type}`}
			href={downloadUrl}
			aria-disabled={!downloadUrl}
			class="w-full max-w-xs"
		>
			<Download class="mr-2 size-4" />
			Download {TYPE_LABELS[type]}
		</Button>
	</div>
</div>
