<script>
	import favicon from '$lib/assets/favicon.svg';

	let { children } = $props();
</script>

<svelte:head>
  <title>Dialog Senja</title>
  <link rel="icon" type="image/png" href="/logo.png" />
</svelte:head>

{@render children?.()}



