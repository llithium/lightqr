<script lang="ts">
	import decodeQR from 'qr/decode.js';
	import { Label } from '$lib/components/ui/label/index.js';
	import * as Card from '$lib/components/ui/card/index.js';
	import { svgToPng } from 'qr/dom.js';
	import ImageIcon from 'lucide-svelte/icons/image';
	import Check from 'lucide-svelte/icons/check';

	let decoded = $state('');
	let imageSrc = $state('');
	let fileName = $state('');
	let dragOver = $state(false);
	let fileInput: HTMLInputElement;

	async function getPng(file: File) {
		return await svgToPng(await file.text(), 512, 512);
	}

	// Single pipeline used by file-picker, drag-drop, and clipboard paste.
	function handleFile(file: File | undefined | null) {
		if (!file || !file.type.startsWith('image/')) return;
		fileName = file.name || 'Pasted image.png';
		const reader = new FileReader();

		reader.onload = async (event) => {
			const img = new Image();
			if (file.type == 'image/svg+xml') {
				img.src = await getPng(file);
			} else {
				img.src = event.target!.result as string;
			}
			imageSrc = img.src;

			img.onload = () => {
				const canvas = document.createElement('canvas');
				canvas.width = img.width;
				canvas.height = img.height;

				const ctx = canvas.getContext('2d');
				ctx!.drawImage(img, 0, 0);

				const imageData = ctx!.getImageData(0, 0, img.width, img.height);

				if (file.type == 'image/svg+xml') {
					for (let i = 0; i < imageData.data.length; i += 4) {
						const r = imageData.data[i];
						const g = imageData.data[i + 1];
						const b = imageData.data[i + 2];

						imageData.data[i] = r < 128 ? 0 : 255;
						imageData.data[i + 1] = g < 128 ? 0 : 255;
						imageData.data[i + 2] = b < 128 ? 0 : 255;
					}
				}

				try {
					decoded = decodeQR({
						height: imageData.height,
						width: imageData.width,
						data: imageData.data
					});
				} catch (error) {
					decoded = 'Decoding failed';
				}
			};
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
</script>

<svelte:window onpaste={onPaste} />

<Card.Root>
	<Card.Content class="grid items-center gap-8 p-6 md:grid-cols-2 md:gap-10 md:p-8">
		<div>
			{#if decoded}
				<div
					class="mb-4 inline-flex items-center gap-1.5 rounded-full bg-primary/15 px-3 py-1.5 text-xs font-extrabold text-primary"
				>
					<Check class="size-3.5" /> Decoded
				</div>
			{/if}
			<h2
				class="mb-7 break-words text-4xl font-extrabold tracking-tight lg:text-5xl {decoded
					? 'text-foreground'
					: 'text-muted-foreground'}"
			>
				{decoded || 'Scan a QR code'}
			</h2>

			<Label for="picture" class="mb-2 block">Add a QR image</Label>
			<button
				type="button"
				onclick={() => fileInput.click()}
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
	</Card.Content>
</Card.Root>

<style>
</style>
