<script lang="ts">
  let file: File | null = null;
  let spectrogramUrl: string | null = null;
  let loading = false;

  async function uploadFile() {
    if (!file) return;

    loading = true;
    const formData = new FormData();
    formData.append("file", file);

    try {
      const res = await fetch("http://127.0.0.1:8000/spectrogram", {
        method: "POST",
        body: formData
      });

      if (!res.ok) throw new Error(`Server error: ${res.status}`);

      const blob = await res.blob();
      spectrogramUrl = URL.createObjectURL(blob);
    } catch (err) {
      console.error(err);
      alert("Failed to generate spectrogram.");
    } finally {
      loading = false;
    }
  }
</script>

<main class="p-8 font-sans">
  <div class="w-full text-center">
    <div class="my-4">
      <h1 class="text-6xl font-bold mb-1 tracking-widest" style="font-family: 'Alumni Sans Pinstripe', sans-serif;">HethFinder</h1>
      <h4>A tool for analyzing Hermit Thrush recordings</h4>
    </div>
    <div class="w-full flex flex-col">

      <input type="file" accept=".wav" on:change="{(e) => file = e.target.files?.[0] ?? null}" class="mb-4 mx-auto border border-red-500" />
      
      <button
          on:click="{uploadFile}"
          class="block px-4 py-2 bg-blue-950 hover:bg-blue-900 max-w-xl mx-auto text-white rounded-xl disabled:opacity-50"
          disabled={loading || !file}
        >
          {loading ? "Generating..." : "Upload & Generate Spectrogram"}
      </button>
    
  </div>
    {#if spectrogramUrl}
    <div class="mt-4">
      <h2 class="text-lg font-bold mb-2">Spectrogram</h2>
      <img src={spectrogramUrl} alt="Spectrogram" class="border border-blue object-cover w-full" />
    </div>
    {/if}
  </div>
</main>