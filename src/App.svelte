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
  let isMobile = false;

  const cvs = document.createElement('canvas');
  const ctx = cvs.getContext('2d', { willReadFrequently: true });

  onMount(() => {
    isMobile = window.innerWidth <= 768;
    
    if (windowEl && !isMobile) {
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

    // Listen for resize events
    const handleResize = () => {
      const wasMobile = isMobile;
      isMobile = window.innerWidth <= 768;
      
      // If switching from mobile to desktop, recenter the window
      if (wasMobile && !isMobile && windowEl) {
        left = (window.innerWidth - 550) / 2;
        top = (window.innerHeight - 480) / 2;
      }
    };
    
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
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
    if (isMobile) return; // Disable dragging on mobile
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
    class:mobile={isMobile}
    bind:this={windowEl}
    style={isMobile ? '' : `position: absolute; top: ${top}px; left: ${left}px; min-width: 550px; min-height: 480px; resize: both; overflow: hidden;`}
  >
    <div class="title-bar" on:mousedown={onMouseDown} style={isMobile ? '' : 'cursor: move;'}>
      <div class="title-bar-text">Windows 98 JPEG Artifact Generator</div>
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
          <button class="button default" on:click={resetImage}>Reset</button>
          <button class="button default" on:click={downloadImage}>Download</button>
        </fieldset>

        <div class="status-bar">
          <p class="status-bar-field">Tip: try low base quality and many loops for chunky artifacts.</p>
          <p class="status-bar-field">Image is processed locally in your browser.</p>
        </div>
      </div>
    </div>
    <a
      class="watermark"
      href="https://github.com/hritsh"
      target="_blank"
      rel="noopener noreferrer"
      title="GitHub: hritsh"
    >
      <svg class="github-icon" viewBox="0 0 24 24" width="16" height="16" aria-hidden="true">
        <polygon points="23 9 23 15 22 15 22 17 21 17 21 19 20 19 20 20 19 20 19 21 18 21 18 22 16 22 16 23 15 23 15 18 14 18 14 17 15 17 15 16 17 16 17 15 18 15 18 14 19 14 19 9 18 9 18 6 16 6 16 7 15 7 15 8 14 8 14 7 10 7 10 8 9 8 9 7 8 7 8 6 6 6 6 9 5 9 5 14 6 14 6 15 7 15 7 16 9 16 9 18 7 18 7 17 6 17 6 16 4 16 4 17 5 17 5 19 6 19 6 20 9 20 9 23 8 23 8 22 6 22 6 21 5 21 5 20 4 20 4 19 3 19 3 17 2 17 2 15 1 15 1 9 2 9 2 7 3 7 3 5 4 5 4 4 5 4 5 3 7 3 7 2 9 2 9 1 15 1 15 2 17 2 17 3 19 3 19 4 20 4 20 5 21 5 21 7 22 7 22 9 23 9"/>
      </svg>
      <span>github.com/hritsh</span>
    </a>
  </div>
</div>