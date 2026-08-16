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

	const TYPE_LABELS = {
		png: 'PNG',
		jpg: 'JPEG',
		webp: 'WebP',
		svg: 'SVG'
	} as const;
	type FileType = keyof typeof TYPE_LABELS;
	type RasterType = Exclude<FileType, 'svg'>;

	const RASTER_MIME = {
		png: 'image/png',
		jpg: 'image/jpeg',
		webp: 'image/webp'
	} as const satisfies Record<RasterType, string>;

	function isRasterType(type: FileType): type is RasterType {
		return type !== 'svg';
	}

	// The URL is user-editable, so every param is validated before it reaches state.
	function initialType(): FileType {
		const raw = page.url.searchParams.get('type');
		return raw === 'svg' || raw === 'png' || raw === 'jpg' || raw === 'webp' ? raw : 'png';
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

	function svgToRaster(
		svg: string,
		width: number,
		height: number,
		type: RasterType
	): Promise<string> {
		if (type === 'png') return svgToPng(svg, width, height);

		return new Promise((resolve, reject) => {
			const url = URL.createObjectURL(new Blob([svg], { type: 'image/svg+xml' }));
			const image = new Image();
			const cleanup = () => URL.revokeObjectURL(url);

			image.onload = () => {
				try {
					const canvas = document.createElement('canvas');
					canvas.width = width;
					canvas.height = height;

					const context = canvas.getContext('2d');
					if (!context) throw new Error('Unable to create canvas context');

					// JPEG has no alpha channel, and a white background is appropriate for QR codes.
					context.fillStyle = '#ffffff';
					context.fillRect(0, 0, width, height);
					context.imageSmoothingEnabled = false;
					context.drawImage(image, 0, 0, width, height);

					const mime = RASTER_MIME[type];
					const dataUrl = canvas.toDataURL(mime, 1);
					if (!dataUrl.startsWith(`data:${mime}`)) {
						throw new Error(`${TYPE_LABELS[type]} export is not supported by this browser`);
					}

					resolve(dataUrl);
				} catch (error) {
					reject(error);
				} finally {
					cleanup();
				}
			};

			image.onerror = () => {
				cleanup();
				reject(new Error('Unable to render QR code'));
			};
			image.src = url;
		});
	}

	function dataUrlSize(dataUrl: string): number {
		const comma = dataUrl.indexOf(',');
		if (comma === -1) return 0;

		const base64 = dataUrl.slice(comma + 1);
		const padding = base64.endsWith('==') ? 2 : base64.endsWith('=') ? 1 : 0;
		return Math.floor((base64.length * 3) / 4) - padding;
	}

	function formatBytes(bytes: number): string {
		if (bytes < 1024) return `${bytes} B`;
		if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
		return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
	}

	let text = $state('');
	let type = $state<FileType>(initialType());
	let ecc = $state<ErrorCorrection>(initialEcc());
	// `sliderValue` tracks the thumb for the live label; `renderSize` lags behind it so a
	// drag across the range doesn't re-encode a 1000px raster image on every tick.
	let sliderValue = $state(initialSize());
	let renderSize = $state(initialSize());

	// Nothing is encoded until there is real content, so the preview never offers a
	// download for a QR the user didn't ask for.
	let hasContent = $derived(text.trim().length > 0);
	let qrCode = $derived(hasContent ? encodeQR(text, 'svg', { ecc, border: 1, scale: 1 }) : null);
	let eccOption = $derived(EC_OPTIONS.find((option) => option.value === ecc)!);

	let svgUrl = $state<string | null>(null);
	let rasterData = $state<string | null>(null);
	let rasterSource = $state<string | null>(null);
	let rasterRendering = $state(false);
	let downloadUrl = $derived(
		type === 'svg' ? svgUrl : rasterRendering ? null : rasterData
	);
	let fileSize = $derived(
		!qrCode
			? null
			: type === 'svg'
				? new Blob([qrCode], { type: 'image/svg+xml' }).size
				: rasterData && !rasterRendering
					? dataUrlSize(rasterData)
					: null
	);

	// One object URL per QR revision, revoked as soon as it is superseded.
	$effect(() => {
		if (!qrCode || type !== 'svg') {
			svgUrl = null;
			return;
		}
		const url = URL.createObjectURL(new Blob([qrCode], { type: 'image/svg+xml' }));
		svgUrl = url;
		return () => URL.revokeObjectURL(url);
	});

	// Rasterising is async, so a slow large render must not overwrite a newer one.
	// Keep the previous image painted during size-only renders to avoid a preview flash.
	let renderId = 0;
	$effect(() => {
		if (!qrCode || !isRasterType(type)) {
			++renderId;
			rasterData = null;
			rasterSource = null;
			rasterRendering = false;
			return;
		}

		const id = ++renderId;
		const svg = qrCode;
		const size = renderSize;
		const outputType = type;

		// A size change doesn't alter the QR modules, so the previous raster is a valid
		// visual placeholder until the new dimensions finish encoding. Content/ECC changes
		// clear it so we never display a QR for stale data.
		if (rasterSource !== svg) rasterData = null;
		rasterRendering = true;

		svgToRaster(svg, size, size, outputType)
			.then((data) => {
				if (id !== renderId) return;
				rasterData = data;
				rasterSource = svg;
				rasterRendering = false;
			})
			.catch(() => {
				if (id !== renderId) return;
				rasterData = null;
				rasterSource = null;
				rasterRendering = false;
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

<div class="grid gap-8 md:grid-cols-2 md:items-start">
	<div class="space-y-5">
		<div class="space-y-2">
			<!-- block: an inline <label> sits inside the wrapper's line box and drops a
			     few px, which knocks this column out of line with the heading opposite. -->
			<Label for="text" class="block">Content to encode</Label>
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
				<Label for="type" class="block">Format</Label>
				<Select.Root
					onValueChange={(value) => setParam('type', value)}
					bind:value={type}
					name="type"
					type="single"
				>
					<Select.Trigger class="h-10 w-full">{TYPE_LABELS[type]}</Select.Trigger>
					<Select.Content>
						<Select.Item value="png">PNG</Select.Item>
						<Select.Item value="jpg">JPEG</Select.Item>
						<Select.Item value="webp">WebP</Select.Item>
						<Select.Item value="svg">SVG</Select.Item>
					</Select.Content>
				</Select.Root>
			</div>
			<div class="space-y-2">
				<Label for="errorCorrection" class="block">Error correction</Label>
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

		{#if type !== 'svg'}
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
		<!-- leading-none matches the Label opposite it, so both columns start flush. -->
		<h2 class="text-sm font-bold uppercase leading-none tracking-wide text-muted-foreground">
			Your QR code
		</h2>

		{#if qrCode}
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
							src={rasterData ?? undefined}
							alt={`QR code for ${text}`}
							class="h-full w-full object-contain [image-rendering:pixelated]"
						/>
					{/if}
				</div>
			</div>
		{:else}
			<!-- Same footprint as the rendered preview, so the column doesn't jump
			     the moment the first character is typed. -->
			<div
				class="flex size-[calc(clamp(11rem,32vh,17rem)+2rem)] items-center justify-center rounded-2xl border-2 border-dashed border-border bg-muted/50 p-4 text-center text-sm text-muted-foreground"
			>
				Your QR code<br />will appear here
			</div>
		{/if}

		<div class="flex flex-wrap justify-center gap-2">
			<span
				class="rounded-full border border-border bg-muted px-3 py-1 text-xs font-bold text-muted-foreground"
				>{TYPE_LABELS[type]}</span
			>
			{#if type !== 'svg'}
				<span
					class="rounded-full border border-border bg-muted px-3 py-1 text-xs font-bold text-muted-foreground"
					>{sliderValue}×{sliderValue}</span
				>
			{/if}
			{#if fileSize !== null}
				<span
					class="rounded-full border border-border bg-muted px-3 py-1 text-xs font-bold tabular-nums text-muted-foreground"
					>{formatBytes(fileSize)}</span
				>
			{:else if qrCode && type !== 'svg'}
				<span
					class="rounded-full border border-border bg-muted px-3 py-1 text-xs font-bold text-muted-foreground"
					>Calculating…</span
				>
			{/if}
			<span
				class="rounded-full border border-border bg-muted px-3 py-1 text-xs font-bold text-muted-foreground"
				>EC {eccOption.percent}</span
			>
		</div>

		<!-- With no href the Button renders a real <button>, so `disabled` actually
		     blocks activation rather than just dimming a live link. -->
		<Button
			download={downloadUrl ? `qr-code.${type}` : undefined}
			href={downloadUrl ?? undefined}
			disabled={!downloadUrl}
			class="w-full max-w-xs"
		>
			<Download class="mr-2 size-4" />
			Download {TYPE_LABELS[type]}
		</Button>
	</div>
</div>