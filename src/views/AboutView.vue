<script setup lang="ts">
import { ref, nextTick } from 'vue'
import Cropper from 'cropperjs'
import { ElDialog, ElButton } from 'element-plus'

const dialogVisible = ref(false)
const cropperRef = ref<Cropper | null>(null)
const isCircle = ref(false)
const editedImage = ref<string | null>(null)
const originalImage = ref('https://fengyuanchen.github.io/cropperjs/images/picture.jpg')

// State for transformations
const rotateDeg = ref(0)
const scaleX = ref(1)
const scaleY = ref(1)

const cropperWidth = ref(0)
const cropperHeight = ref(0)

const imageRef = ref<HTMLImageElement>()

function openDialog() {
  dialogVisible.value = true
  nextTick(() => {
    initializeCropper()
  })
}

function initializeCropper() {
  if (cropperRef.value) {
    return
  }

  const template = `
        <div class="cropper-container">
          <div class="cropper-wrap-box">
            <cropper-canvas background style="width:${cropperWidth.value}px;height:${cropperHeight.value}px">
              <cropper-image rotatable scalable translatable></cropper-image>
              <cropper-shade hidden></cropper-shade>
              <cropper-handle action="select" plain></cropper-handle>
              <cropper-selection id="cropperSelection" initial-coverage="0.5" movable resizable aspect-ratio="1">
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
          </div>
         <div class="cropper-preview-box">
          <cropper-viewer selection="#cropperSelection"></cropper-viewer>
         </div>
        </div>
      `

  cropperRef.value = new Cropper(imageRef.value as HTMLImageElement, {
    template: template,
  })
}

function handleUpload(event: Event) {
  const input = event.target as HTMLInputElement
  if (input.files && input.files[0]) {
    const reader = new FileReader()
    reader.onload = (e) => {
      originalImage.value = e.target?.result as string
      reset()
      nextTick(() => initializeCropper())
    }
    reader.readAsDataURL(input.files[0])
  }
}

function saveCrop() {
  if (!cropperRef.value) return
  const selection = cropperRef.value.getCropperSelection()
  if (selection) {
    selection
      .$toCanvas({
        width: 512,
        height: 512,
      })
      .then((canvas) => {
        editedImage.value = canvas.toDataURL('image/png')
        dialogVisible.value = false
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
  if (cropperRef.value) {
    const selection = cropperRef.value.getCropperSelection()
    selection?.$moveTo(0, 0)
    initializeCropper()
  }
}

function move(x: number, y: number) {
  cropperRef.value?.getCropperSelection()?.$move(x, y)
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
  cropperRef.value?.getCropperSelection()?.$zoom(ratio)
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
      <img ref="imageRef" :src="originalImage" />
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
        cropper-canvas{
          height: 200px !important;

        }
      }
      .cropper-preview-box {
        flex: 1;
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
