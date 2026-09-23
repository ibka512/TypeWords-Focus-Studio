<script setup lang="ts">
import { BaseIcon, ToastComponent } from '@/base'
import Logo from '@/components/Logo.vue'
import IeDialog from '@/components/dialog/IeDialog.vue'
import useTheme from '@/core/hooks/theme.ts'
import { useRuntimeStore } from '@/core/stores/runtime.ts'
import { useSettingStore } from '@/core/stores/setting.ts'
import { ShortcutKey } from '@/core/types/enum.ts'
import { onMounted, watch } from 'vue'
import { useRoute } from 'vue-router'
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css'
import { useInit } from '@/core/composables/useInit.ts'
import { useI18n } from 'vue-i18n'
import { Supabase } from '@/core/utils/supabase.ts'
import MiniProgram from '@/components/MiniProgram.vue'
import WordCollectPopover from '@/components/word/WordCollectPopover.vue'

const { toggleTheme, getTheme, setTheme } = useTheme()
const runtimeStore = useRuntimeStore()
const settingStore = useSettingStore()
const runtimeConfig = useRuntimeConfig()
const init = useInit()
const route = useRoute()
const { locales, setLocale } = useI18n()

let expand = $ref(true)
let localeOpen = $ref(false)

function toggleExpand(value: boolean) {
  expand = value
  document.documentElement.style.setProperty('--aside-width', value ? '15rem' : '5rem')
}

function togglePinnedNavigation() {
  settingStore.sideExpand = !settingStore.sideExpand
  toggleExpand(settingStore.sideExpand)
}

watch(
  () => settingStore.load,
  loaded => {
    if (!loaded) return
    toggleExpand(settingStore.sideExpand)
    setTheme(settingStore.theme)
  }
)

watch(
  () => settingStore.theme,
  theme => setTheme(theme)
)

watch(
  () => route.fullPath,
  () => (localeOpen = false)
)

const appBasePath = String(runtimeConfig.app.baseURL || '/').replace(/\/$/, '')
const logicalRoutePath = $computed(() => {
  const strippedPath =
    !appBasePath || appBasePath === '/' || !route.path.startsWith(appBasePath)
      ? route.path
      : route.path.slice(appBasePath.length) || '/'
  return strippedPath === '/' ? strippedPath : strippedPath.replace(/\/+$/, '')
})

const showUtilities = $computed(() =>
  ['/words', '/articles', '/setting', '/help', '/doc', '/feedback'].includes(logicalRoutePath)
)

const immersive = $computed(() =>
  ['/practice-words/', '/practice-articles/', '/words-test/'].some(prefix => logicalRoutePath.includes(prefix))
)

const currentSection = $computed(() => {
  if (logicalRoutePath.includes('article') || logicalRoutePath.includes('book')) return '文章学习'
  if (logicalRoutePath.includes('setting')) return '偏好设置'
  if (logicalRoutePath.includes('help') || logicalRoutePath.includes('doc')) return '帮助与资料'
  if (logicalRoutePath.includes('feedback')) return '反馈'
  return '单词训练'
})

onMounted(() => {
  init()
  window.umami?.track('sync', { check: Supabase.check() })
})

function onMouseEnter() {
  if (!settingStore.sideExpand) toggleExpand(true)
}

function onMouseLeave() {
  if (!settingStore.sideExpand) toggleExpand(false)
}
</script>

