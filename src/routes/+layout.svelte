<script lang="ts">
	import '../app.css';
	let { children } = $props();
</script>

<main class="h-screen overflow-auto p-8">
	{@render children()}
</main>
