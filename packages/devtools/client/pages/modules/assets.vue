<!-- eslint-disable no-console -->
<script setup lang="ts">
import { onKeyDown, useMagicKeys } from '@vueuse/core'
import Fuse from 'fuse.js'
import type { AssetInfo } from '~/../src/types'

definePageMeta({
  icon: 'carbon-image-copy',
  title: 'Assets',
})

const search = ref('')

// TODO: add global settings for user
const upload = ref(false)
const showAll = ref(false)
const showByFolder = ref(false)
const view = ref<'list' | 'grid'>('grid')

const folders = ref<string[]>(['/'])
const foldersFlat = computed(() => folders.value.join(''))
const currentFolder = computed(() => folders.value[folders.value.length - 1])

const assets = computedAsync(async () => {
  return await rpc.getStaticAssets(foldersFlat.value, showAll.value)
})

const fuse = computed(() => new Fuse(assets.value || [], {
  keys: [
    'path',
  ],
}))

const filtered = computed(() => {
  const result = search.value
    ? fuse.value.search(search.value).map(i => i.item)
    : (assets.value || [])
  return result
})

const byFolders = computed(() => {
  const result: Record<string, AssetInfo[]> = {}
  for (const asset of filtered.value) {
    const folder = `${asset.path.split('/').slice(0, -1).join('/')}/`
    if (!result[folder])
      result[folder] = []
    result[folder].push(asset)
  }
  return Object.entries(result).sort(([a], [b]) => a.localeCompare(b))
})

const selected = ref<AssetInfo>()
const selectedDrawer = ref(false)
function selectAsset(asset: AssetInfo) {
  if (wsConnecting.value || wsError.value)
    return

  upload.value = false
  if (asset.type === 'folder')
    return folders.value.push(asset.path)

  selected.value = asset
  selectedDrawer.value = true
}

function unSelectAsset() {
  selected.value = undefined
  selectedDrawer.value = false
}

function changeFolder(folder: string) {
  console.log(folder)
  if (folder === currentFolder.value || wsConnecting.value || wsError.value || showAll.value)
    return
  folders.value = folders.value.slice(0, folders.value.indexOf(folder) + 1)
}

function goBackFolder() {
  if (currentFolder.value === '/' || wsConnecting.value || wsError.value || showAll.value)
    return
  folders.value.pop()
}

function toggleView() {
  view.value = view.value === 'list' ? 'grid' : 'list'
}

onKeyDown('Escape', () => {
  selected.value = undefined
  selectedDrawer.value = false
})

const { Enter, Space, Shift, Control, ArrowUp, ArrowDown, Backspace, BracketRight, BracketLeft } = useMagicKeys()

watchEffect(() => {
  if (ArrowUp.value) {
    if (selected.value) {
      const index = filtered.value.indexOf(selected.value)
      if (index > 0)
        selected.value = filtered.value[index - 1]
    }
    else {
      selected.value = filtered.value[0]
    }
  }
  if (ArrowDown.value) {
    if (selected.value) {
      const index = filtered.value.indexOf(selected.value)
      if (index < filtered.value.length - 1)
        selected.value = filtered.value[index + 1]
    }
    else {
      selected.value = filtered.value[0]
    }
  }
  if (Enter.value) {
    if (selected.value) {
      // TODO:
      if (selected.value.type === 'folder')
        return
      else
        return
    }
  }
  if (Space.value) {
    if (selected.value) {
      // dg
      selectedDrawer.value = !selectedDrawer.value
    }
  }
  if (Backspace.value)
    goBackFolder()
  // TODO: add history for folders
})

const router = useRouter()

watch(folders, () => {
  router.push({ query: { folder: foldersFlat.value } })
}, { deep: true })

onMounted(() => {
  if (router.currentRoute.value.query.folder) {
    const query = (router.currentRoute.value.query.folder as string)
    // TODO: use a better way to sanitize
    const xssPattern = /[<>]/g
    const encoded = encodeURIComponent(query).replace(xssPattern, '')
    const decoded = decodeURIComponent(encoded).replace(xssPattern, '')
    folders.value.push(...decoded.split('/').filter(Boolean).map(str => `${str}/`))
  }
})

const navbar = ref<HTMLElement>()

const copy = useCopy()
const showMenu = ref(false)

function refreshAssets() {
  upload.value = false
  selected.value = undefined
  showMenu.value = false

  // TODO: use a better way to refresh
  const items = folders.value
  folders.value = []
  folders.value = items
}

const contextOptions = reactive({ x: 0, y: 0 })

function contextMenu(e: MouseEvent) {
  e.preventDefault()
  contextOptions.x = e.x
  contextOptions.y = e.y

  showMenu.value = true

  const clickOutside = (e: MouseEvent) => {
    if (!navbar.value?.contains(e.target as Node)) {
      closeContext()
      document.removeEventListener('click', clickOutside)
    }
  }
  document.addEventListener('click', clickOutside)
}

function contextMenuAsset(asset: AssetInfo) {
  if (wsConnecting.value || wsError.value)
    return

  selected.value = asset
  showMenu.value = true
}

