<script>
	import favicon from '$lib/assets/favicon.svg';

	let { children } = $props();
</script>

<svelte:head>
	<!-- <link rel="icon" href={favicon} /> -->
	 <link rel="icon" href="/logo.png" />
    <title>Dialog Senja</title>
</svelte:head>

{@render children?.()}
