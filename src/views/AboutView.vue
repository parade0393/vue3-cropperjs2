<script setup lang="ts">
import { ref, nextTick } from 'vue'
import Cropper from 'cropperjs'

import { ElDialog, ElButton } from 'element-plus'

const dialogVisible = ref(false)
const cropperInstance = ref<Cropper | null>(null)
const isCircle = ref(false)
const editedImage = ref<string | null>(null)
const originalImage = ref('https://fengyuanchen.github.io/cropperjs/images/picture.jpg')

// State for transformations
const rotateDeg = ref(0)
const scaleX = ref(1)
const scaleY = ref(1)
const infos = ref()
const cropperContainerRef = ref<HTMLDivElement | null>(null)


function formatBytes(bytes: number): string {
  if (bytes < 0) {
    return "Invalid value"; // 可选
  }
  const units: string[] = ["B", "KB", "MB", "GB", "TB", "PB", "EB"];
  const i: number = Math.floor(Math.log(bytes) / Math.log(1024));
  return `${(bytes / Math.pow(1024, i)).toFixed(2)} ${units[i]}`;
}

function openDialog() {
  dialogVisible.value = true
  nextTick(() => {
    initializeCropper()
  })
}

function onCropperSelectionChange() {

  saveCrop()
}

function initializeCropper() {
  if (cropperInstance.value) {
    return
  }

  const template = `
        <cropper-canvas background>
              <cropper-image rotatable scalable translatable></cropper-image>
              <cropper-shade ></cropper-shade>
              <cropper-handle action="select" plain></cropper-handle>
              <cropper-selection initial-aspect-ratio="1" initial-coverage="0.5" id="cropperSelection"  movable resizable >
                <cropper-grid role="grid" bordered covered></cropper-grid>
                <cropper-crosshair centered></cropper-crosshair>
                <cropper-handle action="move"></cropper-handle>
                <cropper-handle action="n-resize"></cropper-handle>
                <cropper-handle action="e-resize"></cropper-handle>
                <cropper-handle action="s-resize"></cropper-handle>
                <cropper-handle action="w-resize"></cropper-handle>
                <cropper-handle action="ne-resize"></cropper-handle>
                <cropper-handle action="nw-resize"></cropper-handle>
                <cropper-handle action="se-resize"></cropper-handle>
                <cropper-handle action="sw-resize"></cropper-handle>
              </cropper-selection>
            </cropper-canvas>
      `
  const image = new Image()
  image.src = originalImage.value
  image.crossOrigin = 'anonymous' // 解决跨域问题
  cropperInstance.value = new Cropper(image, {
    template: template,
    container: cropperContainerRef.value as HTMLDivElement,
  })

  //否则刚开始的时候不会显示预览
  setTimeout(() => {
    cropperInstance.value
      ?.getCropperSelection()
      ?.addEventListener('change', onCropperSelectionChange)
    cropperInstance.value?.getCropperSelection()?.$move(10, 0)
  }, 0)
}

function handleUpload(event: Event) {
  const input = event.target as HTMLInputElement
  if (input.files && input.files[0]) {
    const reader = new FileReader()
    reader.onload = (e) => {
      const upoadImg = e.target?.result as string
      cropperInstance.value?.getCropperImage()?.setAttribute('src', upoadImg)
    }
    reader.readAsDataURL(input.files[0])
  }
}

function saveCrop() {
  if (!cropperInstance.value) return
  const selection = cropperInstance.value.getCropperSelection()
  if (selection) {
    selection
      .$toCanvas({
        beforeDraw(ctx, canvas) {
          const x = canvas.width / 2
          const y = canvas.height / 2
          const radius = Math.min(canvas.width, canvas.height) / 2
          // 绘制圆形裁剪区域
          ctx.beginPath()
          ctx.arc(x, y, radius, 0, Math.PI * 2, true)
          ctx.closePath()
          ctx.clip()
        },
      })
      .then((canvas) => {
        canvas.toBlob((blob) => {
          if (!blob) return
          const fileReader: FileReader = new FileReader()
          fileReader.readAsDataURL(blob)
          fileReader.onloadend = (e) => {
            if (!e.target?.result || !blob) return
            infos.value = {
              base64: e.target.result,
              blob,
              info: { size: blob.size, width: selection.width, height: selection.height },
            }
            //   // editedImage.value = canvas.toDataURL('image/png')
            //   editedImage.value = e.target.result as string
            // dialogVisible.value = false
          }
        })
      })
  }
}

