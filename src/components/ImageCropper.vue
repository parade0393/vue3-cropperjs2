<script setup lang="ts">
import { ref, nextTick, watch, defineExpose,onBeforeUnmount } from 'vue'
import Cropper from 'cropperjs'

import { ElDialog, ElButton, ElImage } from 'element-plus'

const CROPPER_TEMPLATE = `
        <cropper-canvas background>
              <cropper-image rotatable scalable translatable></cropper-image>
              <cropper-shade></cropper-shade>
              <cropper-handle action="select" plain></cropper-handle>
              <cropper-selection initial-aspect-ratio="1" aspect-ratio="1" initial-coverage="0.5" id="cropperSelection"  movable resizable >
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


export interface CropperData {
  base64: string;
  blob: Blob;
  info: {
    size: number;
    width: number;
    height: number;
  };
}

interface Props{
  title?: string,
  outerWidth?: string,
  isCircle?: boolean,
  innerHeight?: string,
  imgSrc?: string,
  lockCircle?: boolean,
  template?:string
}

const props = withDefaults(defineProps<Props>(), {
  title: '图片编辑',
  outerWidth: '40%',
  isCircle: true,
  innerHeight: '200px',
  imgSrc: '',
  lockCircle: false,
  template:CROPPER_TEMPLATE
})

const emits = defineEmits<{ (e: 'on-save', data: CropperData): void }>()

const componentIsCircle = ref(props.isCircle)

const dialogVisible = ref(false)
const cropperInstance = ref<Cropper | null>(null)
const originalImage = ref(props.imgSrc)


const infos = ref<CropperData | undefined>()
const cropperContainerRef = ref<HTMLDivElement | null>(null)

function formatBytes(bytes: number): string {
  if (bytes < 0) {
    return 'Invalid value' // 可选
  }
  const units: string[] = ['B', 'KB', 'MB', 'GB', 'TB', 'PB', 'EB']
  const i: number = Math.floor(Math.log(bytes) / Math.log(1024))
  return `${(bytes / Math.pow(1024, i)).toFixed(2)} ${units[i]}`
}

function openDialog() {
  dialogVisible.value = true
  nextTick(() => {
    initializeCropper()
  })
}

function closeDialog() {
  dialogVisible.value = false
  if(cropperInstance.value){
    cropperInstance.value.getCropperSelection()?.removeEventListener('change', onCropperSelectionChange)
    cropperInstance.value = null
  }
}

function onCropperSelectionChange() {
  console.log('裁剪区域发生变化')
  saveCrop(false)
}

function initializeCropper() {
  if (cropperInstance.value) {
    return
  }
  const image = new Image()
  image.src = originalImage.value
  image.crossOrigin = 'anonymous' // 解决跨域问题
  cropperInstance.value = new Cropper(image, {
    template: props.template,
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
      setTimeout(() => {
        saveCrop(false)
      }, 500)
    }
    reader.readAsDataURL(input.files[0])
  }
}

function saveCrop(isSave: boolean) {
  if (!cropperInstance.value) return
  const selection = cropperInstance.value.getCropperSelection()
  if (selection) {
    selection
      .$toCanvas({
        beforeDraw(ctx, canvas) {
          if(!componentIsCircle.value) return
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
            const result = {
              base64: e.target.result as string,
              blob,
              info: { size: blob.size, width: selection.width, height: selection.height },
            }
            infos.value = result
            if(isSave){
              emits('on-save', result)
              closeDialog()
            }
          }
        })
      })
  }
}

function toggleShape() {
  componentIsCircle.value = !componentIsCircle.value
  saveCrop(false)
}

function reset() {
  if (cropperInstance.value) {
    const selection = cropperInstance.value.getCropperSelection()
    selection?.$reset()
  }
}

function move(x: number, y: number) {
  cropperInstance.value?.getCropperImage()?.$move(x, y)
}


function rotate(degree: number) {
  cropperInstance.value?.getCropperImage()?.$rotate(degree)
}

function zoom(ratio: number) {
  cropperInstance.value?.getCropperImage()?.$scale(ratio)
}


watch(() => props.isCircle, (val) => {
  componentIsCircle.value = val
})

watch(() => props.imgSrc, (val) => {
  originalImage.value = val
})

onBeforeUnmount(() => {
  if (cropperInstance.value) {
    cropperInstance.value.getCropperSelection()?.removeEventListener('change', onCropperSelectionChange)
    cropperInstance.value = null
  }
})


defineExpose({ openDialog, closeDialog });
</script>

<template>
    <el-dialog v-model="dialogVisible" modal-class="cropper-modal" :title="title" :width="outerWidth">
      <div class="cropper-container">
        <div ref="cropperContainerRef" class="cropper-wrap-box"></div>
        <div class="cropper-preview-box">
          <el-image :src="infos?.base64" v-show="infos?.base64"/>
          <div class="perview-info" v-show="infos?.base64">
            <p>图像大小：{{ infos?.info.width || 0 }} x {{ infos?.info.height || 0 }}像素</p>
            <p>文件大小: {{ formatBytes(infos?.info.size || 0) }}({{ infos?.info.size || 0 }}字节)</p>
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
        <button v-if="!lockCircle" @click="toggleShape" class="toolbar-button">
          {{ componentIsCircle ? '矩形' : '圆形' }}裁剪
        </button>
        <button @click="reset" class="toolbar-button">重置</button>
        <button @click="move(-10, 0)" class="toolbar-button">左移</button>
        <button @click="move(10, 0)" class="toolbar-button">右移</button>
        <button @click="move(0, -10)" class="toolbar-button">上移</button>
        <button @click="move(0, 10)" class="toolbar-button">下移</button>
        <button @click="rotate(-45)" class="toolbar-button">逆时针旋转</button>
        <button @click="rotate(45)" class="toolbar-button">顺时针旋转</button>
        <button @click="zoom(1.1)" class="toolbar-button">放大</button>
        <button @click="zoom(0.9)" class="toolbar-button">缩小</button>
      </div>
      <template #footer>
        <el-button @click="closeDialog">取消</el-button>
        <el-button @click="saveCrop(true)" type="primary">保存</el-button>
      </template>
    </el-dialog>
</template>

<style scoped lang="scss">
  .cropper-modal {
    .cropper-container {
      display: flex;
      flex-direction: row;
      height: v-bind(innerHeight);
      gap: 20px;
      .cropper-wrap-box {
        flex: 1;
        position: relative;
        :deep(cropper-canvas) {
          height: v-bind(innerHeight) !important;
        }
        :deep(cropper-shade) {
          border-radius: v-bind('componentIsCircle ? "50%" : "unset"');
          border: 1px solid #ccc;
        }
        :deep(cropper-handle) {
          background-color: unset;
        }
      }
      .cropper-preview-box {
        flex: 1;
        display: flex;
        flex-direction: column;
        .el-image{
          align-self: center;
        }
        :deep(cropper-viewer) {
          border-radius: v-bind('componentIsCircle ? "50%" : "unset"');
        }
        .perview-info{
          flex: 1;
          display: flex;
          flex-direction: column;
          justify-content: flex-end;
        }
      }
    }
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
