<script lang="ts">
  import { onMount, onDestroy } from "svelte";

  let canvas: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D | null;

  let isDrawing: boolean = false;
  let lastX: number = 0;
  let lastY: number = 0;
  let color: string = "#FFFFFF";
  let minSize: number = 1;
  let maxSize: number = 100;
  let sizeValue: number = 5;
  let erase: boolean = false;
  let fillMode: boolean = false;
  
  let history: string[] = [];
  let undoEnabled: boolean = false;

  function toggleErase(): void {
    erase = !erase;
  }

  function toggleFill(): void {
    fillMode = !fillMode;
  }

  function chooseColor(event: Event): void {
    const target = event.target as HTMLInputElement;
    color = target.value;
  }

  function clear(): void {
    ctx?.clearRect(0, 0, canvas.width, canvas.height);
    saveCanvasToLocalStorage();
    localStorage.removeItem("canvasData");
    history = [];
    undoEnabled = false;
  }

  function startDrawing(event: MouseEvent): void {
    if (!fillMode) {
      saveHistory();
      isDrawing = true;
      [lastX, lastY] = [event.clientX - 5, event.clientY - 5];
    }
  }

  function draw(event: MouseEvent): void {
    if (!isDrawing || !ctx || fillMode) return;

    erase 
      ? ctx.globalCompositeOperation = 'destination-out' 
      : ctx.globalCompositeOperation = 'source-over';

    ctx.strokeStyle = color;
    ctx.lineWidth = sizeValue;
    ctx.lineJoin = "round";
    ctx.lineCap = "round";

    ctx.beginPath();
    ctx.moveTo(lastX, lastY);
    ctx.lineTo(event.clientX - 5, event.clientY - 5);
    ctx.stroke();
    [lastX, lastY] = [event.clientX - 5, event.clientY - 5];

    saveCanvasToLocalStorage();
  }

  function stopDrawing(): void {
    isDrawing = false;
  }

  function saveDrawing(): void {
    if (!canvas) return;
    const link = document.createElement("a");
    link.href = canvas.toDataURL("image/png");
    link.download = "drawing.png";
    link.click();
  }

  function resizeCanvas(): void {
    const imageData = ctx?.getImageData(0, 0, canvas.width, canvas.height);
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
    if (ctx && imageData) {
      ctx.putImageData(imageData, 0, 0);
    }
  }

  function saveCanvasToLocalStorage(): void {
    if (canvas) {
      const canvasData = canvas.toDataURL();
      localStorage.setItem("canvasData", canvasData);
    }
  }

  function loadCanvasFromLocalStorage(): void {
    const savedData = localStorage.getItem("canvasData");
    if (savedData && ctx) {
      const image: HTMLImageElement = new Image();
      image.src = savedData;
      image.onload = () => {
        ctx?.drawImage(image, 0, 0);
      };
    }
  }

  function saveHistory(): void {
    if (canvas && ctx) {
      const canvasData = canvas.toDataURL();
      history.push(canvasData);
      undoEnabled = true;
    }
  }

  function undo(): void {
    if (history.length === 0 || !ctx) return;

    const previousState = history.pop();
    undoEnabled = history.length > 0;

    if (previousState) {
      const image = new Image();
      image.src = previousState;
      image.onload = () => {
        ctx?.clearRect(0, 0, canvas.width, canvas.height);
        ctx?.drawImage(image, 0, 0);
        saveCanvasToLocalStorage();
      };
    }
  }

  function fill(event: MouseEvent): void {
    if (!ctx || !canvas || !fillMode) return;
    saveHistory();

    const fillColor = hexToRgb(color);
    const [startX, startY] = [event.clientX, event.clientY];
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    const data = imageData.data;
    const targetColor = getColorAtPixel(data, startX, startY);

    if (!colorsMatch(fillColor, targetColor)) {
      floodFill(startX, startY, fillColor, targetColor, imageData);
      ctx.putImageData(imageData, 0, 0);
      saveCanvasToLocalStorage();
    }
  }

  function floodFill(x: number, y: number, fillColor: number[], targetColor: number[], imageData: ImageData): void {
    const stack = [[x, y]];
    const width = imageData.width;
    const data = imageData.data;

    while (stack.length) {
      const [curX, curY] = stack.pop()!;
      const index = (curY * width + curX) * 4;

      if (colorsMatch(getColorAtPixel(data, curX, curY), targetColor)) {
        setColorAtPixel(data, curX, curY, fillColor);

        if (curX > 0) stack.push([curX - 1, curY]);
        if (curX < width - 1) stack.push([curX + 1, curY]);
        if (curY > 0) stack.push([curX, curY - 1]);
        if (curY < imageData.height - 1) stack.push([curX, curY + 1]);
      }
    }
  }

  function getColorAtPixel(data: Uint8ClampedArray, x: number, y: number): number[] {
    const index = (y * canvas.width + x) * 4;
    return [data[index], data[index + 1], data[index + 2], data[index + 3]];
  }

  function setColorAtPixel(data: Uint8ClampedArray, x: number, y: number, color: number[]): void {
    const index = (y * canvas.width + x) * 4;
    data[index] = color[0];
    data[index + 1] = color[1];
    data[index + 2] = color[2];
    data[index + 3] = 255;
  }

  function colorsMatch(a: number[], b: number[]): boolean {
    return a[0] === b[0] && a[1] === b[1] && a[2] === b[2] && a[3] === b[3];
  }

  function hexToRgb(hex: string): number[] {
    const bigint = parseInt(hex.slice(1), 16);
    return [(bigint >> 16) & 255, (bigint >> 8) & 255, bigint & 255, 255];
  }

  onMount(() => {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
    ctx = canvas.getContext("2d");
    
    loadCanvasFromLocalStorage();

    window.addEventListener("resize", resizeCanvas);
  });

  onDestroy(() => {
    window.removeEventListener("resize", resizeCanvas);
  });
</script>

<main>
  <canvas
    bind:this={canvas}
    on:mousedown={fillMode ? fill : startDrawing}
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
  <button on:click={toggleErase} id="erase-btn">
    Erase: {erase ? "on" : "off"}
  </button>
  <button on:click={toggleFill} id="fill-btn">
    Fill: {fillMode ? "on" : "off"}
  </button>
  <button on:click={undo} id="undo-btn" disabled={!undoEnabled}>
    Undo
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
  }

  #fill-btn {
    position: absolute;
    top: 200px;
    left: 10px;
    background-color: gray;
    width: 75px;
    height: 25px;
    color: white;
  }

  #undo-btn {
    position: absolute;
    top: 250px;
    left: 10px;
    background-color: gray;
    width: 75px;
    height: 25px;
    color: white;
  }

  #save-btn {
    position: absolute;
    top: 10px;
    right: 10px;
    font-family: Consolas;
  }
</style>
