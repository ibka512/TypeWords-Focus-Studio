<script setup lang="ts">
import { useSettingStore } from '@/core/stores/setting.ts'
import { useRouter } from 'vue-router'
import { IS_DEV } from '@/core/config/env'
import { withAppBaseURL } from '@/core/utils/base-url'

const settingStore = useSettingStore()
const router = useRouter()
const darkLogoSrc = withAppBaseURL('/imgs/logo/logo-text-black.png')
const lightLogoSrc = withAppBaseURL('/imgs/logo/logo-text-white.png')

function goHome() {
  if (IS_DEV) {
    router.push('/')
  } else {
    location.href = window.atob('aHR0cHM6Ly90eXBld29yZHMuY2M=')
  }
}
</script>

<template>
  <button type="button" class="logo-button center" aria-label="返回首页" @click="goHome">
    <img v-show="settingStore.theme === 'dark'" :src="lightLogoSrc" alt="Type Words" />
    <img v-show="settingStore.theme !== 'dark'" :src="darkLogoSrc" alt="Type Words" />
  </button>
</template>

<style scoped lang="scss">
.logo-button {
  padding: 0;
  border: 0;
  color: inherit;
  background: transparent;
}

img {
  cursor: pointer;
  height: 2rem;
}
</style>
