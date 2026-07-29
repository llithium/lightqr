<script lang="ts">
	import { goto } from '$app/navigation';
	import { page } from '$app/state';
	import * as Tabs from '$lib/components/ui/tabs/index.js';
	import Generate from '@/components/generate.svelte';
	import Scan from '@/components/scan.svelte';
	import Zap from 'lucide-svelte/icons/zap';
	let activeTab = $state(page.url.searchParams.get('tab') || 'generate');

	function onTabChange() {
		const newURL = new URL(page.url);
		newURL.searchParams.set('tab', activeTab);
		goto(newURL, { replaceState: true, keepFocus: true, noScroll: true });
	}
</script>

<main class="flex min-h-dvh w-full flex-col">
	<section class="mx-auto flex w-full max-w-5xl flex-1 flex-col px-4 py-8 md:py-10">
		<Tabs.Root
			onValueChange={onTabChange}
			bind:value={activeTab}
			class="flex w-full flex-1 flex-col"
		>
			<header class="mb-8 flex flex-wrap items-center justify-between gap-4">
				<div class="flex items-center gap-3">
					<div
						class="flex size-11 items-center justify-center rounded-2xl bg-primary text-primary-foreground shadow-lg shadow-primary/30"
					>
						<Zap class="size-5" />
					</div>
					<div>
						<h1 class="text-2xl font-extrabold tracking-tight">LightQR</h1>
						<p class="text-sm text-muted-foreground">Generate &amp; scan QR codes, instantly</p>
					</div>
				</div>
				<Tabs.List>
					<Tabs.Trigger class="px-7" value="generate">Generate</Tabs.Trigger>
					<Tabs.Trigger class="px-7" value="scan">Scan</Tabs.Trigger>
				</Tabs.List>
			</header>

			<!-- Display is gated on data-[state=active] so it never overrides the `hidden`
			     attribute bits-ui puts on the inactive panel. -->
			<Tabs.Content
				value="generate"
				class="flex-1 data-[state=active]:flex data-[state=active]:flex-col data-[state=active]:justify-center"
			>
				<Generate />
			</Tabs.Content>
			<Tabs.Content
				value="scan"
				class="flex-1 data-[state=active]:flex data-[state=active]:flex-col data-[state=active]:justify-center"
			>
				<Scan />
			</Tabs.Content>

			<p class="mt-8 text-center text-sm text-muted-foreground">
				Tip: higher error-correction survives logos &amp; scuffs · LightQR keeps everything in your
				browser
			</p>
		</Tabs.Root>
	</section>
</main>