function closeContext() {
  showMenu.value = false
  selected.value = undefined
}

function createFolder() {
  const folder = prompt('Enter folder name')
  if (!folder)
    return
  rpc.writeStaticAssets([], currentFolder.value + folder)
    .then(refreshAssets)
    .catch(console.error)
}

function deleteItem() {
  if (!selected.value)
    return
  rpc.deleteStaticAsset(selected.value.filePath)
    .then(refreshAssets)
    .catch(console.error)
}
</script>

<template>
  <div relative h-full of-auto @contextmenu="contextMenu">
    <div ref="navbar" flex="~ col gap-2" border="b base" p4 navbar-glass flex-1 pb2>
      <div flex="~ gap4">
        <NTextInput
          v-model="search"
          placeholder="Search..."
          icon="carbon-search"
          flex-auto
          p="x5 y2"
          n="primary"
        />
        <div flex-none flex="~ gap4">
          <button
            title="Toggle view"
            @click="toggleView"
          >
            <NIcon v-if="view === 'grid'" icon="i-carbon-list" />
            <NIcon v-else icon="i-carbon-grid" />
          </button>
        </div>
      </div>
      <div items-center flex justify-between>
        <div flex items-center>
          <NIconButton icon="carbon-arrow-left" w-8 h-8 mr-2 @click="goBackFolder" />
          <div v-for="folder of folders" :key="folder" cursor-pointer underline hover-text-gray :class="{ 'text-green': folder === currentFolder }" @click="changeFolder(folder)">
            <span>
              {{ folder }}
            </span>
          </div>
        </div>
        <div>
          <span>show all assets: <NCheckbox v-model="showAll" /></span>
          <span v-if="view === 'grid' && showAll">show by folder: <NCheckbox v-model="showByFolder" /></span>
        </div>
        <div op50>
          <span v-if="search">{{ filtered.length }} matched · </span>
          <span>{{ assets?.length }} assets in total</span>
        </div>
      </div>
    </div>

    <template v-if="view === 'grid'">
      <template v-if="showByFolder && showAll">
        <NSectionBlock
          v-for="[folder, items] of byFolders"
          :key="folder"
          :text="folder"
          :description="`${items.length} items`"
          :open="items.length < 30"
          :padding="false"
        >
          <div px2 mt--4 grid="~ cols-minmax-8rem">
            <AssetGridItem v-for="a of items" :key="a.path" :asset="a" :folder="folder" @click="selectAsset(a)" @contextmenu="contextMenuAsset(a)" />
          </div>
        </NSectionBlock>
      </template>
      <div v-else p2 grid="~ cols-minmax-8rem">
        <AssetGridItem v-for="a of filtered" :key="a.path" :asset="a" @click="selectAsset(a)" @contextmenu="contextMenuAsset(a)" />
      </div>
    </template>
    <div v-else>
      <AssetListItem v-for="a of filtered" :key="a.path" :asset="a" :class="{ 'border border-base': a === selected }" @click="selectAsset(a)" @contextmenu="contextMenuAsset(a)" />
    </div>
    <DrawerRight
      :model-value="!!selected && selectedDrawer"
      w-120
      auto-close
      :navbar="navbar"
      @close="unSelectAsset"
    >
      <AssetDetails v-if="selected" :asset="selected" @deleted="refreshAssets()" />
    </DrawerRight>

    <NButton fixed right-5 bottom-5 @click="upload = true">
      <NIcon icon="carbon-cloud-upload" /> Upload Files
    </NButton>
    <DrawerRight
      :model-value="upload"
      auto-close
      w-120
      @close="upload = false"
    >
      <DropZone
        :folder="foldersFlat"
        @uploaded="refreshAssets"
      />
    </DrawerRight>

    <ContextMenu v-model:show="showMenu" :options="contextOptions" @close="closeContext">
      <template v-if="!selected">
        <NButton @click="createFolder">
          <NIcon icon="carbon-folder-add" />
          New Folder
        </NButton>
        <NButton>
          <NIcon icon="carbon-document-add" />
          New File
        </NButton>
        <NButton @click="upload = true">
          <NIcon icon="carbon-cloud-upload" />
          Upload Files
        </NButton>
      </template>

      <template v-if="selected">
        {{ selected.path }}
        <template v-if="selected.type === 'folder'">
          <NButton @click="selectAsset(selected!)">
            <NIcon icon="carbon-arrow-right" />
            Open
          </NButton>
        </template>
        <template v-else>
          <NButton>
            <NIcon icon="carbon-terminal" />
            Open in editor
          </NButton>
          <NButton>
            <NIcon icon="carbon-launch" />
            Open in browser
          </NButton>
        </template>
        <NButton @click="copy(selected!.publicPath)">
          <NIcon icon="carbon-copy" />
          Copy public path
        </NButton>
        <NButton v-if="selected.type !== 'folder'" @click="deleteItem">
          <NIcon icon="carbon-trash-can" />
          Delete Item
        </NButton>
      </template>

      <NButton @click="refreshAssets">
        <NIcon icon="carbon-reset" />
        Refresh
      </NButton>
    </ContextMenu>
  </div>
</template>
