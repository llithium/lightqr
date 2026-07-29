<script lang="ts">
	import decodeQR from 'qr/decode.js';
	import { Label } from '$lib/components/ui/label/index.js';
	import { Button } from '$lib/components/ui/button/index.js';
	import { svgToPng } from 'qr/dom.js';
	import ImageIcon from 'lucide-svelte/icons/image';
	import Check from 'lucide-svelte/icons/check';
	import Copy from 'lucide-svelte/icons/copy';
	import ExternalLink from 'lucide-svelte/icons/external-link';
	import X from 'lucide-svelte/icons/x';

	let decoded = $state('');
	let error = $state('');
	let imageSrc = $state('');
	let fileName = $state('');
	let dragOver = $state(false);
	let copied = $state(false);
	let fileInput = $state<HTMLInputElement>();
	let copyTimer: ReturnType<typeof setTimeout> | undefined;

	// Most QR payloads are links, so offer to open the decoded value when it is one.
	let link = $derived.by(() => {
		try {
			const url = new URL(decoded);
			return url.protocol === 'http:' || url.protocol === 'https:' ? url.href : null;
		} catch {
			return null;
		}
	});

	async function getPng(file: File) {
		return await svgToPng(await file.text(), 512, 512);
	}

	function reset() {
		decoded = '';
		error = '';
		imageSrc = '';
		fileName = '';
		copied = false;
		// Clearing the input lets the same file be picked twice in a row.
		if (fileInput) fileInput.value = '';
	}

	// Guards against an earlier, slower image landing after a newer one.
	let loadId = 0;

	// Single pipeline used by file-picker, drag-drop, and clipboard paste.
	function handleFile(file: File | undefined | null) {
		if (!file) return;

		reset();
		const id = ++loadId;

		if (!file.type.startsWith('image/')) {
			error = 'That file is not an image.';
			return;
		}
		fileName = file.name || 'Pasted image.png';

		const isSvg = file.type === 'image/svg+xml';
		const reader = new FileReader();

		reader.onerror = () => {
			if (id === loadId) error = 'Could not read that file.';
		};

		reader.onload = async (event) => {
			const img = new Image();

			img.onerror = () => {
				if (id === loadId) error = 'Could not load that image.';
			};

			img.onload = () => {
				if (id !== loadId) return;

				const canvas = document.createElement('canvas');
				canvas.width = img.width;
				canvas.height = img.height;

				const ctx = canvas.getContext('2d');
				if (!ctx) {
					error = 'Could not read the image data.';
					return;
				}
				ctx.drawImage(img, 0, 0);

				const imageData = ctx.getImageData(0, 0, img.width, img.height);

				// Rasterised SVGs come back anti-aliased; snap to pure black/white so the
				// decoder sees crisp modules.
				if (isSvg) {
					for (let i = 0; i < imageData.data.length; i += 4) {
						imageData.data[i] = imageData.data[i] < 128 ? 0 : 255;
						imageData.data[i + 1] = imageData.data[i + 1] < 128 ? 0 : 255;
						imageData.data[i + 2] = imageData.data[i + 2] < 128 ? 0 : 255;
					}
				}

				try {
					decoded = decodeQR({
						height: imageData.height,
						width: imageData.width,
						data: imageData.data
					});
				} catch {
					error = "We couldn't find a QR code in that image.";
				}
			};

			try {
				img.src = isSvg ? await getPng(file) : (event.target!.result as string);
			} catch {
				if (id === loadId) error = 'Could not load that image.';
				return;
			}
			if (id === loadId) imageSrc = img.src;
		};

		reader.readAsDataURL(file);
	}

	function onInputChange(event: Event) {
		const target = event.currentTarget as HTMLInputElement;
		handleFile(target.files?.[0]);
	}

	function onDrop(event: DragEvent) {
		event.preventDefault();
		dragOver = false;
		handleFile(event.dataTransfer?.files?.[0]);
	}

	// Paste a QR image straight from the clipboard (Cmd/Ctrl-V).
	function onPaste(event: ClipboardEvent) {
		const items = event.clipboardData?.items ?? [];
		for (const item of items) {
			if (item.type.startsWith('image/')) {
				handleFile(item.getAsFile());
				event.preventDefault();
				break;
			}
		}
	}

	async function copyDecoded() {
		try {
			await navigator.clipboard.writeText(decoded);
			copied = true;
			clearTimeout(copyTimer);
			copyTimer = setTimeout(() => (copied = false), 2000);
		} catch {
			error = 'Could not copy to the clipboard.';
		}
	}
