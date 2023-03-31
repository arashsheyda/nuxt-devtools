<!-- eslint-disable no-console -->
<script setup lang="ts">
const props = defineProps({
  folder: {
    type: String,
    required: true,
  },
})
const emit = defineEmits(['uploaded'])
const dropZone = ref<HTMLElement>()
const files = ref<File[]>([])
const isDragging = ref(false)

function activeDropZone(e: DragEvent) {
  e.preventDefault()
  isDragging.value = true
}

function deactiveDropZone(e: DragEvent) {
  e.preventDefault()
  isDragging.value = false
}

const showNotification = useNotification()

// TODO: add option to settings
const uploadTypes: string[] = ['image', 'video']

function onDrop(e: DragEvent) {
  e.preventDefault()
  isDragging.value = false
  const inputFiles = Array.from(e.dataTransfer!.files)
  if (inputFiles.length) {
    const newFiles: File[] = []
    const existingFileNames: string[] = files.value.map(file => file.name)
    for (const file of inputFiles) {
      if (existingFileNames.includes(file.name)) {
        let i = 1
        const [name, ext] = file.name.split('.')
        while (existingFileNames.includes(`${name} (${i}).${ext}`))
          i++

        const newFilename = `${name}-${i}.${ext}`
        const blob = new Blob([file], { type: file.type })
        // Object.defineProperty(blob, 'name', { value: newFilename, writable: false })
        const newFile = new File([blob], newFilename, { lastModified: Date.now() })
        newFiles.push(newFile)
        existingFileNames.push(newFilename)
      }
      else {
        if (file.type === '')
          showNotification('Folders and hidden files are not supported yet', 'carbon:face-dissatisfied')

        else if (uploadTypes.some(type => file.type.includes(type)))
          newFiles.push(file)

        else
          showNotification(`"${file.type}" file type is not allowed`, 'carbon:face-dizzy')
      }
    }
    files.value = [...files.value, ...newFiles]

    // inputFiles.forEach((file: File) => {
    //   if (files.value.find(f => f.name === file.name)) {
    //     let i = 1
    //     const [name, ext] = file.name.split('.')
    //     while (files.value.find(f => f.name === `${name} (${i}).${ext}`))
    //       i++
    //     const newFilename = `${name}-${i}.${ext}`
    //     const blob = new Blob([file], { type: file.type })
    //     Object.defineProperty(blob, 'name', { value: newFilename, writable: false })
    //     file = blob
    //   }

    //   if (file.type === '') {
    //     // TODO: support folders & . files
    //     showNotification('not supported yet', 'carbon:face-dissatisfied')
    //   }

    //   else {
    //     for (let index = 0; index < uploadTypes.length; index++) {
    //       if (file.type.includes(uploadTypes[index]))
    //         return files.value.push(file)
    //       else
    //         showNotification(`${file.type} File type not allowed`, 'carbon:face-dizzy')
    //     }
    //   }
    // })
  }
}

async function uploadFiles() {
  if (wsConnecting.value || wsError.value)
    return

  const readyFiles = []

  for (const file of files.value) {
    const reader = new FileReader()
    reader.readAsDataURL(file)
    const result = await new Promise((resolve) => {
      reader.onload = () => resolve(reader.result as string)
    }) as string
    const data = result.split(';base64,').pop() as string
    readyFiles.push({
      name: file.name,
      data,
    })
  }

  await rpc.writeStaticAssets([...readyFiles], props.folder).then(() => {
    emit('uploaded')
    clearFiles()
    showNotification('Files uploaded successfully!', 'carbon:face-cool')
  }).catch((error) => {
    console.error(error)
    showNotification('Upload failed!', 'carbon:face-dizzy')
  })
}

function clearFiles() {
  files.value = []
}

function removeFile(index: number) {
  files.value.splice(index, 1)
}

function blobIt(file: any) {
  return URL.createObjectURL(file)
}

function changeName(file: any, newName: string) {
  const blob = new Blob([file], { type: file.type })
  Object.defineProperty(blob, 'name', { value: newName, writable: false })
  const newFile = new File([blob], newName, { lastModified: Date.now() })
  files.value.splice(files.value.indexOf(file), 1, newFile)
  return newFile
}
</script>

<template>
  <div
    ref="dropZone"
    w-full z--10 h-full
    @dragover="activeDropZone"
    @dragleave="deactiveDropZone"
    @drop="onDrop"
  >
    <div h-full items-center justify-center relative flex>
      <div fixed bg-dark top-0 left-0 right-0 bg-opacity-50 z-10 p-4 backdrop-blur>
        {{ folder }}
      </div>
      <div>
        <span v-if="!files.length">
          Drag and drop files here...
        </span>
      </div>

      <div v-if="files.length" top-0 right-0 left-0 grid p-4 overflow-auto absolute grid-cols-3 gap-5 py-20>
        <div v-for="file, index of files" :key="file.name" relative>
          <button absolute bg-dark rounded-lg flex top--4 p-1 right--4>
            <NIcon icon="carbon-close" @click="removeFile(index)" />
          </button>
          <img w-full object-cover :src="blobIt(file)">
          <input w-full type="text" :value="file.name" @change="changeName(file, ($event.target as HTMLInputElement).value)">
        </div>
      </div>

      <div fixed right-0 left-0 flex bottom-0>
        <button v-if="files.length" w-full h-full p-4 bg-gray-5 @click="clearFiles">
          <NIcon icon="carbon-close" /> Clear
        </button>
        <button :disabled="!files.length" w-full h-full p-4 :class="files.length ? 'bg-green-6 hover-bg-green-5' : 'bg-gray-5'" @click="uploadFiles">
          <NIcon icon="carbon-cloud-upload" /> Upload Files
        </button>
      </div>
    </div>
  </div>
</template>
