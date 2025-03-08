<script lang="ts">
	import decodeQR from '@paulmillr/qr/decode.js';
	import { Input } from '$lib/components/ui/input/index.js';
	import { Label } from '$lib/components/ui/label/index.js';
	import * as Card from '$lib/components/ui/card/index.js';
	import { svgToPng } from '@paulmillr/qr/dom.js';

	let files: FileList | undefined = $state();
	let decoded = $state('');
	let imageSrc = $state('');
	async function getPng(file: File) {
		return await svgToPng(await file.text(), 512, 512);
	}

	$effect(() => {
		if (files && files.length > 0) {
			const file = files[0];
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
	});
</script>

<Card.Root>
	<Card.Header>
		<Card.Title
			>{#if decoded}
				Decoded
			{:else}
				Decode
			{/if}</Card.Title
		>
		<Card.Description
			><span class="scroll-m-20 text-4xl font-extrabold tracking-tight lg:text-5xl text-foreground"
				>{decoded}</span
			></Card.Description
		>
	</Card.Header>
	<Card.Content class="flex w-full justify-center">
		<div class="grid w-full h-fit items-center gap-1.5">
			<Label for="picture">QR Code</Label>
			<Input
				class={`${files ? null : 'bg-primary'}`}
				bind:files
				id="picture"
				type="file"
				accept="image/*"
			/>
		</div>
	</Card.Content>
	<Card.Footer class="flex w-full justify-center">
		{#if imageSrc}
			<img src={imageSrc} alt="QR Code" />
		{/if}
	</Card.Footer>
</Card.Root>

<style>
</style>
