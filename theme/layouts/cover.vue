<script setup lang="ts">
import { computed } from 'vue'
import { handleBackground, isImageBackground } from '../layoutHelper'
import defaultBackground from '../assets/fairmat_slide_background.jpg'

const props = defineProps<{
  background?: string
}>()

const background = computed(() => props.background ?? defaultBackground)
const style = computed(() => handleBackground(background.value))
const hasImage = computed(() => isImageBackground(background.value))
</script>

<template>
  <div
    class="slidev-layout cover"
    :class="{ 'has-bg-image': hasImage }"
    :style="style"
  >
    <div class="my-auto w-full">
      <slot />
    </div>
  </div>
</template>
