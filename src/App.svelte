<script lang="ts">
  import { onMount } from "svelte";

  let canvas: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D | null;

  let isDrawing = false;
  let lastX = 0;
  let lastY = 0;
  let color = "#FFFFFF";
  let minSize = 1;
  let maxSize = 100;
  let sizeValue = 5;
  let erase: boolean = false;

  function toggleErase() {
    erase = !erase;
  }

  function chooseColor(event: Event) {
    const target = event.target as HTMLInputElement;
    color = target.value;
  }

  function clear() {
    ctx?.clearRect(0, 0, canvas.width, canvas.height);
  }

  function startDrawing(event: MouseEvent) {
    isDrawing = true;
    [lastX, lastY] = [event.offsetX, event.offsetY];
  }

  function draw(event: MouseEvent) {
  if (!isDrawing || !ctx) return;

  if (erase) {
    ctx.globalCompositeOperation = 'destination-out';
  } else {
    ctx.globalCompositeOperation = 'source-over';
  }

  ctx.strokeStyle = color;
  ctx.lineWidth = sizeValue;
  ctx.lineJoin = "round";
  ctx.lineCap = "round";

  ctx.beginPath();
  ctx.moveTo(lastX, lastY);
  ctx.lineTo(event.offsetX, event.offsetY);
  ctx.stroke();
  [lastX, lastY] = [event.offsetX, event.offsetY];
}

  function stopDrawing() {
    isDrawing = false;
  }

  function saveDrawing() {
    if (!canvas) return;
    const link = document.createElement("a");
    link.href = canvas.toDataURL("image/png");
    link.download = "drawing.png";
    link.click();
  }

  onMount(() => {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
    ctx = canvas.getContext("2d");
  });
</script>

<main>
  <canvas
    bind:this={canvas}
    on:mousedown={startDrawing}
    on:mousemove={draw}
    on:mouseup={stopDrawing}
    on:mouseout={stopDrawing}
    on:blur={stopDrawing}
  ></canvas>
  <p id="color-text">Color: {color}</p>
  <input
    type="color"
    id="color-picker"
    on:input={chooseColor}
    bind:value={color}
  />
  <button id="clear-btn" on:click={clear}>Clear</button>
  <p id="range-text">Stroke Size: {sizeValue}</p>
  <input
    type="range"
    id="line-range"
    min={minSize}
    max={maxSize}
    bind:value={sizeValue}
  />
  <button on:click={toggleErase} id="erase-btn"
    >Erase: {erase ? "on" : "off"}
    </button>
    <button id="save-btn" on:click={saveDrawing}>Save as Image</button>
</main>

<style>
  main {
    background-color: black;
    margin: 0;
    padding: 0;
    height: 100%;
    overflow: hidden;
  }

  canvas {
    display: block;
    width: 100vw;
    height: 100vh;
  }

  #color-text {
    position: absolute;
    top: 10px;
    left: 10px;
    font-size: 18px;
    color: #fff;
    font-family: Consolas;
  }

  #color-picker {
    position: absolute;
    top: 50px;
    left: 10px;
    width: 75px;
    height: 50px;
    border-radius: 5px;
  }

  #clear-btn {
    position: absolute;
    bottom: 10px;
    right: 10px;
    font-size: 18px;
    background-color: rgb(50, 50, 50);
    color: #fff;
    padding: 10px 20px;
    border: none;
    cursor: pointer;
    transition: 0.5s ease;
    font-family: Consolas;
  }
  #line-range {
    position: absolute;
    left: 10px;
    bottom: 10px;
  }
  #range-text {
    position: absolute;
    left: 10px;
    bottom: 20px;
    color: white;
    font-family: Consolas;
  }

  #erase-btn {
    color: white;
    position: absolute;
    top: 150px;
    left: 10px;
    background-color: gray;
    width: 75px;
    height: 25px;
    font-family: monospace;
  }

  #save-btn {
    position: absolute;
    top: 10px;
    right: 10px;
    font-family: Consolas;
  }
</style>
