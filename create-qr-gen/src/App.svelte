<script lang="ts">
  import { createQrPngDataUrl } from '@svelte-put/qr';
  import { onMount } from 'svelte';

  function getUrlParams() {
    const params = new URLSearchParams(window.location.search);
    return {
      text: params.get('text') || 'https://herenow.chat',
      moduleFill: params.get('moduleFill') || '#000000',
      backgroundFill: params.get('backgroundFill') || '#ffffff',
      shape: params.get('shape') || 'square',
      logoSrc: params.get('logoSrc') || '',
      errorCorrectionLevel: params.get('errorCorrectionLevel') || 'M',
      size: parseInt(params.get('size')) || 300,
      margin: parseInt(params.get('margin')) || 4,
      format: params.get('format') || 'png',
      title: params.get('title') || 'HereNow',
      titleFont: params.get('titleFont') || 'Merriweather',
      titleColor: params.get('titleColor') || '#000000',
      titlePosition: params.get('titlePosition') || 'above',
      titleSize: parseInt(params.get('titleSize')) || 40,
      titleWeight: params.get('titleWeight') || 'normal',
      titleStyle: params.get('titleStyle') || 'normal',
      textAlign: params.get('textAlign') || 'center',
      lineHeight: parseFloat(params.get('lineHeight')) || 1.2,
      letterSpacing: parseFloat(params.get('letterSpacing')) || 0,
    };
  }

  // -- Reactive Variables --
  let text = 'https://herenow.chat';
  let moduleFill = '#000000';
  let backgroundFill = '#ffffff';
  let shape: 'square' | 'circle' = 'square';
  let logoSrc = '';
  let errorCorrectionLevel: 'M' | 'L' | 'Q' | 'H' = 'M';
  let size = 300;
  let margin = 4;
  let format = 'png';

  let title = 'HereNow';
  let titleFont = 'Merriweather';
  let titleColor = '#000000';
  let titlePosition: 'above' | 'below' = 'above';
  let titleSize = 40;
  let titleWeight = 'normal';
  let titleStyle = 'normal';
  let textAlign: CanvasTextAlign = 'center'; // 'left' | 'center' | 'right' etc.
  let lineHeight = 1.2;  // used if you decide to implement multi-line
  let letterSpacing = 0;
  let error = '';
  let loading = false;

  let titleBorderColor = '#000000';
  let titleBorderThickness = 2;
  let titleBorderRadius = 10;
  let underline = false;
  let borderEnabled = false;
  let padding = 20;  // Space around the QR and text

  let fontLoading = false;

  // -- Canvas & Render Output --
  let canvas: HTMLCanvasElement;
  let qrSrc = '';

  // -- Font Loading --
  let availableFonts = [
    'Open Sans',
    'Roboto',
    'Lora',
    'Montserrat',
    'Playfair Display',
    'Nunito',
    'Raleway',
    'Merriweather',
    'Poppins',
    'Lobster',
    'Pacifico'
  ];
  let loadedFonts = new Set();

  function loadFont(font: string) {
    if (!loadedFonts.has(font)) {
      fontLoading = true;
      const link = document.createElement('link');
      link.href = `https://fonts.googleapis.com/css2?family=${font.replace(/ /g, '+')}&display=swap`;
      link.rel = 'stylesheet';
      link.onload = () => {
        loadedFonts.add(font);
        fontLoading = false;
      };
      link.onerror = () => {
        console.error(`Failed to load font: ${font}`);
        fontLoading = false;
      };
      document.head.appendChild(link);
    }
  }

  // -- Single Reactive Block --
  // Whenever ANY of these variables change, re-run loadFont(titleFont) and updateQR()
  // so that the preview & download reflect the latest values.
  $: {
    text;
    moduleFill;
    backgroundFill;
    shape;
    logoSrc;
    errorCorrectionLevel;
    margin;
    size;
    format;
    title;
    titleFont;
    titleColor;
    titlePosition;
    titleSize;
    titleWeight;
    titleStyle;
    textAlign;
    lineHeight;
    letterSpacing;
    titleBorderColor;
    titleBorderThickness;
    titleBorderRadius;
    underline;
    borderEnabled;
    padding;

    loadFont(titleFont);
  }

  // Reactive block to update QR code when font loading is complete
  $: if (!fontLoading) {
    updateQR();
  }

  // -- Main QR generation logic --
  async function updateQR() {
    loading = true;
    error = '';

    try {
      const baseQrSrc = await createQrPngDataUrl({
        data: text,
        width: size,
        height: size,
        backgroundFill,
        moduleFill,
        logo: logoSrc || undefined,
        shape,
        errorCorrectionLevel,
        margin
      });

      drawCanvasWithTitle(baseQrSrc, title);
    } catch (e) {
      console.error(e);
      error = 'Failed to generate QR code.';
    } finally {
      loading = false;
    }
  }

  // -- Draw the final composite of QR + title text on canvas --
  function drawCanvasWithTitle(baseQrSrc: string, customTitle: string) {
    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    const qrPadding = padding;
    const textMargin = 20;
    const maxTextWidth = size;

    ctx.font = `${titleStyle} ${titleWeight} ${titleSize}px ${titleFont}`;
    const textMetrics = ctx.measureText(customTitle);
    const titleHeight = textMetrics.actualBoundingBoxAscent + textMetrics.actualBoundingBoxDescent;

    const canvasWidth = size + margin * 2 + qrPadding * 2;
    const canvasHeight = size + qrPadding * 2 + titleHeight + textMargin;

    canvas.width = canvasWidth;
    canvas.height = canvasHeight;

    if (borderEnabled) {
      ctx.fillStyle = titleBorderColor;
      ctx.beginPath();
      ctx.roundRect(
        titleBorderThickness / 2,
        titleBorderThickness / 2,
        canvasWidth - titleBorderThickness,
        canvasHeight - titleBorderThickness,
        titleBorderRadius
      );
      ctx.lineWidth = titleBorderThickness;
      ctx.stroke();
    }

    ctx.fillStyle = backgroundFill;
    ctx.fillRect(
      borderEnabled ? titleBorderThickness : 0,
      borderEnabled ? titleBorderThickness : 0,
      canvasWidth - (borderEnabled ? titleBorderThickness * 2 : 0),
      canvasHeight - (borderEnabled ? titleBorderThickness * 2 : 0)
    );

    const img = new Image();
    img.src = baseQrSrc;
    img.onload = () => {
      const qrY = titlePosition === 'above' ? titleHeight + textMargin : qrPadding;
      ctx.drawImage(img, qrPadding + margin, qrY, size, size);

      const { fontSize, wrappedText } = calculateFontSize(ctx, customTitle, maxTextWidth, 100, titleSize, lineHeight);

      ctx.font = `${titleStyle} ${titleWeight} ${fontSize}px ${titleFont}`;
      ctx.fillStyle = titleColor;
      ctx.textAlign = textAlign;
      ctx.textBaseline = 'top';

      const x = getTextX(ctx, customTitle, canvasWidth);
      let textY = titlePosition === 'above' ? qrPadding : size + qrPadding + textMargin;

      wrappedText.forEach((line, index) => {
        ctx.fillText(line, x, textY + index * (fontSize * lineHeight));
        if (underline) {
          ctx.beginPath();
          ctx.moveTo(x, textY + index * (fontSize * lineHeight) + fontSize);
          ctx.lineTo(x + ctx.measureText(line).width, textY + index * (fontSize * lineHeight) + fontSize);
          ctx.strokeStyle = titleColor;
          ctx.lineWidth = 1.5;
          ctx.stroke();
        }
      });

      qrSrc = canvas.toDataURL(`image/${format}`);
    };

    img.onerror = () => {
      error = 'Failed to load QR code image.';
    };
  }

  // Utility function to calculate the required font size
  function calculateFontSize(ctx, text, maxWidth, maxHeight, initialFontSize, lineHeight) {
    let fontSize = initialFontSize;
    ctx.font = `${titleStyle} ${titleWeight} ${fontSize}px ${titleFont}`;
    let wrappedText = wrapText(ctx, text, maxWidth);

    while (wrappedText.length * fontSize * lineHeight > maxHeight) {
      fontSize -= 2;
      ctx.font = `${titleStyle} ${titleWeight} ${fontSize}px ${titleFont}`;
      wrappedText = wrapText(ctx, text, maxWidth);
    }

    return { fontSize, wrappedText };
  }

  // Utility function to wrap text
  function wrapText(ctx: CanvasRenderingContext2D, text: string, maxWidth: number) {
    const words = text.split(' ');
    const lines = [];
    let currentLine = words[0];

    for (let i = 1; i < words.length; i++) {
      const word = words[i];
      const width = ctx.measureText(currentLine + ' ' + word).width;
      if (width < maxWidth) {
        currentLine += ' ' + word;
      } else {
        lines.push(currentLine);
        currentLine = word;
      }
    }
    lines.push(currentLine);
    return lines;
  }

  // -- Utility: measure text width + spacing for alignment --
  function measureTextWidthWithSpacing(ctx: CanvasRenderingContext2D, text: string) {
    let total = 0;
    for (let i = 0; i < text.length; i++) {
      total += ctx.measureText(text[i]).width;
      if (i < text.length - 1) {
        total += letterSpacing;
      }
    }
    return total;
  }

  // -- Utility: get base X coordinate based on textAlign + width --
  function getTextX(ctx: CanvasRenderingContext2D, text: string, canvasWidth: number) {
    const textWidth = ctx.measureText(text).width;
    switch (textAlign) {
      case 'center':
        return canvasWidth / 2;
      case 'right':
      case 'end':
        return canvasWidth - 20; // small right padding
      default:
        // left or start
        return 20; // small left padding
    }
  }

  // -- Utility: draw text letter-by-letter to apply custom spacing --
  function drawSingleLineTextWithSpacing(
    ctx: CanvasRenderingContext2D,
    text: string,
    y: number,
    canvasWidth: number
  ) {
    const spacedWidth = measureTextWidthWithSpacing(ctx, text);
    let x: number;

    if (textAlign === 'center') {
      x = canvasWidth / 2 - spacedWidth / 2;
    } else if (textAlign === 'right' || textAlign === 'end') {
      x = canvasWidth - spacedWidth - 20;
    } else {
      x = 20;
    }

    for (let i = 0; i < text.length; i++) {
      const char = text[i];
      ctx.fillText(char, x, y);
      x += ctx.measureText(char).width + letterSpacing;
    }
  }

  // -- File upload for an optional logo in the QR --
  function handleFileUpload(event: Event) {
    const input = event.target as HTMLInputElement;
    const file = input?.files?.[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (e) => {
      const result = e?.target?.result;
      if (typeof result === 'string') {
        logoSrc = result;
      } else {
        console.error('FileReader result is not a string.');
      }
    };
    reader.readAsDataURL(file);
  }

  // -- Download the final QR code image --
  function downloadQR() {
    const link = document.createElement('a');
    link.href = qrSrc;
    link.download = `qr-code.${format}`;
    link.click();
  }

  onMount(() => {
    const params = getUrlParams();
    text = params.text;
    moduleFill = params.moduleFill;
    backgroundFill = params.backgroundFill;
    shape = params.shape;
    logoSrc = params.logoSrc;
    errorCorrectionLevel = params.errorCorrectionLevel;
    size = params.size;
    margin = params.margin;
    format = params.format;
    title = params.title;
    titleFont = params.titleFont;
    titleColor = params.titleColor;
    titlePosition = params.titlePosition;
    titleSize = params.titleSize;
    titleWeight = params.titleWeight;
    titleStyle = params.titleStyle;
    textAlign = params.textAlign;
    lineHeight = params.lineHeight;
    letterSpacing = params.letterSpacing;
  });
</script>

<style>
  main {
    max-width: 1200px;
    margin: 2rem auto;
    color: #f0f0f0;
    background: #1e1e1e;
    padding: 2rem;
    border-radius: 12px;
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.2);
  }

  h1 {
    text-align: center;
    margin-bottom: 2rem;
  }

  .container {
    display: flex;
    flex-wrap: wrap;
    gap: 1rem;
  }

  .column {
    flex: 1;
    min-width: 300px;
    padding: 1rem;
    box-sizing: border-box;
    background: #2e2e2e;
    border-radius: 8px;
  }

  .preview-column {
    top: 1rem;
  }

  label {
    display: block;
    margin: 1rem 0 0.5rem;
    font-weight: bold;
  }

  input,
  select,
  button {
    margin-bottom: 1rem;
    padding: 0.75rem;
    font-size: 1rem;
    width: 100%;
    max-width: 400px;
    box-sizing: border-box;
    border: 1px solid #ddd;
    border-radius: 6px;
  }

  input[type='checkbox'] {
    margin-bottom: 1rem;
    transform: scale(1.2);
  }

  button {
    background-color: #646cff;
    color: white;
    cursor: pointer;
    border: none;
  }

  button:hover {
    background-color: #535bf2;
  }

  .loading {
    font-size: 1.2rem;
    color: #888;
  }

  .error-box {
    color: #ffaaaa;
    margin-top: 1rem;
    font-weight: bold;
  }

  canvas {
    display: none; /* Hidden; only used to generate data URL */
  }

  img {
    margin-top: 1rem;
    max-width: 100%;
    height: auto;
    border-radius: var(--border-radius);
    border: var(--border-thickness) solid var(--border-color);
  }

  .sticky-preview {
    position: sticky;
    top: 1rem;
    display: inline-block;
  }

  button {
    margin-top: 1rem;
  }