<template>
  <div class="focus-layout" :class="{ 'is-immersive': immersive }">
    <aside
      class="focus-sidebar"
      :class="{ 'is-collapsed': !expand }"
      @mouseenter="onMouseEnter"
      @mouseleave="onMouseLeave"
    >
      <div class="focus-sidebar__top">
        <div class="focus-brand">
          <Logo class="focus-brand__logo" />
          <BaseIcon
            class="focus-sidebar__pin"
            :title="settingStore.sideExpand ? '收起侧边栏' : '固定侧边栏'"
            @click="togglePinnedNavigation"
          >
            <IconFluentPin24Filled v-if="settingStore.sideExpand" />
            <IconFluentPin20Regular v-else />
          </BaseIcon>
        </div>

        <p v-if="expand" class="focus-sidebar__label">学习空间</p>
        <nav class="focus-nav" aria-label="主要导航">
          <NuxtLink to="/words" class="focus-nav__item">
            <IconFluentTextUnderlineDouble20Regular />
            <span>{{ $t('words') }}</span>
          </NuxtLink>
          <NuxtLink id="article" to="/articles" class="focus-nav__item">
            <IconFluentBookLetter20Regular />
            <span>{{ $t('articles') }}</span>
          </NuxtLink>
        </nav>

        <p v-if="expand" class="focus-sidebar__label focus-sidebar__label--secondary">资源</p>
        <nav class="focus-nav" aria-label="帮助导航">
          <NuxtLink to="/doc" class="focus-nav__item">
            <IconFluentDocument20Regular />
            <span>{{ $t('document') }}</span>
          </NuxtLink>
          <NuxtLink to="/help" class="focus-nav__item">
            <IconFluentQuestionCircle20Regular />
            <span>{{ $t('help') }}</span>
          </NuxtLink>
          <NuxtLink to="/feedback" class="focus-nav__item">
            <IconFluentCommentEdit20Regular />
            <span>{{ $t('feedback') }}</span>
          </NuxtLink>
        </nav>
      </div>

      <div class="focus-sidebar__bottom">
        <NuxtLink to="/setting" class="focus-nav__item">
          <IconFluentSettings20Regular />
          <span>{{ $t('setting') }}</span>
          <span v-if="runtimeStore.isError" class="focus-status-dot" aria-label="同步异常"></span>
        </NuxtLink>
        <p v-if="expand" class="focus-sidebar__hint">所有学习数据优先保存在本地</p>
      </div>
    </aside>

    <div class="focus-workspace">
      <header v-if="showUtilities" class="focus-topbar">
        <div>
          <p class="focus-topbar__eyebrow">FOCUS STUDIO</p>
          <p class="focus-topbar__title">{{ currentSection }}</p>
        </div>
        <div class="focus-topbar__actions">
          <MiniProgram v-if="settingStore.load && !settingStore.first" />

          <div class="focus-locale">
            <BaseIcon title="切换语言" :active="localeOpen" @click="localeOpen = !localeOpen">
              <IconPhTranslate />
            </BaseIcon>
            <div v-if="localeOpen" class="focus-locale__menu" role="menu">
              <button
                v-for="locale in locales"
                :key="locale.code"
                type="button"
                role="menuitem"
                @click="setLocale(locale.code); localeOpen = false"
              >
                {{ locale.name }}
              </button>
            </div>
          </div>

          <BaseIcon
            :title="`${$t('toggle_theme')}(${settingStore.shortcutKeyMap[ShortcutKey.ToggleTheme]})`"
            @click="toggleTheme"
          >
            <IconFluentWeatherMoon16Regular v-if="getTheme() === 'light'" />
            <IconFluentWeatherSunny16Regular v-else />
          </BaseIcon>
        </div>
      </header>

      <div
        v-if="runtimeStore.isError"
        class="focus-sync-error"
        role="button"
        tabindex="0"
        @click="navigateTo('/setting?index=6')"
        @keydown.enter="navigateTo('/setting?index=6')"
      >
        <ToastComponent
          type="error"
          :duration="0"
          :shadow="false"
          :showClose="false"
          :message="$t('sync_failed_toast')"
        />
      </div>

      <main class="focus-main">
        <router-view />
      </main>
    </div>

    <nav v-if="!immersive" class="focus-mobile-nav" aria-label="移动端主要导航">
      <NuxtLink to="/words" class="focus-mobile-nav__item">
        <IconFluentTextUnderlineDouble20Regular />
        <span>{{ $t('words') }}</span>
      </NuxtLink>
      <NuxtLink to="/articles" class="focus-mobile-nav__item">
        <IconFluentBookLetter20Regular />
        <span>{{ $t('articles') }}</span>
      </NuxtLink>
      <NuxtLink to="/setting" class="focus-mobile-nav__item">
        <IconFluentSettings20Regular />
        <span>{{ $t('setting') }}</span>
        <span v-if="runtimeStore.isError" class="focus-status-dot" aria-label="同步异常"></span>
      </NuxtLink>
    </nav>

    <IeDialog />
    <WordCollectPopover />
  </div>
</template>

<style scoped lang="scss">
.focus-layout {
  min-height: 100vh;
  display: flex;
  background: var(--focus-canvas);
}

.focus-sidebar {
  position: fixed;
  inset: 0 auto 0 0;
  z-index: 30;
  width: var(--aside-width);
  min-width: var(--aside-width);
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  box-sizing: border-box;
  padding: 1.25rem 1rem;
  border-right: 1px solid var(--focus-border);
  background: color-mix(in srgb, var(--focus-surface) 92%, transparent);
  backdrop-filter: blur(18px);
  overflow: hidden;
  transition: width var(--focus-duration) var(--focus-ease);

  &__top,
  &__bottom {
    min-width: 3rem;
  }

  &__bottom {
    padding-top: 1rem;
    border-top: 1px solid var(--focus-border);
  }

  &__pin {
    flex: 0 0 auto;
  }

  &__label {
    margin: 2rem 0 0.5rem 0.75rem;
    color: var(--focus-ink-tertiary);
    font-size: 0.6875rem;
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
  }

  &__label--secondary {
    margin-top: 1.5rem;
  }

  &__hint {
    margin: 0.75rem 0.75rem 0;
    color: var(--focus-ink-tertiary);
    font-size: 0.75rem;
    line-height: 1.5;
  }

  &.is-collapsed {
    padding-inline: 1rem;

    .focus-nav__item span,
    .focus-sidebar__hint {
      opacity: 0;
    }

    .focus-brand__logo {
      width: 2.75rem;
      overflow: hidden;

      :deep(img) {
        max-width: 2.75rem;
        height: auto;
        object-fit: contain;
      }
    }
  }
}

