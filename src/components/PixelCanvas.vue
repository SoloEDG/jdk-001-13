<script setup lang="ts">
import { computed, ref } from 'vue'
import { useEditorStore } from '../editor/useEditorStore'

const store = useEditorStore()

const gridRef = ref<HTMLElement | null>(null)

const displaySize = computed(() => {
  const size = store.canvasSize
  const cell = Math.floor(Math.min(512 / size, 24))
  return { size, cell, px: size * cell }
})

/** Maps a pointer event to a cell; optionally clamps out-of-bounds positions
 *  to the nearest edge cell so drags leaving the canvas keep stroking. */
function cellFromEvent(e: PointerEvent, clamp: boolean): { x: number; y: number } | null {
  const el = gridRef.value
  if (!el) return null
  const rect = el.getBoundingClientRect()
  const { size, cell } = displaySize.value
  let x = Math.floor((e.clientX - rect.left) / cell)
  let y = Math.floor((e.clientY - rect.top) / cell)
  if (clamp) {
    x = Math.max(0, Math.min(size - 1, x))
    y = Math.max(0, Math.min(size - 1, y))
    return { x, y }
  }
  if (x < 0 || y < 0 || x >= size || y >= size) return null
  return { x, y }
}

function onPointerDown(e: PointerEvent) {
  if (e.button !== 0) return
  if (store.isPlaying || store.previewFrameIndex !== null) return
  const cell = cellFromEvent(e, false)
  if (!cell) return
  e.preventDefault()
  gridRef.value?.setPointerCapture(e.pointerId)
  store.onCanvasPointerDown(cell.x, cell.y)
}

function onPointerMove(e: PointerEvent) {
  if (!store.isPainting) return
  const cell = cellFromEvent(e, true)
  if (!cell) return
  store.onCanvasPointerMove(cell.x, cell.y)
}

function onPointerUp(e: PointerEvent) {
  store.onCanvasPointerUp()
  if (gridRef.value?.hasPointerCapture(e.pointerId)) {
    gridRef.value.releasePointerCapture(e.pointerId)
  }
}

function cellStyle(index: number) {
  const color = store.compositePixels[index]
  const size = displaySize.value.size
  const x = index % size
  const y = Math.floor(index / size)
  const checker = (x + y) % 2 === 0 ? '#2a2d35' : '#24272e'
  if (color === null || color === undefined) {
    return { backgroundColor: checker }
  }
  return { backgroundColor: color }
}
</script>

<template>
  <main class="canvas-area">
    <div class="canvas-meta">
      <span>{{ store.canvasSize }}×{{ store.canvasSize }}</span>
      <span v-if="store.isPlaying" class="badge playing">动画预览中</span>
      <span v-else-if="store.previewFrameIndex !== null" class="badge">预览帧</span>
    </div>
    <div
      ref="gridRef"
      class="pixel-grid"
      :style="{
        gridTemplateColumns: `repeat(${displaySize.size}, ${displaySize.cell}px)`,
        width: `${displaySize.px}px`,
        height: `${displaySize.px}px`,
      }"
      @pointerdown="onPointerDown"
      @pointermove="onPointerMove"
      @pointerup="onPointerUp"
      @pointercancel="onPointerUp"
    >
      <button
        v-for="(_, index) in store.compositePixels"
        :key="index"
        type="button"
        class="pixel-cell"
        :style="cellStyle(index)"
        :aria-label="`像素 ${index}`"
        tabindex="-1"
      />
    </div>
  </main>
</template>
