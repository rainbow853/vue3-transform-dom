<script setup lang="ts">
import { TransferToWindow } from 'transfer-to-window';
import { ref, computed, onMounted, watchEffect } from 'vue';
import defaultTransferToWindow from './defaultTransferToWindow';
const emit = defineEmits(['transform']);

const props = defineProps({
  width: {
    type: Number,
    required: true,
  },
  height: {
    type: Number,
    required: true,
  },
  minWH: {
    type: Number,
    default: 1,
  },
  maxWH: {
    type: Number,
    default: Infinity,
  },
  limitInWindow: {
    type: Boolean,
    default: false,
  },
  limitSize: {
    type: Number,
    default: 100,
  },
  disableMove: {
    type: Boolean,
    default: false,
  },
  disableContextMove: {
    type: Boolean,
    default: false,
  },
  disableScale: {
    type: Boolean,
    default: false,
  },
  scaleStep: {
    type: Number,
    default: 0.1,
  },
})

const el = ref();
const transfer2window = ref<TransferToWindow>(defaultTransferToWindow);
const style = computed(() => {
  const { inw, inh, dx, dy, scale } = transfer2window.value;
  return {
    width: `${inw}px`,
    height: `${inh}px`,
    transform: `translate(${dx}px, ${dy}px) scale(${scale})`
  }
})

watchEffect(() => {
  const { outw, outh, scale, dx, dy } = transfer2window.value;
  el.value && emit('transform', [outw, outh, scale, dx, dy]);
})

onMounted(() => forceFresh());

/**
 * 强制刷新
 */
function forceFresh() {
  transfer2window.value = new TransferToWindow({
    inw: props.width,
    inh: props.height,
    outw: el.value.clientWidth,
    outh: el.value.clientHeight,
    minWH: props.minWH,
    maxWH: props.maxWH,
    limitInWindow: props.limitInWindow,
    limitSize: props.limitSize,
  })
}

export interface coor {
  x: number;
  y: number;
}

/**
 * 获取鼠标相对于el的坐标
 */
function getOutCoorPosition(e: MouseEvent): coor {
  let rect = el.value.getBoundingClientRect();
  return { x: e.clientX - rect.left, y: e.clientY - rect.top };
}

/**
 * 获取坐标
 */
function getPosition(e: MouseEvent, limitInWindow?: boolean): coor | null {
  const coor = getOutCoorPosition(e);
  const pos = transfer2window.value.transOutToIn([coor.x, coor.y]);
  return (limitInWindow && !transfer2window.value.inCoorIsIn(pos[0], pos[1])) ?
    null : { x: pos[0], y: pos[1] };
}

/**
 * 鼠标滚动事件，缩放$tranformDIV
 */
function wheel(e: WheelEvent) {
  if (props.disableScale) return;
  e.preventDefault()
  const coor = getOutCoorPosition(e);
  const ratio = 1 - props.scaleStep * Math.sign(e.deltaY);
  transfer2window.value.zoom(coor.x, coor.y, ratio);
}

let pos: coor;
/**
 * 鼠标按下事件，平移$tranformDIV
 */
function mousedown(e: MouseEvent) {
  if (props.disableMove) return;
  if (props.disableContextMove && e.button === 2) return;
  pos = getOutCoorPosition(e);
  document.addEventListener('mousemove', mousemove)
  document.addEventListener('mouseup', mouseup)
}

/**
 * 鼠标按下移动事件，平移$tranformDIV
 * 鼠标未按下移动事件，获取当前鼠标位置对应图像的灰度值
 */
function mousemove(e: MouseEvent) {
  let curPos = getOutCoorPosition(e);
  transfer2window.value.translate(curPos.x - pos.x, curPos.y - pos.y);
  pos = curPos;
}

/**
 * 鼠标抬起事件，平移$tranformDIV
 */
function mouseup() {
  document.removeEventListener('mousemove', mousemove)
  document.removeEventListener('mouseup', mouseup)
}

/**
 * 中心缩放
 */
function zoomByCenter(zoomOut: boolean) {
  const ratio = zoomOut ? 1 + props.scaleStep : 1 - props.scaleStep;
  transfer2window.value.zoom(
    transfer2window.value.outw / 2,
    transfer2window.value.outh / 2,
    ratio,
  )
}

defineExpose({
  transfer2window,
  getPosition,
  forceFresh,
  zoomByCenter,
})
</script>

<template>
  <div ref="el" style="width: 100%;height: 100%;overflow: hidden;position: relative;" @wheel="wheel"
    @mousedown="mousedown">
    <div style="position: relative;transform-origin: 0 0;" :style="style">
      <slot :transfer2window="transfer2window" :scale="transfer2window.scale"></slot>
    </div>
  </div>
</template>