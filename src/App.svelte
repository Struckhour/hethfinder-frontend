<script lang="ts">
  
  import leftBorder from './assets/images/left-side-bg.png';
  import rightBorder from './assets/images/right-side-bg.png';
  import backWhite from './assets/images/background-white.png';


  let file: File | null = null;
  let spectrogramUrl: string | null = null;
  let loading = false;
  let modelStatus = '';
  let detections: number[] = [];


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

  async function runModel() {
    if (!file) return;

    loading = true;
    modelStatus = 'Uploading file...';
    detections = [];

    const formData = new FormData();
    formData.append("file", file);

    try {
      const res = await fetch("http://127.0.0.1:8000/predict", {
        method: "POST",
        body: formData
      });

      if (!res.ok) throw new Error(`Server error: ${res.status}`);

      const { job_id } = await res.json();

      let status = 'queued';

      while (status !== 'done') {
        await new Promise((r) => setTimeout(r, 2000));

        const statusRes = await fetch(`http://127.0.0.1:8000/status/${job_id}`);
        if (!statusRes.ok) throw new Error(`Status error: ${statusRes.status}`);

        const statusJson = await statusRes.json();
        status = statusJson.status;
        modelStatus = `Model status: ${status}`;

        if (status === 'error') {
          throw new Error(statusJson.error || 'Job failed');
        }
      }

      const resultRes = await fetch(`http://127.0.0.1:8000/result/${job_id}`);
      if (!resultRes.ok) throw new Error(`Result error: ${resultRes.status}`);

      const resultJson = await resultRes.json();
      detections = resultJson.detections ?? [];
      modelStatus = `Done. Found ${detections.length} detections.`;
    } catch (err) {
      console.error(err);
      modelStatus = 'Prediction failed.';
      alert("Prediction failed.");
    } finally {
      loading = false;
    }
  }


</script>

<main
  class="p-8 font-sans flex h-screen justify-center bg-no-repeat bg-center bg-cover"
  style="background-image: url('{backWhite}');"
>
  <div class="grid grid-cols-1 md:grid-cols-[minmax(50px,auto)_1fr_minmax(50px,auto)] items-start gap-4 max-w-[1000px] mx-auto">
    
    <!-- Left border image -->
    <div class="hidden md:block">
      <img src={leftBorder} alt="Left border" class="h-full object-contain max-h-screen" />
    </div>

    <!-- Main content -->
    <div class="text-center">
      <div class="my-4">
        <h1 class="text-6xl font-bold mb-1 tracking-widest" style="font-family: 'Alumni Sans Pinstripe', sans-serif;">HethFinder</h1>
        <h4>A tool for analyzing Hermit Thrush recordings</h4>
      </div>

      <div class="w-full flex flex-col items-center space-y-4">
        <input
          type="file"
          accept=".wav"
          on:change={(e) => {
            const input = e.currentTarget as HTMLInputElement;
            file = input.files?.[0] ?? null;
          }}
        />
        
        <button
          on:click="{uploadFile}"
          class="block px-4 py-2 bg-blue-950 hover:bg-blue-900 max-w-xl mx-auto text-white rounded-xl disabled:opacity-50"
          disabled={loading || !file}
        >
          {loading ? "Generating..." : "Generate Spectrogram"}
        </button>
        <button on:click={runModel} class="bg-blue-500 hover:bg-blue-400 text-white font-bold py-2 px-4 rounded disabled:opacity-50" disabled={loading || !file}>
          Find Songs
        </button>
        {#if modelStatus}
          <p class="text-sm mt-2">{modelStatus}</p>
        {/if}
        {#if detections.length > 0}
          <div class="mt-4 w-full max-w-xl text-left">
            <h3 class="font-bold mb-2">Detections ({detections.length})</h3>
            <ul class="text-sm max-h-48 overflow-y-auto border p-3 rounded bg-white/70">
              {#each detections as detection}
                <li>{detection}</li>
              {/each}
            </ul>
          </div>
        {/if}
      </div>


    </div>

    <!-- Right border image -->
    <div class="hidden md:block">
      <img src={rightBorder} alt="Right border" class="h-full object-contain max-h-screen" />
    </div>

    {#if spectrogramUrl}
    <div class="my-4 py-4 col-span-1 md:col-span-3 w-full">
      <h2 class="text-lg font-bold mb-2">{file ? file.name : "Spectrogram"}</h2>
      <img src={spectrogramUrl} alt="Spectrogram" class="border-t-4 border-b-4 border-black object-cover w-full" />
    </div>
    {/if}
  </div>
</main>