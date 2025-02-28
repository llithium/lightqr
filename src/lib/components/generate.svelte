<script lang="ts">
	import encodeQR from '@paulmillr/qr';
	const txt = 'Hello World';
	const svgElement = encodeQR(txt, 'svg');
	console.log(svgElement);
</script>

<div>
	<img src={svgElement} alt="" srcset="" />
	{@html svgElement}
</div>

<style>
</style>