.focus-brand {
  min-height: 2.75rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding-inline: 0.25rem;

}

.focus-nav {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;

  &__item {
    min-height: 2.875rem;
    display: flex;
    align-items: center;
    gap: 0.75rem;
    box-sizing: border-box;
    padding: 0.625rem 0.75rem;
    border-radius: 0.75rem;
    color: var(--focus-ink-secondary);
    font-size: 0.9375rem;
    font-weight: 580;
    white-space: nowrap;
    transition:
      color var(--focus-duration) var(--focus-ease),
      background-color var(--focus-duration) var(--focus-ease);

    &:hover {
      color: var(--focus-ink);
      background: var(--focus-surface-strong);
    }

    &.router-link-active {
      color: var(--focus-accent);
      background: var(--focus-accent-soft);
    }

    svg {
      width: 1.25rem;
      height: 1.25rem;
      flex: 0 0 auto;
    }

    span {
      transition: opacity 120ms ease;
    }
  }
}

.focus-status-dot {
  width: 0.5rem;
  height: 0.5rem;
  margin-left: auto;
  border: 2px solid var(--focus-surface);
  border-radius: 50%;
  background: var(--focus-danger);
}

.focus-workspace {
  width: calc(100% - var(--aside-width));
  min-height: 100vh;
  margin-left: var(--aside-width);
  transition:
    width var(--focus-duration) var(--focus-ease),
    margin-left var(--focus-duration) var(--focus-ease);
}

.focus-topbar {
  height: 4.75rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  box-sizing: border-box;
  padding: 0 2rem;
  border-bottom: 1px solid var(--focus-border);
  background: color-mix(in srgb, var(--focus-canvas) 88%, transparent);
  backdrop-filter: blur(16px);

  &__eyebrow {
    margin: 0;
    color: var(--focus-ink-tertiary);
    font-size: 0.625rem;
    font-weight: 750;
    letter-spacing: 0.16em;
  }

  &__title {
    margin: 0.125rem 0 0;
    color: var(--focus-ink);
    font-size: 0.9375rem;
    font-weight: 650;
  }

  &__actions {
    display: flex;
    align-items: center;
    gap: 0.375rem;
  }
}

.focus-locale {
  position: relative;

  &__menu {
    position: absolute;
    top: calc(100% + 0.625rem);
    right: 0;
    z-index: 50;
    width: 12rem;
    max-height: min(24rem, 70vh);
    padding: 0.5rem;
    border: 1px solid var(--focus-border);
    border-radius: 0.875rem;
    background: var(--focus-surface);
    box-shadow: var(--focus-shadow-md);
    overflow-y: auto;

    button {
      width: 100%;
      min-height: 2.5rem;
      padding: 0.5rem 0.75rem;
      border: 0;
      border-radius: 0.625rem;
      color: var(--focus-ink);
      background: transparent;
      text-align: left;
      cursor: pointer;

      &:hover {
        background: var(--focus-surface-strong);
      }
    }
  }
}

.focus-main {
  min-height: calc(100vh - 4.75rem);
}

.focus-sync-error {
  position: fixed;
  top: 1rem;
  left: 50%;
  z-index: 100;
  transform: translateX(-50%);
  cursor: pointer;
}

.focus-mobile-nav {
  display: none;
}

.focus-layout.is-immersive {
  .focus-sidebar {
    display: none;
  }

  .focus-workspace {
    width: 100%;
    margin-left: 0;
  }

  .focus-main {
    min-height: 100vh;
  }
}

@media (max-width: 768px) {
  .focus-sidebar,
  .focus-topbar {
    display: none;
  }

  .focus-workspace {
    width: 100%;
    margin-left: 0;
  }

  .focus-main {
    min-height: 100vh;
    padding-bottom: calc(5.25rem + env(safe-area-inset-bottom, 0px));
  }

  .focus-mobile-nav {
    position: fixed;
    right: 0.75rem;
    bottom: calc(0.75rem + env(safe-area-inset-bottom, 0px));
    left: 0.75rem;
    z-index: 80;
    min-height: 4rem;
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    padding: 0.375rem;
    border: 1px solid var(--focus-border);
    border-radius: 1.25rem;
    background: color-mix(in srgb, var(--focus-surface) 92%, transparent);
    box-shadow: var(--focus-shadow-md);
    backdrop-filter: blur(18px);

    &__item {
      position: relative;
      min-height: 3.25rem;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      gap: 0.125rem;
      border-radius: 0.875rem;
      color: var(--focus-ink-secondary);
      font-size: 0.6875rem;
      font-weight: 650;

      &.router-link-active {
        color: var(--focus-accent);
        background: var(--focus-accent-soft);
      }

      svg {
        width: 1.25rem;
        height: 1.25rem;
      }

      .focus-status-dot {
        position: absolute;
        top: 0.4rem;
        right: calc(50% - 1.15rem);
      }
    }
  }
}
</style>
