<script setup lang="ts">
import type { AssetInfo } from '~/../src/types'

const props = defineProps<{
  asset: AssetInfo
}>()

function pathIncludesAny(path: string, values: string[]) {
  const regex = new RegExp(`\\b(${values.join('|')})\\b`)
  return regex.test(path)
}

const typeToIcon: any = {
  folder: {
    'assets image img': 'vscode-icons:folder-type-images',
    'test': 'vscode-icons:folder-type-test',
    'video mp4': 'vscode-icons:folder-type-video',
    'audio mp3 music': 'vscode-icons:folder-type-audio',
    'code src': 'vscode-icons:folder-type-src',
    'locale i18n l18n translation': 'vscode-icons:folder-type-locale',
    'nuxt': 'vscode-icons:folder-type-nuxt',
    'other': 'vscode-icons:default-folder',
  },
  image: 'i-carbon-image',
  video: 'i-carbon-video',
  audio: 'i-carbon-volume-up',
  font: 'i-carbon-text-small-caps',
  text: 'i-carbon-document',
  json: 'i-carbon-json',
  other: 'i-carbon-document-blank',
}

const icon: ComputedRef<string> = computed(() => {
  const { path, type } = props.asset
  if (type in typeToIcon) {
    const typeIcon = typeToIcon[type]
    if (typeof typeIcon === 'object') {
      for (const [folders, icon] of Object.entries(typeIcon)) {
        const folderNames = folders.split(' ')
        if (pathIncludesAny(path, folderNames))
          return icon
      }
      return typeIcon.other
    }
    else {
      return typeIcon
    }
  }
  else {
    return typeToIcon.other
  }
})
</script>

<template>
  <NIcon h-20 mt-4 w-20 :icon="icon" />
</template>
