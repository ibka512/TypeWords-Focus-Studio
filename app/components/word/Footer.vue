<script setup lang="ts">
import { usePracticeStore } from '@/core/stores/practice.ts'
import { useSettingStore } from '@/core/stores/setting.ts'
import type { PracticeData } from '@/core/composables/practice-words/practice-word-session.ts'
import { ShortcutKey } from '@/core/types/enum.ts'
import type { PracticeFlowConfig, PracticeFlowCursor } from '@/core/composables/practice-words/practice-flow-types.ts'
import { BaseIcon, Tooltip } from '@/base'
import SettingDialog from '@/components/setting/SettingDialog.vue'
import VolumeSettingMiniDialog from '@/components/word/VolumeSettingMiniDialog.vue'
import StageProgress from '@/components/StageProgress.vue'
import { useI18n } from 'vue-i18n'
import {
  useInjectedDisplayActions,
  useInjectedDisplayPolicy,
} from '@/core/composables/practice-words/usePracticeDisplayPolicy.ts'
import { computed, type Ref } from 'vue'
import { getPracticeFlowDisplayState } from '@/core/composables/practice-words/practice-flow-display.ts'

const statStore = usePracticeStore()
const settingStore = useSettingStore()
const { t: $t } = useI18n()
const displayActions = useInjectedDisplayActions()
const effective = useInjectedDisplayPolicy()

const emit = defineEmits<{
  skipStep: []
}>()

let practiceData = inject<PracticeData>('practiceData')
const activeCursor = inject<Ref<PracticeFlowCursor>>('practiceFlowCursor')!
const activeFlowConfig = inject<Ref<PracticeFlowConfig>>('practiceFlowConfig')!
const bumpPracticeTimerActivity = inject<(() => void) | undefined>('bumpPracticeTimerActivity', undefined)

function onTimerRowClick() {
  if (statStore.timerPaused) {
    statStore.resumeTimer()
    bumpPracticeTimerActivity?.()
  } else {
    statStore.pauseTimer('manual')
  }
}

function format(val: number, suffix: string = '', check: number = -1) {
  return val === check ? '-' : val + suffix
}

const flowDisplay = computed(() =>
  getPracticeFlowDisplayState({
    config: activeFlowConfig.value,
    cursor: activeCursor.value,
    wordIndex: practiceData.index,
    wordCount: practiceData.words.length,
    translate: $t,
  })
)
const status = computed(() => flowDisplay.value.status)
const stages = computed(() => flowDisplay.value.stages)
const showSkipStep = computed(() => flowDisplay.value.showSkipStep)
</script>

<template>
  <div class="footer">
    <Tooltip
      :title="`${settingStore.showToolbar ? $t('collapse') : $t('expand')}(${settingStore.shortcutKeyMap[ShortcutKey.ToggleToolbar]})`"
    >
      <IconFluentChevronLeft20Filled
        @click="settingStore.showToolbar = !settingStore.showToolbar"
        class="arrow"
        :class="!settingStore.showToolbar && 'down'"
        color="#999"
      />
    </Tooltip>

    <div class="bottom">
      <StageProgress :stages="stages" />

      <div class="flex justify-between items-center">
        <div class="stat">
          <div class="row">
            <Tooltip title="进度 / 错误数 / 单词数">
              <div class="shrink-0">
                <span> {{ practiceData.index + 1 }}</span> /
                <span class="color-red"> {{ format(practiceData.wrongWords.length, '', 0) }}</span> /
                <span>{{ practiceData.words.length }}</span>
              </div>
            </Tooltip>
            <div class="line"></div>
            <div class="name">{{ status }}</div>
          </div>
          <div class="row">
            <Tooltip title="点击可暂停或恢复学习计时">
              <div class="num cursor-pointer" @click="onTimerRowClick">
                <template v-if="statStore.timerPaused">
                  <IconFluentPause20Regular width="18" height="18" class="inline-block align-middle" />
                </template>
                <template v-else> {{ Math.floor(statStore.spend / 1000 / 60) }}{{ $t('minutes') }} </template>
              </div>
            </Tooltip>
            <div class="line"></div>
            <div class="name">{{ $t('time') }}</div>
          </div>
          <div class="row">
            <Tooltip title="总错词数 | 总词数">
              <div class="num">{{ format(practiceData.allWrongWords.length, '', 0) }} | {{ statStore.total }}</div>
            </Tooltip>
            <div class="line"></div>
            <div class="name">{{ $t('total_words') }}</div>
          </div>
        </div>
        <div class="flex gap-2 justify-center items-center" id="toolbar-icons">
          <SettingDialog type="word" />

          <VolumeSettingMiniDialog />

          <BaseIcon
            v-if="showSkipStep"
            @click="emit('skipStep')"
            :title="`${$t('skip_to_next_stage')}(${settingStore.shortcutKeyMap[ShortcutKey.NextStep]})`"
          >
            <IconFluentArrowRight16Regular />
          </BaseIcon>

          <BaseIcon
            @click="displayActions.toggleDictation()"
            :title="`${$t('toggle_dictation_mode')}(${settingStore.shortcutKeyMap[ShortcutKey.ToggleDictation]})`"
          >
            <IconFluentEyeOff16Regular v-if="effective.isWordMasked" />
            <IconFluentEye16Regular v-else />
          </BaseIcon>

          <BaseIcon
            :title="`${$t('toggle_translation')}(${settingStore.shortcutKeyMap[ShortcutKey.ToggleShowTranslate]})`"
            @click="displayActions.toggleTranslate()"
          >
            <IconPhTranslate v-if="effective.isShowTranslate" />
            <IconFluentTranslateOff16Regular v-else />
          </BaseIcon>

          <BaseIcon
            @click="settingStore.showPanel = !settingStore.showPanel"
            :title="`${$t('word_list')}(${settingStore.shortcutKeyMap[ShortcutKey.TogglePanel]})`"
          >
            <IconFluentTextListAbcUppercaseLtr20Regular />
          </BaseIcon>
        </div>
      </div>
    </div>
    <div class="progress-wrap flex gap-3 items-center color-gray">
      <span class="shrink-0">{{ status }}</span>
      <StageProgress :stages="stages" />
      <Tooltip title="进度 / 错误数 / 单词数">
        <div class="shrink-0">
          <span> {{ practiceData.index + 1 }}</span> /
          <span class="color-red"> {{ format(practiceData.wrongWords.length, '', 0) }}</span> /
          <span>{{ practiceData.words.length }}</span>
        </div>
      </Tooltip>
    </div>
  </div>
