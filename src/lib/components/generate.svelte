<script lang="ts">
	import encodeQR, { type ErrorCorrection } from '@paulmillr/qr';
	import * as Card from '$lib/components/ui/card/index.js';
	import { Input } from '$lib/components/ui/input/index.js';
	import { Label } from '$lib/components/ui/label/index.js';
	import { Button } from '$lib/components/ui/button/index.js';
	import Download from 'lucide-svelte/icons/download';
	import * as Select from '$lib/components/ui/select/index.js';
	import { capitalize } from '@/utils';
	import type { QRCodeType } from '@/types';

	let text = $state('');
	let ecc: ErrorCorrection = $state('medium');
	let type: QRCodeType = $state('svg');
	let scale = $derived((type as QRCodeType) === 'gif' ? 40 : 2);
	let qrCode = $derived(encodeQR(text, type, { ecc: ecc, border: 2, scale: scale }));
	let blob = $derived(
		(type as QRCodeType) === 'gif'
			? URL.createObjectURL(new Blob([qrCode], { type: 'image/gif' }))
			: null
	);
</script>

<div class="flex w-full gap-4">
	<Card.Root class="w-1/2">
		<Card.Header>
			<Card.Title>Generator</Card.Title>
			<!-- <Card.Description>Card Description</Card.Description> -->
		</Card.Header>
		<Card.Content class="h-full flex flex-col gap-4">
			<div>
				<Label for="text">Text or URL</Label>
				<Input bind:value={text} type="text" id="text" maxlength={1000} />
			</div>
			<div>
				<Label for="type">Type</Label>
				<Select.Root bind:value={type} name="type" type="single">
					<Select.Trigger class="w-full">{capitalize(type)}</Select.Trigger>
					<Select.Content>
						<Select.Item value="svg">SVG</Select.Item>
						<Select.Item value="gif">Gif</Select.Item>
					</Select.Content>
				</Select.Root>
			</div>
			<div>
				<Label for="errorCorrection">Error Correction</Label>
				<Select.Root bind:value={ecc} name="errorCorrection" type="single">
					<Select.Trigger class="w-full">{capitalize(ecc)}</Select.Trigger>
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
	<Card.Root class="w-1/2">
		<Card.Header>
			<Card.Title>Preview</Card.Title>
			<!-- <Card.Description>Card Description</Card.Description> -->
		</Card.Header>
		<Card.Content>
			{#if type === 'svg'}
				{@html qrCode}
			{:else if type === 'gif'}
				<img src={blob} alt={text} />
			{/if}
		</Card.Content>
		<Card.Footer>
			<Button variant="outline" class="w-full">
				<Download class="mr-2 size-4" />
				Downlaod
			</Button>
		</Card.Footer>
	</Card.Root>
</div>

<style>
</style>
