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

  function downloadImage() {
    if (!outImg?.src) return;
    
    const link = document.createElement('a');
    link.download = 'jpeg-artifacts.jpg';
    link.href = outImg.src;
    link.click();
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
      <fieldset class="controls">
        <div class="control-grid">
          <label for="file">Open image:</label>
          <input id="file" type="file" accept="image/*" on:change={handleFile} />
          <div></div>
          
          <label for="factor">Base compression:</label>
          <input id="factor" type="range" min="0" max="0.5" step="0.01" bind:value={factor} />
          <div class="control-group">
            <span class="value">{factor.toFixed(2)}</span>
            <button class="button" on:click={compressOnce}>Run Once</button>
          </div>
          
          <label for="loop">Create artifacts:</label>
          {#if isProcessing}
            <div class="progress-indicator segmented" style="height: 18px; min-width: 120px;">
              <span class="progress-indicator-bar" style="width: {processProgress}%;"></span>
            </div>
            <div class="control-group">
              <span class="progress-text">{processedCount}/{totalIterations}</span>
              <button class="button" disabled>Processing...</button>
            </div>
          {:else}
            <input id="loop" type="range" min="1" max="2000" step="1" bind:value={loopCount} />
            <div class="control-group">
              <span class="value">{loopCount} times</span>
              <button class="button" on:click={autoLoop}>Run Loop</button>
            </div>
          {/if}
        </div>
      </fieldset>

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

        <fieldset class="bottom-controls">
          <button class="button" on:click={resetImage}>Reset</button>
          <button class="button" on:click={downloadImage}>Download</button>
        </fieldset>

        <div class="status-bar">
          <p class="status-bar-field">Tip: try low base quality and many loops for chunky artifacts.</p>
          <p class="status-bar-field">Image is processed locally in your browser.</p>
        </div>
      </div>
    </div>
  </div>
</div>