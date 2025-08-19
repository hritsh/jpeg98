<script>
  // State
  let factor = 0.2; // base JPEG quality (0..0.5)
  let loopCount = 100; // iterations for auto-loop
  let inImg; // <img> ref for input/original
  let outImg; // <img> ref for output/processed

  // Window state
  let windowEl;
  let top = 20;
  let left = 20;
  let isDragging = false;
  let dragOffsetX, dragOffsetY;

  // Offscreen canvas
  const cvs = document.createElement('canvas');
  const ctx = cvs.getContext('2d', { willReadFrequently: true });

  // Load from file input
  function handleFile(e) {
    const file = e.target.files?.[0];
    if (!file) return;

    const src = URL.createObjectURL(file);
    inImg.src = src;
    outImg.src = src;

    outImg.onload = () => {
      cvs.width = outImg.naturalWidth;
      cvs.height = outImg.naturalHeight;
      ctx.drawImage(outImg, 0, 0);
    };
  }

  // Apply one round of JPEG compression
  function compressOnce() {
    if (!outImg?.src) return;
    const q = clamp(factor + Math.random() * 0.1, 0, 0.5);
    outImg.src = cvs.toDataURL('image/jpeg', q);
    outImg.onload = () => {
      ctx.drawImage(outImg, 0, 0);
    };
  }

  // Loop compression N times
  function autoLoop() {
    const n = Number(loopCount) || 0;
    for (let i = 0; i < n; i++) compressOnce();
  }

  // Reset output to original
  function resetImage() {
    if (!inImg?.src) return;
    outImg.src = inImg.src;
    outImg.onload = () => {
      cvs.width = outImg.naturalWidth;
      cvs.height = outImg.naturalHeight;
      ctx.drawImage(outImg, 0, 0);
    };
  }

  // Utilities
  function clamp(v, min, max) {
    return Math.max(min, Math.min(max, v));
  }

  function onMouseDown(event) {
    isDragging = true;
    const rect = windowEl.getBoundingClientRect();
    dragOffsetX = event.clientX - rect.left;
    dragOffsetY = event.clientY - rect.top;
    window.addEventListener('mousemove', onMouseMove);
    window.addEventListener('mouseup', onMouseUp);
  }

  function onMouseMove(event) {
    if (!isDragging) return;
    left = event.clientX - dragOffsetX;
    top = event.clientY - dragOffsetY;
  }

  function onMouseUp() {
    isDragging = false;
    window.removeEventListener('mousemove', onMouseMove);
    window.removeEventListener('mouseup', onMouseUp);
  }
</script>

<div class="app">
  <div
    class="window"
    bind:this={windowEl}
    style="position: absolute; top: {top}px; left: {left}px; min-width: 550px; resize: both; overflow: auto;"
  >
    <div class="title-bar" on:mousedown={onMouseDown} style="cursor: move;">
      <div class="title-bar-text">JPEG Artifact Generator</div>
      <div class="title-bar-controls">
        <button aria-label="Minimize"></button>
        <button aria-label="Maximize"></button>
        <button aria-label="Close"></button>
      </div>
    </div>

    <div class="window-body">
      <div class="field-row">
        <label for="file">Open image:</label>
        <input id="file" type="file" accept="image/*" on:change={handleFile} />
      </div>

      <div class="field-row">
        <label for="factor">Base compression</label>
        <input id="factor" type="range" min="0" max="0.5" step="0.01" bind:value={factor} />
        <span class="value">{factor.toFixed(2)}</span>
        <button class="button" on:click={compressOnce}>Create JPEG artifacts</button>
      </div>

      <div class="field-row">
        <label for="loop">Create artifacts</label>
        <input id="loop" type="number" min="1" max="2000" step="1" bind:value={loopCount} style="width: 90px;" />
        <span>times</span>
        <button class="button" on:click={autoLoop}>Auto looper</button>
        <button class="button reset-button" on:click={resetImage}>Reset image</button>
      </div>

      <div class="images">
        <div class="panel">
          <div class="panel-title">Input</div>
          <img bind:this={inImg} alt="input" />
        </div>
        <div class="panel">
          <div class="panel-title">Output</div>
          <img bind:this={outImg} alt="output" />
        </div>
      </div>

      <div class="status-bar">
        <p class="status-bar-field">Tip: try low base quality and many loops for chunky artifacts.</p>
        <p class="status-bar-field">Image is processed locally in your browser.</p>
      </div>
    </div>
  </div>
</div>

<style>
  .app {
    padding: 16px;
  }
  .window {
    font-size: 1.1em;
  }
  .field-row {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 8px;
  }
  .reset-button {
    margin-left: auto;
  }
  .images {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 8px;
    margin-top: 12px;
  }
  .panel {
    border: 1px solid #000;
    padding: 8px;
    background: #fff;
  }
  .panel-title {
    font-weight: bold;
    margin-bottom: 6px;
  }
  img {
    max-width: 100%;
    height: auto;
    image-rendering: pixelated; /* enhances the crunchy vibe */
    border: 1px solid #000;
    background: #c0c0c0;
  }
  .value {
    width: 40px;
    text-align: right;
    display: inline-block;
  }
  .status-bar {
    margin-top: 10px;
  }
</style>
