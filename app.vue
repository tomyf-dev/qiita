<script setup>
import Cropper from 'cropperjs'
import 'cropperjs/dist/cropper.css'
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3'
import { getSignedUrl } from '@aws-sdk/s3-request-presigner'

// S3の署名付きURLを発行する
const createSignedUrl = () => {
  const client = new S3Client({ region: 'ap-northeast-1', endpoint: 'http://127.0.0.1:9000', credentials: { accessKeyId: 'qiita', secretAccessKey: 'password' } })
  const command = new PutObjectCommand({ Bucket: 'local', Key: 'image.jpg' })
  return getSignedUrl(client, command, { expiresIn: 600 })
}

// state
const currentImage = ref('image.jpg')
const selectedFile = ref(null)
const cropper = ref(null)

// element
const input = ref()
const image = ref()

// ファイルが選択されたら、Cropperを有効にした画像を表示する
watch(selectedFile, () => {
  if (!selectedFile.value) return

  const fileReader = new FileReader()
  fileReader.onload = () => {
    image.value.src = fileReader.result
    cropper.value = new Cropper(image.value, {
      aspectRatio: 1,
      autoCropArea: 1,
      viewMode: 1,
      dragMode: 'move',
      guides: false,
      cropBoxMovable: false,
      cropBoxResizable: false,
      minCanvasHeight: 450,
    })
  }
  fileReader.readAsDataURL(selectedFile.value)
})

const select = async (event) => {
  selectedFile.value = event.target.files[0]
}

const apply = () => {
  const mimeType = 'image/jpeg'
  cropper.value.getCroppedCanvas().toBlob(async (blob) => {
    const signedUrl = await createSignedUrl()
    await useFetch(signedUrl, { method: 'PUT', body: blob, headers: { 'Content-Type': mimeType } })
    const url = new URL(signedUrl)
    currentImage.value = url.origin + url.pathname
  }, mimeType)

  selectedFile.value = null
}

const cancel = () => {
  selectedFile.value = null
}
</script>

<template>
  <div class="h-screen flex justify-center items-center">
    <div class="h-[36rem] w-[36rem] border-2 border-violet-200 rounded-lg">
      <div v-if="selectedFile" class="h-full flex flex-col">
        <div class="p-4 flex justify-between items-center bg-violet-100 rounded-t-md">
          <button class="font-bold text-lg hover:text-gray-900/75" type="button" @click="cancel">
            <svg class="w-6 h-6" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" />
            </svg>
          </button>
          <p class="font-bold text-lg">メディアを編集</p>
          <button class="px-4 py-2 rounded-md bg-gray-900/75 text-white hover:bg-gray-800/75" type="button" @click="apply">適用</button>
        </div>
        <hr>
        <div class="p-1 flex-1">
          <div class="h-full w-full">
            <img class="h-0 w-0" ref="image">
          </div>
        </div>
      </div>

      <div v-else class="h-full flex flex-col">
        <div class="p-4 flex justify-center items-center bg-violet-100 rounded-t-md">
          <p class="font-bold text-lg">プロフィールを編集</p>
        </div>
        <hr>
        <div class="p-4 flex-1">
          <div class="h-full flex justify-center items-center">
            <img class="h-52 w-52 rounded-full" :src="currentImage" />
            <button class="w-12 h-12 mt-40 -ml-12 p-2 bg-gray-900/75 rounded-full hover:bg-gray-800/75" type="button" @click="input.click">
              <svg class="w-8 h-8 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" d="M12 16.5V9.75m0 0l3 3m-3-3l-3 3M6.75 19.5a4.5 4.5 0 01-1.41-8.775 5.25 5.25 0 0110.233-2.33 3 3 0 013.758 3.848A3.752 3.752 0 0118 19.5H6.75z" />
              </svg>
            </button>
            <input class="hidden" ref="input" type="file" @change="select">
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
