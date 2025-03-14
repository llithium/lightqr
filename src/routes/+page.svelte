<script lang="ts">
	import { goto } from '$app/navigation';
	import { page } from '$app/state';
	import * as Tabs from '$lib/components/ui/tabs/index.js';
	import Generate from '@/components/generate.svelte';
	import Scan from '@/components/scan.svelte';
	let activeTab = $state(page.url.searchParams.get('tab') || 'generate');

	function onTabChange() {
		const newURL = new URL(page.url);
		newURL.searchParams.set('tab', activeTab);
		goto(newURL);
	}
</script>

<main class="w-screen">
	<section class="container pt-4">
		<Tabs.Root onValueChange={onTabChange} bind:value={activeTab} class="w-full mx-auto">
			<Tabs.List class="w-full">
				<Tabs.Trigger class="w-1/2" value="generate">Generate</Tabs.Trigger>
				<Tabs.Trigger class="w-1/2" value="scan">Scan</Tabs.Trigger>
			</Tabs.List>
			<Tabs.Content value="generate"><Generate /></Tabs.Content>
			<Tabs.Content value="scan"><Scan /></Tabs.Content>
		</Tabs.Root>
	</section>
</main>
