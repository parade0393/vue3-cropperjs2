<script setup lang="ts">
import { ref } from 'vue'
import ImageCropper, { type CropperData } from '@/components/ImageCropper.vue'
import { ElButton } from 'element-plus'



const editedImage = ref<string | null>(null)
const originalImage = ref('https://fengyuanchen.github.io/cropperjs/images/picture.jpg')
const cropperRef = ref<InstanceType<typeof ImageCropper> | null>(null)

const handleSave = (data: CropperData) => {
  editedImage.value = data.base64 as string
}

const openCropper = () => {
  cropperRef.value?.openDialog()
}
</script>

<template>
  <div class="page-container">
    <div class="image-display">
      <h2>当前图片</h2>
      <img :src="editedImage || originalImage" alt="Current Image" class="displayed-image" />
      <el-button @click="openCropper" type="primary">编辑</el-button>
    </div>
    <ImageCropper ref="cropperRef" @on-save="handleSave" :img-src="originalImage" />
  </div>
</template>

<style scoped lang="scss">
.page-container {
  padding: 20px;
  text-align: center;
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
</style>
