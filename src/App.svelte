<script lang="ts">
  
  import leftBorder from './assets/images/left-side-bg.png';
  import rightBorder from './assets/images/right-side-bg.png';
  import backWhite from './assets/images/background-white.png';


  let file: File | null = null;
  let spectrogramUrl: string | null = null;
  let loading = false;
  let modelStatus = '';
  let detections: number[] = [];
  $: timeTicks = niceTimeTicks(durationSec);
  $: freqTicks = niceFreqTicks(minFreq, maxFreq);

  let durationSec: number = 60;   // TEMP: replace later with real value
  let sampleRate: number = 22050; // TEMP: replace later

  const nFft = 2048;
  const freqBinStart = 120;
  const freqBinEnd = 742;

  const minFreq = (freqBinStart * sampleRate) / nFft;
  const maxFreq = (freqBinEnd * sampleRate) / nFft;

  let pixelsPerSecond = 40; // default zoom


  const PX_PER_SEC = 30;
  const MIN_TRACK_WIDTH = 1000;

  $: trackWidth = durationSec
    ? Math.max(durationSec * PX_PER_SEC, MIN_TRACK_WIDTH)
    : MIN_TRACK_WIDTH;

  function formatTime(sec: number): string {
    const m = Math.floor(sec / 60);
    const s = Math.floor(sec % 60);
    return `${m}:${String(s).padStart(2, "0")}`;
  }

  function niceTimeTicks(duration: number): number[] {
    if (duration <= 30) return Array.from({ length: Math.floor(duration / 5) + 1 }, (_, i) => i * 5);
    if (duration <= 120) return Array.from({ length: Math.floor(duration / 10) + 1 }, (_, i) => i * 10);
    if (duration <= 600) return Array.from({ length: Math.floor(duration / 30) + 1 }, (_, i) => i * 30);
    return Array.from({ length: Math.floor(duration / 60) + 1 }, (_, i) => i * 60);
  }

  function niceFreqTicks(minF: number, maxF: number): number[] {
    const step = 1000;
    const start = Math.ceil(minF / step) * step;
    const ticks: number[] = [];
    for (let f = start; f <= maxF; f += step) {
      ticks.push(f);
    }
    return ticks;
  }




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
  class="p-8 pt-2 font-sans flex flex-col items-center min-h-screen bg-no-repeat bg-center bg-cover"
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
          on:change={async (e) => {
            const input = e.currentTarget as HTMLInputElement;
            file = input.files?.[0] ?? null;

            if (!file) return;

            // Create temporary audio element
            const audio = document.createElement("audio");
            const objectUrl = URL.createObjectURL(file);

            audio.src = objectUrl;

            // Wait for metadata (this is the key part)
            await new Promise<void>((resolve, reject) => {
              audio.addEventListener("loadedmetadata", () => resolve(), { once: true });
              audio.addEventListener("error", () => reject(new Error("Failed to read metadata")), { once: true });
            });

            // Set duration
            durationSec = audio.duration;
            console.log("Duration:", durationSec);
            // Clean up
            URL.revokeObjectURL(objectUrl);
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

    
  </div>
  {#if spectrogramUrl}
    <div class="my-4 py-4 w-full">
      <h2 class="text-lg font-bold mb-2">
        {file ? file.name : "Spectrogram"}
      </h2>

      <div class="flex m-2 items-stretch w-full">

        <!-- Y AXIS -->
        <div class="relative w-14 shrink-0 border-r border-gray-300">
          {#each freqTicks as freq}
            <div
              class="absolute right-1 text-xs text-gray-700 -translate-y-1/2"
              style={`top: ${100 - ((freq - minFreq) / (maxFreq - minFreq)) * 100}%;`}
            >
              {Math.round(freq / 1000)}k
            </div>
          {/each}
        </div>

        <!-- MAIN AREA -->
        <div class="flex-1 min-w-0">
          <div class="overflow-x-auto overflow-y-hidden">
            <div style={`width: ${trackWidth}px;`}>

              <!-- SPECTROGRAM -->
              <div class="border-t-4 border-b-4 border-black bg-white">
                <img
                  src={spectrogramUrl}
                  alt="Spectrogram"
                  class="block max-w-none"
                  style={`width: ${trackWidth}px; height: 450px;`}
                />
              </div>

              <!-- X AXIS -->
              <div class="relative h-8 mt-1 border-t border-gray-300">
                {#each timeTicks as t}
                  <div
                    class="absolute text-xs text-gray-700 -translate-x-1/2 top-1 whitespace-nowrap"
                    style={`left: ${(t / durationSec) * trackWidth}px;`}
                  >
                    {formatTime(t)}
                  </div>
                {/each}
              </div>

            </div>
          </div>
        </div>
      </div>
    </div>
    {/if}
</main>