function toggleShape() {
  isCircle.value = !isCircle.value
  // Re-initialize cropper to apply shape change
  initializeCropper()
}

function reset() {
  rotateDeg.value = 0
  scaleX.value = 1
  scaleY.value = 1
  if (cropperInstance.value) {
    const selection = cropperInstance.value.getCropperSelection()
    selection?.$moveTo(0, 0)
    initializeCropper()
  }
}

function move(x: number, y: number) {
  cropperInstance.value?.getCropperSelection()?.$move(x, y)
}

function flip(horizontal: boolean) {
  if (horizontal) {
    scaleX.value *= -1
  } else {
    scaleY.value *= -1
  }
  initializeCropper()
}

function rotate(degree: number) {
  rotateDeg.value += degree
  initializeCropper()
}

function zoom(ratio: number) {
  cropperInstance.value?.getCropperSelection()?.$zoom(ratio)
}
</script>

<template>
  <div class="page-container">
    <div class="image-display">
      <h2>当前图片</h2>
      <img :src="editedImage || originalImage" alt="Current Image" class="displayed-image" />
      <el-button @click="openDialog" type="primary">编辑</el-button>
    </div>

    <el-dialog v-model="dialogVisible" modal-class="cropper-modal" title="编辑图片" width="40%">
      <div class="cropper-container">
        <div ref="cropperContainerRef" class="cropper-wrap-box"></div>
        <div class="cropper-preview-box">
          <el-image :src="infos?.base64" />
          <div>
            <p>原始大小：{{ infos?.info.size||0 }}</p>
            <p>图片大小: {{ formatBytes(infos?.info.size || 0) }}</p>
            <p>宽度: {{ infos?.info.width || 0 }} px</p>
            <p>高度: {{ infos?.info.height || 0 }} px</p>
          </div>
        </div>
      </div>
      <div class="toolbar">
        <input
          type="file"
          @change="handleUpload"
          accept="image/*"
          style="display: none"
          id="upload-input"
        />
        <label for="upload-input" class="toolbar-button">上传</label>
        <button @click="toggleShape" class="toolbar-button">
          {{ isCircle ? '矩形' : '圆形' }}裁剪
        </button>
        <button @click="reset" class="toolbar-button">重置</button>
        <button @click="move(-10, 0)" class="toolbar-button">左移</button>
        <button @click="move(10, 0)" class="toolbar-button">右移</button>
        <button @click="move(0, -10)" class="toolbar-button">上移</button>
        <button @click="move(0, 10)" class="toolbar-button">下移</button>
        <button @click="flip(true)" class="toolbar-button">水平翻转</button>
        <button @click="flip(false)" class="toolbar-button">垂直翻转</button>
        <button @click="rotate(-45)" class="toolbar-button">逆时针旋转</button>
        <button @click="rotate(45)" class="toolbar-button">顺时针旋转</button>
        <button @click="zoom(0.1)" class="toolbar-button">放大</button>
        <button @click="zoom(-0.1)" class="toolbar-button">缩小</button>
      </div>
      <template #footer>
        <el-button @click="dialogVisible = false">取消</el-button>
        <el-button @click="saveCrop" type="primary">保存</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<style scoped lang="scss">
.page-container {
  padding: 20px;
  text-align: center;
  :deep(.cropper-modal) {
    .cropper-container {
      display: flex;
      flex-direction: row;
      height: 200px;
      gap: 20px;
      .cropper-wrap-box {
        flex: 1;
        position: relative;
        cropper-canvas {
          height: 200px !important;
        }
        cropper-shade {
          border-radius: 50%;
          border: 1px solid #ccc;
        }
        cropper-handle {
          background-color: unset;
        }
      }
      .cropper-preview-box {
        flex: 1;
        cropper-viewer {
          border-radius: 50%;
        }
      }
    }
  }
}

.image-display {
  margin-bottom: 20px;
}

.displayed-image {
  max-width: 100%;
  max-height: 300px;
  border: 1px solid #ccc;
  margin-bottom: 10px;
}

.toolbar {
  padding-top: 10px;
  padding-bottom: 10px;
  border-bottom: 1px solid #ccc;
  margin-bottom: 10px;
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
}

.toolbar-button {
  padding: 8px 12px;
  border: 1px solid #ccc;
  background-color: #f7f7f7;
  cursor: pointer;
  border-radius: 4px;
}

.toolbar-button:hover {
  background-color: #eee;
}

.dialog-actions {
  display: flex;
  justify-content: flex-end;
  width: 100%;
  margin-top: auto;
}
</style>
