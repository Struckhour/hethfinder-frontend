<script lang="ts">
  
  import leftBorder from './assets/images/left-side-bg.png';
  import rightBorder from './assets/images/right-side-bg.png';
  import backWhite from './assets/images/background-white.png';


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

  async function runModel() {
    if (!file) return;

    loading = true;

    const formData = new FormData();
    formData.append("file", file);

    try {
      const res = await fetch("http://127.0.0.1:8000/predict", {
        method: "POST",
        body: formData
      });

      const json = await res.json();
      console.log(json);
      alert(`Detected ${json.detections.length} events`);
    } catch (err) {
      console.error(err);
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
          Run Model
        </button>
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