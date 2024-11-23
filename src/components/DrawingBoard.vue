<template>
  <div class="drawing-board">
    <div class="controls mb-4 flex gap-4 items-center">
      <input
        type="color"
        v-model="strokeColor"
        class="w-12 h-8 cursor-pointer"
      />
      <select v-model="strokeWidth" class="border rounded px-2 py-1">
        <option value="2">细</option>
        <option value="5">中</option>
        <option value="10">粗</option>
      </select>
      <button
        @click="undo"
        :disabled="!canUndo"
        class="p-2 text-blue-500 hover:bg-blue-50 rounded-full disabled:opacity-50 disabled:cursor-not-allowed"
        title="撤销 (Ctrl+Z)"
      >
        <ArrowUturnLeftIcon class="w-5 h-5" />
      </button>
      <button
        @click="redo"
        :disabled="!canRedo"
        class="p-2 text-blue-500 hover:bg-blue-50 rounded-full disabled:opacity-50 disabled:cursor-not-allowed"
        title="重做 (Ctrl+Y)"
      >
        <ArrowUturnRightIcon class="w-5 h-5" />
      </button>
      <button
        @click="clearCanvas"
        class="p-2 text-red-500 hover:bg-red-50 rounded-full"
        title="清除画布"
      >
        <TrashIcon class="w-5 h-5" />
      </button>
      <button
        @click="exportImage"
        class="p-2 text-green-500 hover:bg-green-50 rounded-full"
        title="导出图片"
      >
        <ArrowDownTrayIcon class="w-5 h-5" />
      </button>
    </div>
    <canvas
      ref="canvas"
      @mousedown="startDrawing"
      @mousemove="draw"
      @mouseup="stopDrawing"
      @mouseleave="stopDrawing"
      @touchstart="handleTouchStart"
      @touchmove="handleTouchMove"
      @touchend="stopDrawing"
      class="border rounded-lg shadow-sm bg-white touch-none"
      width="800"
      height="600"
    ></canvas>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed } from 'vue';
import {
  ArrowUturnLeftIcon,
  ArrowUturnRightIcon,
  TrashIcon,
  ArrowDownTrayIcon,
} from '@heroicons/vue/24/outline';

const canvas = ref<HTMLCanvasElement | null>(null);
const isDrawing = ref(false);
const strokeColor = ref('#000000');
const strokeWidth = ref('5');
const undoStack = ref<ImageData[]>([]);
const redoStack = ref<ImageData[]>([]);
let ctx: CanvasRenderingContext2D | null = null;
let lastX = 0;
let lastY = 0;

const canUndo = computed(() => undoStack.value.length > 1);
const canRedo = computed(() => redoStack.value.length > 0);

onMounted(() => {
  if (canvas.value) {
    ctx = canvas.value.getContext('2d');
    if (ctx) {
      ctx.lineCap = 'round';
      ctx.lineJoin = 'round';
      saveState();
    }
  }
  window.addEventListener('keydown', handleKeyDown);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown);
});

function handleKeyDown(e: KeyboardEvent) {
  if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === 'z') {
    e.preventDefault();
    if (e.shiftKey) {
      redo();
    } else {
      undo();
    }
  } else if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === 'y') {
    e.preventDefault();
    redo();
  }
}

function saveState() {
  if (!ctx || !canvas.value) return;
  undoStack.value.push(
    ctx.getImageData(0, 0, canvas.value.width, canvas.value.height)
  );
  redoStack.value = [];
  if (undoStack.value.length > 50) {
    undoStack.value.shift();
  }
}

function undo() {
  if (!ctx || !canvas.value || !canUndo.value) return;
  const currentState = ctx.getImageData(
    0,
    0,
    canvas.value.width,
    canvas.value.height
  );
  redoStack.value.push(currentState);
  undoStack.value.pop();
  const lastState = undoStack.value[undoStack.value.length - 1];
  ctx.putImageData(lastState, 0, 0);
}

function redo() {
  if (!ctx || !canvas.value || !canRedo.value) return;
  const redoState = redoStack.value.pop()!;
  const currentState = ctx.getImageData(
    0,
    0,
    canvas.value.width,
    canvas.value.height
  );
  undoStack.value.push(currentState);
  ctx.putImageData(redoState, 0, 0);
}

function startDrawing(e: MouseEvent) {
  isDrawing.value = true;
  [lastX, lastY] = [e.offsetX, e.offsetY];
}

function draw(e: MouseEvent) {
  if (!isDrawing.value || !ctx) return;

  ctx.beginPath();
  ctx.strokeStyle = strokeColor.value;
  ctx.lineWidth = Number(strokeWidth.value);
  ctx.moveTo(lastX, lastY);
  ctx.lineTo(e.offsetX, e.offsetY);
  ctx.stroke();
  [lastX, lastY] = [e.offsetX, e.offsetY];
}

function handleTouchStart(e: TouchEvent) {
  e.preventDefault();
  if (!canvas.value) return;
  const rect = canvas.value.getBoundingClientRect();
  const touch = e.touches[0];
  lastX = touch.clientX - rect.left;
  lastY = touch.clientY - rect.top;
  isDrawing.value = true;
}

function handleTouchMove(e: TouchEvent) {
  e.preventDefault();
  if (!isDrawing.value || !ctx || !canvas.value) return;
  const rect = canvas.value.getBoundingClientRect();
  const touch = e.touches[0];
  const x = touch.clientX - rect.left;
  const y = touch.clientY - rect.top;

  ctx.beginPath();
  ctx.strokeStyle = strokeColor.value;
  ctx.lineWidth = Number(strokeWidth.value);
  ctx.moveTo(lastX, lastY);
  ctx.lineTo(x, y);
  ctx.stroke();
  [lastX, lastY] = [x, y];
}

function stopDrawing() {
  if (isDrawing.value) {
    saveState();
    isDrawing.value = false;
  }
}

function clearCanvas() {
  if (!ctx || !canvas.value) return;
  ctx.clearRect(0, 0, canvas.value.width, canvas.value.height);
  saveState();
}

function exportImage() {
  if (!canvas.value) return;

  // 创建一个临时链接
  const link = document.createElement('a');
  link.download = `drawing-${new Date()
    .toISOString()
    .slice(0, 19)
    .replace(/[:-]/g, '')}.png`;
  link.href = canvas.value.toDataURL('image/png');

  // 触发下载
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}
</script>