</script>

<svelte:window onpaste={onPaste} />

<div class="grid items-center gap-8 md:grid-cols-2 md:gap-10">
	<!-- min-w-0: grid items default to min-width:auto, which lets a long decoded
	     string widen the column past the viewport instead of wrapping. -->
	<div class="min-w-0">
		<div aria-live="polite">
			{#if decoded}
				<div
					class="mb-4 inline-flex items-center gap-1.5 rounded-full bg-primary/15 px-3 py-1.5 text-xs font-extrabold text-primary"
				>
					<Check class="size-3.5" /> Decoded
				</div>
			{/if}
			<h2
				class="break-words text-4xl font-extrabold tracking-tight lg:text-5xl {decoded
					? 'text-foreground'
					: 'text-muted-foreground'}"
			>
				{decoded || 'Scan a QR code'}
			</h2>
			{#if error}
				<p class="mt-3 text-sm font-semibold text-destructive">{error}</p>
			{/if}
		</div>

		{#if decoded}
			<div class="mt-4 flex flex-wrap items-center gap-2">
				<Button variant="outline" size="sm" onclick={copyDecoded}>
					{#if copied}
						<Check class="mr-1.5 size-3.5" /> Copied
					{:else}
						<Copy class="mr-1.5 size-3.5" /> Copy
					{/if}
				</Button>
				{#if link}
					<Button variant="outline" size="sm" href={link} target="_blank" rel="noopener noreferrer">
						<ExternalLink class="mr-1.5 size-3.5" /> Open link
					</Button>
				{/if}
				<Button variant="ghost" size="sm" onclick={reset}>
					<X class="mr-1.5 size-3.5" /> Clear
				</Button>
			</div>
		{/if}

		<Label for="picture" class="mb-2 mt-7 block">Add a QR image</Label>
		<button
			type="button"
			onclick={() => fileInput?.click()}
			ondragover={(event) => {
				event.preventDefault();
				dragOver = true;
			}}
			ondragleave={() => (dragOver = false)}
			ondrop={onDrop}
			class={`flex w-full items-center gap-3 rounded-2xl border-2 border-dashed p-4 text-left transition-colors ${
				dragOver
					? 'border-primary bg-primary/10'
					: 'border-primary/40 bg-muted hover:border-primary hover:bg-primary/5'
			}`}
		>
			<span
				class="flex size-10 shrink-0 items-center justify-center rounded-xl bg-primary/15 text-primary"
			>
				<ImageIcon class="size-5" />
			</span>
			<span class="min-w-0 flex-1">
				<span class="block truncate font-semibold">{fileName || 'Choose a QR image'}</span>
				<span class="block text-sm text-muted-foreground">Click, drop, or paste an image</span>
			</span>
			<kbd
				class="shrink-0 rounded-md border border-b-2 bg-background px-2 py-1 text-xs font-bold text-muted-foreground"
				>⌘V</kbd
			>
		</button>
		<input
			bind:this={fileInput}
			onchange={onInputChange}
			id="picture"
			type="file"
			accept="image/*"
			class="hidden"
		/>
	</div>

	<div class="flex justify-center">
		{#if imageSrc}
			<div class="rounded-2xl bg-white p-4 shadow-lg shadow-black/5">
				<img src={imageSrc} alt="Scanned QR Code" class="max-h-64 w-auto rounded-lg" />
			</div>
		{:else}
			<div
				class="flex aspect-square w-full max-w-xs items-center justify-center rounded-2xl border-2 border-dashed border-border bg-muted/50 text-center text-sm text-muted-foreground"
			>
				Your scanned image<br />will appear here
			</div>
		{/if}
	</div>
</div>
