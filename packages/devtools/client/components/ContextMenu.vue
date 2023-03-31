<script setup lang="ts">
interface ContextMenuOptions {
  x: number
  y: number
  minWidth?: number
}

const props = defineProps({
  show: {
    type: Boolean,
    required: true,
  },
  options: {
    type: Object as PropType<ContextMenuOptions>,
    required: true,
  },
})

const contextMenu = ref<HTMLElement>()

function openMenu() {
  const menu = contextMenu.value!
  const { x, y, minWidth } = props.options

  if (minWidth)
    menu.style.minWidth = `${minWidth}px`

  setPosition(x, y)
}

function setPosition(x: number, y: number) {
  const menu = contextMenu.value!
  const height = menu.offsetHeight
  const width = menu.offsetWidth
  const windowWidth = window.innerWidth
  const windowHeight = window.innerHeight

  if (x + width > windowWidth)
    x = windowWidth - width

  if (y + height > windowHeight)
    y = windowHeight - height

  menu.style.left = `${x}px`
  menu.style.top = `${y}px`
}

watch(() => props.options.x, () => {
  openMenu()
})
</script>

<template>
  <div v-show="show" ref="contextMenu" fixed z-11>
    <div grid p-3 rounded-lg bg-dark gap-2>
      <slot />
    </div>
  </div>
</template>
