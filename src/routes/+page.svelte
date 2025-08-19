<script lang="ts">
  let imageSrc: string | null = null;
  let canvasEl: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D;

  let originalImg: HTMLImageElement | null = null;
  let originalData: ImageData | null = null;

  let sortDirection: "horizontal" | "vertical" = "horizontal";
  let sortBy:
    | "brightness"
    | "red"
    | "green"
    | "blue"
    | "hue"
    | "saturation"
    | "lightness" = "brightness";
  let threshold = 0;
  let jitter = 0;
  let reverse = false;
  let blend = 100;

  function handleFileUpload(event: Event) {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (e) => {
      imageSrc = e.target?.result as string;
      originalImg = new Image();
      originalImg.src = imageSrc;
      originalImg.onload = () => {
        canvasEl.width = originalImg!.width;
        canvasEl.height = originalImg!.height;
        ctx = canvasEl.getContext("2d")!;
        ctx.drawImage(originalImg!, 0, 0);
        originalData = ctx.getImageData(0, 0, canvasEl.width, canvasEl.height);
      };
    };
    reader.readAsDataURL(file);
  }

  function pixelSort() {
    if (!ctx) return;
    const { width, height } = canvasEl;

    const imgData = ctx.getImageData(0, 0, width, height);
    const data = imgData.data;

    if (sortDirection === "horizontal") {
      for (let y = 0; y < height; y++) {
        const selected: number[][] = [];
        const idxs: number[] = [];

        for (let x = 0; x < width; x++) {
          const i = (y * width + x) * 4;
          const px = [data[i], data[i + 1], data[i + 2], data[i + 3]];
          if (passesThreshold(px)) {
            selected.push(px);
            idxs.push(i);
          }
        }

        selected.sort((a, b) => {
          let diff = pixelValue(a) - pixelValue(b);
          if (jitter > 0) diff += (Math.random() - 0.5) * jitter * 50;
          return diff;
        });
        if (reverse) selected.reverse();

        for (let k = 0; k < idxs.length; k++) {
          const i = idxs[k];
          const p = selected[k];
          data[i] = p[0];
          data[i + 1] = p[1];
          data[i + 2] = p[2];
          data[i + 3] = p[3];
        }
      }
    } else {
      for (let x = 0; x < width; x++) {
        const selected: number[][] = [];
        const idxs: number[] = [];

        for (let y = 0; y < height; y++) {
          const i = (y * width + x) * 4;
          const px = [data[i], data[i + 1], data[i + 2], data[i + 3]];
          if (passesThreshold(px)) {
            selected.push(px);
            idxs.push(i);
          }
        }

        selected.sort((a, b) => {
          let diff = pixelValue(a) - pixelValue(b);
          if (jitter > 0) diff += (Math.random() - 0.5) * jitter * 50;
          return diff;
        });
        if (reverse) selected.reverse();

        for (let k = 0; k < idxs.length; k++) {
          const i = idxs[k];
          const p = selected[k];
          data[i] = p[0];
          data[i + 1] = p[1];
          data[i + 2] = p[2];
          data[i + 3] = p[3];
        }
      }
    }

    if (blend < 100 && originalData) {
      const orig = originalData.data;
      for (let i = 0; i < data.length; i++) {
        data[i] = Math.round(
          (blend / 100) * data[i] + (1 - blend / 100) * orig[i]
        );
      }
    }

    ctx.putImageData(imgData, 0, 0);
  }

  function resetImage() {
    if (originalImg && ctx) {
      ctx.drawImage(originalImg, 0, 0);
    }
  }

  function pixelValue([r, g, b, _]: number[]) {
    switch (sortBy) {
      case "red":
        return r;
      case "green":
        return g;
      case "blue":
        return b;
      case "brightness":
        return (r + g + b) / 3;
      case "hue":
        return rgbToHsl(r, g, b)[0] * 360;
      case "saturation":
        return rgbToHsl(r, g, b)[1] * 100;
      case "lightness":
        return rgbToHsl(r, g, b)[2] * 100;
    }
  }

  function passesThreshold([r, g, b, _]: number[]) {
    if (threshold <= 0) return true;
    const bright = (r + g + b) / 3;
    return bright >= threshold;
  }

  function rgbToHsl(r: number, g: number, b: number): [number, number, number] {
    r /= 255;
    g /= 255;
    b /= 255;
    const max = Math.max(r, g, b),
      min = Math.min(r, g, b);
    let h = 0,
      s = 0,
      l = (max + min) / 2;

    if (max !== min) {
      const d = max - min;
      s = l > 0.5 ? d / (2 - max - min) : d / (max + min);
      switch (max) {
        case r:
          h = (g - b) / d + (g < b ? 6 : 0);
          break;
        case g:
          h = (b - r) / d + 2;
          break;
        case b:
          h = (r - g) / d + 4;
          break;
      }
      h /= 6;
    }
    return [h, s, l];
  }
</script>

<div class="controls">
  <label for="file">
    Upload
    <input
      type="file"
      accept="image/*"
      on:change={handleFileUpload}
      id="file"
    />
  </label>

  <label for="direction">
    Direction:
    <select bind:value={sortDirection} id="direction">
      <option value="horizontal">Horizontal</option>
      <option value="vertical">Vertical</option>
    </select>
  </label>

  <label for="sort">
    Sort by:
    <select bind:value={sortBy} id="sort">
      <option value="brightness">Brightness</option>
      <option value="red">Red</option>
      <option value="green">Green</option>
      <option value="blue">Blue</option>
      <option value="hue">Hue</option>
      <option value="saturation">Saturation</option>
      <option value="lightness">Lightness</option>
    </select>
  </label>

  <label for="threshold">
    Threshold:
    <input
      type="range"
      min="0"
      max="255"
      bind:value={threshold}
      id="threshold"
    />
    {threshold}
  </label>

  <label for="jitter">
    Jitter:
    <input
      type="range"
      min="0"
      max="1"
      step="0.01"
      bind:value={jitter}
      id="jitter"
    />
    {Math.round(jitter * 100)}%
  </label>

  <label for="reverse">
    Reverse:
    <input type="checkbox" bind:checked={reverse} id="reverse" />
  </label>

  <label for="blend">
    Blend:
    <input type="range" min="0" max="100" bind:value={blend} id="blend" />
    {blend}%
  </label>

  <button on:click={pixelSort}>Sort</button>
  <button on:click={resetImage} disabled={!originalImg}>Reset</button>
</div>

<canvas bind:this={canvasEl} class={originalImg ? "" : "hide"}></canvas>

<style>
  .controls {
    padding: 0.8rem;
    max-width: max-content;
    display: flex;
    flex-wrap: wrap;
    gap: 1rem;
    margin: 1rem auto;
    justify-content: center;
    border: 1px solid white;
    border-radius: 3px;
    position: sticky;
    top: 1rem;
    background: black;
  }
  button {
    width: 3.5rem;
  }
  canvas {
    display: block;
    margin: auto;
    max-width: calc(100% - 32px);
    border: 1px solid #ccc;
  }
  canvas.hide {
    display: none;
  }
  label {
    display: flex;
    flex-direction: column;
    font-size: 0.9rem;
  }
  label[for="file"] {
    background: white;
    color: black;
    padding: 0.8rem;
    display: grid;
    place-items: center;
  }
  input#file {
    display: none;
  }
</style>
