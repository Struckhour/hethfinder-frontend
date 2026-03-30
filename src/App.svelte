<script>
  import { onMount } from "svelte";

  let imageUrl = "";
  let loading = true;

  onMount(async () => {
    loading = true;
    const res = await fetch("http://127.0.0.1:8000/plot/scatter");
    const blob = await res.blob();
    imageUrl = URL.createObjectURL(blob);
    loading = false;
  });
</script>

<main class="p-8 font-sans">
  <h1 class="text-2xl font-bold mb-4">Test Plot via fetch</h1>

  {#if loading}
    <p>Loading...</p>
  {:else}
    <img src={imageUrl} alt="Plot from backend" class="max-w-xl border rounded shadow" />
  {/if}
</main>