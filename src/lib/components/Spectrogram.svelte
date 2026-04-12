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

      if (!res.ok) {
        throw new Error(`Server error: ${res.status}`);
      }

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

<div class="p-4 space-y-4">
  <input type="file" accept=".wav" on:change="{(e) => {
      const input = e.currentTarget as HTMLInputElement;
      file = input.files?.[0] ?? null;
    }}" 
  />
  <button
    on:click="{uploadFile}"
    class="px-4 py-2 bg-blue-600 text-white rounded disabled:opacity-50"
    disabled={loading || !file}
  >
    {loading ? "Generating..." : "Upload & Generate Spectrogram"}
  </button>

  {#if spectrogramUrl}
    <div class="mt-4">
      <h2 class="text-lg font-bold mb-2">Spectrogram</h2>
      <img src={spectrogramUrl} alt="Spectrogram" class="border" />
    </div>
  {/if}
</div>