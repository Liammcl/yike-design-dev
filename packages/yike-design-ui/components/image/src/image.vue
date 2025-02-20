<template>
  <div
    v-observe-visibility="handleVisibilityChange"
    :class="cls"
    :style="sizeStyle"
  >
    <img
      :class="bem('img')"
      :style="fitStyle"
      :src="imageSrc"
      v-bind="imgAttrs"
      @load="onImgLoaded"
      @error="onImgLoadError"
      @click="onImageClick"
    />
    <div v-if="!isLoaded" :class="bem('overlay-container')">
      <slot v-if="isError" name="error">
        <div :class="bem('overlay', ['error'])">
          <div class="icon">
            <slot name="error-icon">
              <IconImageBackupOutline />
            </slot>
          </div>
          <div v-if="props.alt" class="alt">
            {{ props.alt }}
          </div>
        </div>
      </slot>
      <slot v-if="isLoading" name="loading">
        <div :class="bem('overlay', ['loading'])"></div>
      </slot>
    </div>
    <div v-if="isShowFooter" :class="footerCls">
      <div>
        <div class="title">{{ props.title }}</div>
        <div class="description">{{ props.description }}</div>
      </div>
      <div>
        <slot name="extra"></slot>
      </div>
    </div>
    <Preview
      v-model:visible="previewVOpen"
      :src="props.src"
      :popup-container="props.popupContainer"
      @cancel="props.onCancel"
    ></Preview>
  </div>
</template>
<script setup lang="ts">
import { computed, watch, ref } from 'vue'
import type { CSSProperties } from 'vue'
import { ImageProps } from './image'
import { pick, normalizeImageSizeProp, vObserveVisibility } from './utils'
import { createCssScope } from '../../utils'
import useImageLoadState from './hooks/use-image-load-status'
import { IconImageBackupOutline } from '../../svg-icon'
import Preview from './preview.vue'
import '../style'

defineOptions({
  name: 'YkImage',
})

const props = withDefaults(defineProps<ImageProps>(), {
  preview: true,
  isLazy: false,
  footerPosition: 'inner',
})

const bem = createCssScope('image')

const previewVOpen = ref(false)

const imageSrc = ref<string>()

const { isLoaded, isError, isLoading, setLoadStatus } = useImageLoadState()

const cls = computed(() => [
  bem(),
  {
    [`${bem('with-error')}`]: isError.value,
    [`${bem('with-loading')}`]: isLoading.value,
  },
])

const footerCls = computed(() => [bem('footer', [props.footerPosition])])

const imgAttrs = computed(() =>
  pick(props, ['width', 'height', 'title', 'alt']),
)

const fitStyle = computed<CSSProperties>(() => {
  if (props.fit) return { objectFit: props.fit }
  return {}
})

const sizeStyle = computed(() => ({
  width: normalizeImageSizeProp(props.width),
  height: isShowFooter.value ? undefined : normalizeImageSizeProp(props.height),
}))

const isShowFooter = computed(() => props.title || props.description)

//记录当前是否在可视区域，为监听到src变化可及时更新
const nowVisible = ref<boolean>(false)

const handleVisibilityChange = (visible: boolean) => {
  if (props.isLazy) {
    if (!imageSrc.value && visible) imageSrc.value = props.src
  } else imageSrc.value = props.src
  nowVisible.value = visible
}

watch(
  () => props.src,
  (newSrc) => {
    setLoadStatus('loading')
    if (nowVisible.value) {
      //更新src时在可视区域内直接渲染
      imageSrc.value = newSrc
    }
    if (props.isLazy && !nowVisible.value) {
      //更新src时在非可视区域，但是需要懒加载先清空旧src等待加载
      imageSrc.value = ''
    }
  },
)

const onImgLoaded = () => setLoadStatus('loaded')

const onImgLoadError = () => setLoadStatus('error')

const onImageClick = () => {
  if (!props.preview) return
  previewVOpen.value = true
}
</script>
