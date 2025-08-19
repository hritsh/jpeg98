<script>
  import { onMount } from 'svelte';
  import imageURL from './assets/lion.jpeg';

  let factor = 0.2;
  let loopCount = 100;
  let inImg;
  let outImg;

  let windowEl;
  let top;
  let left;
  let isDragging = false;
  let dragOffsetX, dragOffsetY;

  let isProcessing = false;
  let processProgress = 0;
  let processedCount = 0;
  let totalIterations = 0;

  const cvs = document.createElement('canvas');
  const ctx = cvs.getContext('2d', { willReadFrequently: true });

  onMount(() => {
    if (windowEl) {
      left = (window.innerWidth - 550) / 2;
      top = (window.innerHeight - 480) / 2;
    }
    if (inImg && outImg) {
      inImg.src = imageURL;
      outImg.src = imageURL;
      outImg.onload = () => {
        cvs.width = outImg.naturalWidth;
        cvs.height = outImg.naturalHeight;
        ctx.drawImage(outImg, 0, 0);
      };
    }
  });

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

  function compressOnce() {
    if (!outImg?.src) return;
    const q = clamp(factor + Math.random() * 0.1, 0, 0.5);
    outImg.src = cvs.toDataURL('image/jpeg', q);
    outImg.onload = () => {
      ctx.drawImage(outImg, 0, 0);
    };
  }

  function autoLoop() {
    if (isProcessing) return;
    
    const n = Number(loopCount) || 0;
    if (n <= 0) return;

    isProcessing = true;
    processedCount = 0;
    totalIterations = n;
    processProgress = 0;

    processIterations();
  }

  function processIterations() {
    const batchSize = 5;
    const endBatch = Math.min(processedCount + batchSize, totalIterations);
    
    for (let i = processedCount; i < endBatch; i++) {
      compressOnce();
      processedCount++;
    }
    
    processProgress = (processedCount / totalIterations) * 100;

    if (processedCount < totalIterations) {
      requestAnimationFrame(processIterations);
    } else {
      isProcessing = false;
      processProgress = 0;
    }
  }

  function resetImage() {
    if (!inImg?.src) return;
    outImg.src = inImg.src;
    outImg.onload = () => {
      cvs.width = outImg.naturalWidth;
      cvs.height = outImg.naturalHeight;
      ctx.drawImage(outImg, 0, 0);
    };
  }

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
    style="position: absolute; top: {top}px; left: {left}px; min-width: 550px; min-height: 480px; resize: both; overflow: hidden;"
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
      <div class="controls">
        <div class="field-row">
          <label for="file">Open image:</label>
          <input id="file" type="file" accept="image/*" on:change={handleFile} />
        </div>

        <div class="field-row">
          <label for="factor">Base compression</label>
          <input id="factor" type="range" min="0" max="0.5" step="0.01" bind:value={factor} />
          <span class="value">{factor.toFixed(2)}</span>
          <button class="button" on:click={compressOnce}>Create Artifacts</button>
        </div>

        <div class="field-row">
          <label for="loop">Create artifacts</label>
          <input id="loop" type="number" min="1" max="2000" step="1" bind:value={loopCount} style="width: 90px;" />
          <span>times</span>
          <button class="button default" on:click={autoLoop} disabled={isProcessing}>
            {isProcessing ? 'Processing...' : 'Auto looper'}
          </button>
          <button class="button reset-button" on:click={resetImage}>Reset image</button>
        </div>

        {#if isProcessing}
          <div class="field-row progress-row">
            <label>Progress:</label>
            <div class="progress-container">
              <div class="progress-bar" style="width: {processProgress}%"></div>
            </div>
            <span class="progress-text">{processedCount}/{totalIterations}</span>
          </div>
        {/if}
      </div>

      <div class="content">
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
</div>

<style>
  .app {
    padding: 16px;
  }
  .window {
    font-size: 1.1em;
    display: flex;
    flex-direction: column;
    width: 550px;
    height: 480px;
  }
  .window-body {
    flex-grow: 1;
    display: flex;
    flex-direction: column;
    padding: 8px;
    overflow: hidden;
  }
  .controls {
    flex-shrink: 0;
  }
  .content {
    flex-grow: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
  }
  .field-row {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 8px;
  }
  .progress-row {
    margin-bottom: 12px;
  }
  .reset-button {
    margin-left: auto;
  }
  .progress-container {
    flex-grow: 1;
    height: 16px;
    background: #c0c0c0;
    border: 2px inset #c0c0c0;
    position: relative;
    min-width: 120px;
  }
  .progress-bar {
    height: 100%;
    background: linear-gradient(90deg, #0080ff 0%, #004080 100%);
    transition: width 0.1s ease;
  }
  .progress-text {
    font-size: 0.9em;
    min-width: 60px;
    text-align: center;
  }
  .images {
    flex-grow: 1;
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 8px;
    min-height: 0;
  }
  .panel {
    border: 1px solid #000;
    padding: 8px;
    background: #fff;
    display: flex;
    flex-direction: column;
    min-height: 0;
  }
  .panel-title {
    font-weight: bold;
    margin-bottom: 6px;
    flex-shrink: 0;
  }
  img {
    max-width: 100%;
    flex-grow: 1;
    object-fit: contain;
    image-rendering: pixelated;
    border: 1px solid #000;
    background: #c0c0c0;
    min-height: 0;
  }
  .value {
    width: 40px;
    text-align: right;
    display: inline-block;
  }
  .status-bar {
    flex-shrink: 0;
    margin-top: 8px;
  }
</style>