</template>

<style scoped lang="scss">
.footer {
  flex-shrink: 0;
  width: var(--toolbar-width);
  position: relative;
  z-index: 20;

  &.hide {
    margin-bottom: -6rem;
    margin-top: 3rem;

    .progress-wrap {
      bottom: calc(100% + 1.8rem);
    }
  }

  .bottom {
    position: relative;
    z-index: 10;
    width: 100%;
    box-sizing: border-box;
    margin-bottom: 0.75rem;
    padding: 0.875rem 1rem calc(0.875rem + env(safe-area-inset-bottom, 0px));
    border: 1px solid var(--focus-border);
    border-radius: 1.125rem;
    background: color-mix(in srgb, var(--focus-surface) 94%, transparent);
    box-shadow: var(--focus-shadow-md);
    backdrop-filter: blur(18px);

    .stat {
      display: flex;
      justify-content: flex-start;
      gap: 0;
      margin-top: 0.625rem;

      .row {
        min-width: 5.25rem;
        display: flex;
        flex-direction: column;
        align-items: flex-start;
        gap: 0.125rem;
        padding: 0 1rem;
        color: var(--focus-ink-secondary);
        font-size: 0.8125rem;

        &:first-child {
          padding-left: 0;
        }

        & + .row {
          border-left: 1px solid var(--focus-border);
        }

        .line {
          display: none;
        }

        .num,
        > div:first-child {
          color: var(--focus-ink);
          font-weight: 680;
        }

        .name {
          color: var(--focus-ink-tertiary);
          font-size: 0.6875rem;
        }
      }
    }
  }

  .progress-wrap {
    width: var(--toolbar-width);
    transition: bottom 0.3s ease;
    padding: 0 0.6rem;
    box-sizing: border-box;
    position: fixed;
    bottom: 1rem;
    z-index: 1;
  }

  .arrow {
    position: absolute;
    top: -2.5rem;
    left: 50%;
    cursor: pointer;
    transition:
      top var(--focus-duration) var(--focus-ease),
      transform var(--focus-duration) var(--focus-ease);
    transform: rotate(-90deg);
    padding: 0.5rem;
    font-size: 1.2rem;

    &.down {
      top: -3.25rem;
      transform: rotate(90deg);
    }
  }
}

@media (max-width: 768px) {
  .footer {
    width: 100%;

    .bottom {
      padding: 0.75rem 0.75rem calc(0.75rem + env(safe-area-inset-bottom, 0px));
      border-radius: 1rem;

      .stat {
        margin-top: 0.5rem;
        gap: 0;
        flex-direction: row;
        overflow-x: auto;

        .row {
          min-width: 4rem;
          gap: 0.125rem;
          padding-inline: 0.625rem;

          .num {
            font-size: 0.8rem;
            font-weight: bold;
          }

          .name {
            font-size: 0.7rem;
          }
        }
      }

      .flex.gap-2 {
        display: grid;
        grid-template-columns: repeat(5, 2.5rem);
        gap: 0.25rem;
        justify-content: center;

        .base-icon {
          padding: 0;
          font-size: 1rem;
          min-height: 44px;
          min-width: 44px;
          display: flex;
          align-items: center;
          justify-content: center;
        }
      }
    }

    .progress-wrap {
      width: 100%;
      padding: 0 0.5rem;
      bottom: 0.5rem;
    }

    .arrow {
      font-size: 1rem;
      padding: 0.3rem;
    }
  }
}

// 超小屏幕适配
@media (max-width: 480px) {
  .footer {
    .bottom {
      padding: 0.625rem;

      .stat {
        margin-top: 0.375rem;
        gap: 0;

        .row {
          min-width: 3.5rem;
          gap: 0.1rem;

          .num {
            font-size: 0.7rem;
          }

          .name {
            font-size: 0.6rem;
          }

          // 隐藏部分统计信息，只保留关键数据
          &:nth-child(n + 3) {
            display: none;
          }
        }
      }

      .flex.gap-2 {
        gap: 0.2rem;

        .base-icon {
          padding: 0.2rem;
          font-size: 0.9rem;
        }
      }
    }

    .progress-wrap {
      padding: 0 0.3rem;
      bottom: 0.3rem;
    }
  }
}

@media (prefers-reduced-motion: reduce) {
  .footer .progress-wrap,
  .footer .arrow {
    transition-duration: 0.01ms;
  }
}
</style>