</style>

<main>
  <h1>QR Code Generator</h1>
  <div class="container">
    <!-- Left Column: Inputs -->
    <div class="column">
      <label for="text-input">Enter URL or Text:</label>
      <input id="text-input" type="text" bind:value={text} />

      <label for="title-input">Title:</label>
      <input id="title-input" type="text" bind:value={title} />

      <label for="title-font">Title Font:</label>
      <select id="title-font" bind:value={titleFont}>
      {#each availableFonts as font}
      <option value={font} title={`Font: ${font}`}>{font}</option>
      {/each}
      </select>

      <label for="title-size">Title Size:</label>
      <input id="title-size" type="number" bind:value={titleSize} />

      <label for="title-color">Title Color:</label>
      <input id="title-color" type="color" bind:value={titleColor} />

      <label for="title-position">Title Position:</label>
      <select id="title-position" bind:value={titlePosition}>
      <option value="above">Above QR Code</option>
      <option value="below">Below QR Code</option>
      </select>

      <label for="foreground-color">Foreground Color:</label>
      <input id="foreground-color" type="color" bind:value={moduleFill} />

      <label for="background-color">Background Color:</label>
      <input id="background-color" type="color" bind:value={backgroundFill} />

      <label for="logo-input">Logo (optional):</label>
      <input id="logo-input" type="file" accept="image/*" on:change={handleFileUpload} />

      <label for="error-correction-level">Error Correction Level:</label>
      <select id="error-correction-level" bind:value={errorCorrectionLevel}>
      <option value="L">Low (L)</option>
      <option value="M">Medium (M)</option>
      <option value="Q">Quartile (Q)</option>
      <option value="H">High (H)</option>
      </select>

      <label for="border-enabled">Enable Border:</label>
      <input id="border-enabled" type="checkbox" bind:checked={borderEnabled} />

      <label for="border-color">Border Color:</label>
      <input id="border-color" type="color" bind:value={titleBorderColor} />

      <label for="border-thickness">Border Thickness:</label>
      <input id="border-thickness" type="number" bind:value={titleBorderThickness} />

      <label for="padding">Padding:</label>
      <input id="padding" type="number" bind:value={padding} />
    </div>

    <!-- Right Column: Preview/Result -->
    <div class="column preview-column">
      {#if loading}
        <p class="loading">Generating QR code...</p>
      {:else if error}
        <div class="error-box">❌ {error}</div>
      {:else if qrSrc}
        <!-- Wrap the image + button together -->
        <div class="sticky-preview">
          <img src={qrSrc} alt="Generated QR Code" />
          <button on:click={downloadQR}>Download QR Code</button>
        </div>
      {/if}
    </div>
  </div>

  <!-- Hidden canvas for final rendering -->
  <canvas bind:this={canvas}></canvas>
</main>