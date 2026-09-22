<script setup lang="ts">
import { useBaseStore } from '@/core/stores/base.ts'
import { useRouter } from 'vue-router'
import {
  BaseButton,
  BaseIcon,
  BasePage,
  Calendar,
  DeleteIcon,
  Dialog,
  OptionButton,
  PopConfirm,
  Progress,
  Switch,
  Toast,
  Tooltip,
} from '@/base'
import {
  _getAccomplishDate,
  _getDictDataByUrl,
  _nextTick,
  debounce,
  getShufflePracticeWords,
  isMobile,
  loadJsLib,
  msToHourMinute,
  resourceWrap,
  type ShufflePracticeSetting,
  total,
  useNav,
} from '@/core/utils'
import type { DictResource, Statistics } from '@/core/types/types.ts'
import { onMounted, onUnmounted, watch } from 'vue'
import { useRuntimeStore } from '@/core/stores/runtime.ts'
import Book from '@/components/Book.vue'
import { getDefaultDict } from '@/core/types/func.ts'
import PracticeSettingDialog from '@/components/word/PracticeSettingDialog.vue'
import ChangeLastPracticeIndexDialog from '@/components/word/ChangeLastPracticeIndexDialog.vue'
import { useSettingStore } from '@/core/stores/setting.ts'
import { useFetch } from '@vueuse/core'
import {
  APP_NAME,
  DICT_LIST,
  LIB_JS_URL,
  Old_Host,
  Origin,
  TourConfig,
  WordPracticeModeNameMap,
  WordPracticeModeUrlMap,
} from '@/core/config/env.ts'
import PracticeWordListDialog from '@/components/word/PracticeWordListDialog.vue'
import ShufflePracticeSettingDialog from '@/components/word/ShufflePracticeSettingDialog.vue'
import { flushStatToStore } from '@/core/composables/usePracticePersistence'
import { useDataSyncPersistence } from '@/core/composables/useDataSyncPersistence'
import { WordPracticeMode } from '@/core/types/enum.ts'
import {
  type PracticeWordCache,
  UnsupportedPracticeCacheVersionError,
  usePracticeWordPersistence,
} from '@/core/composables/practice-words/practice-word-session.ts'
import dayjs from 'dayjs'
import { getActiveCustomFlowId, getUserFlow } from '@/core/composables/practice-words/practice-flow-runtime.ts'
import { createStudyTask } from '@/core/composables/practice-words/study-task.ts'

const store = useBaseStore()
const settingStore = useSettingStore()
const wordPersistence = usePracticeWordPersistence()
const dataSync = useDataSyncPersistence()
const router = useRouter()
const { nav } = useNav()
const runtimeStore = useRuntimeStore()
let loading = $ref(true)
let isSaveData = $ref(false)
let unsupportedCacheVersion = false

async function loadPracticeCache() {
  try {
    return await wordPersistence.load()
  } catch (error) {
    if (!(error instanceof UnsupportedPracticeCacheVersionError)) throw error
    unsupportedCacheVersion = true
    Toast.error('练习缓存来自更高版本，请升级后再继续')
    return null
  }
}

const shouldShowDialogPracticeMode = [WordPracticeMode.Shuffle, WordPracticeMode.ShuffleWordsTest]

useSeoMeta({
  title: `在线背单词与英语打字练习｜${APP_NAME}`,
  description: '在电脑上选择 CET-4、CET-6、考研、GRE、IELTS 等词库，通过键盘跟打、拼写和科学间隔复习高效背单词。',
  ogTitle: `在线背单词与英语打字练习｜${APP_NAME}`,
  ogDescription: '在电脑上用键盘打字背单词，支持 50+ 词库和科学间隔复习。',
  twitterTitle: `在线背单词与英语打字练习｜${APP_NAME}`,
  twitterDescription: '在电脑上用键盘打字背单词，支持 50+ 词库和科学间隔复习。',
})

let practiceData = $ref<PracticeWordCache>({
  taskWords: {
    new: [],
    review: [],
  },
} as any)
let dueReviewCount = $ref(0)

function refreshStudyTask() {
  const result = createStudyTask()
  practiceData.taskWords = result.taskWords
  dueReviewCount = result.dueReviewCount
  return result
}

function toggleAutoAddRandomReview(enabled: boolean) {
  settingStore.autoAddRandomReviewWhenNoDue = enabled
  const result = refreshStudyTask()
  if (!enabled) return
  if (result.randomReviewCount > 0) {
    Toast.success(`已将 ${result.randomReviewCount} 个随机复习词加入本次学习`)
  } else {
    Toast.warning('暂无单词可以复习，先学习一些新词后再来看看吧')
  }
}

const effectiveReviewRatio = $computed(() => {
  const dict = store.sdict
  const isEnd = dict.length !== 1 && dict.lastLearnIndex >= dict.length - 1
  return isEnd ? settingStore.wordReviewRatio || 1 : settingStore.wordReviewRatio
})

const reviewWordLimit = $computed(() => {
  return Math.max(0, Math.floor(store.sdict.perDayStudyNumber * effectiveReviewRatio))
})

const reviewWordTip = $computed(() => {
  const dailyGoal = store.sdict.perDayStudyNumber
  const actualCount = practiceData?.taskWords?.review?.length ?? 0
  const rule = `复习词来自记忆曲线中今天及以前到期的已学单词，并会排除本组新词、已掌握词和已忽略词。“${effectiveReviewRatio} 倍”只决定数量上限：每日新词目标 ${dailyGoal} × ${effectiveReviewRatio}，本组最多安排 ${reviewWordLimit} 个。\n`

  if (isSaveData) {
    return `${rule}当前是已生成的未完成任务，共安排 ${actualCount} 个复习词；\n实际数量取决于任务生成时符合条件的到期词，不会用未到期词补足。`
  }
  if (reviewWordLimit === 0) {
    return `${rule}当前数量上限为 0，因此本组不安排复习词。`
  }
  if (dueReviewCount === 0 && actualCount > 0) {
    return `${rule}当前没有到期复习词，已按“加入随机复习”设置从已学单词中随机加入 ${actualCount} 个。`
  }
  if (actualCount < reviewWordLimit) {
    return `${rule}当前只有 ${actualCount} 个符合条件的到期词，因此本组安排 ${actualCount} 个，不会用未到期词补足。`
  }
  return `${rule}当前本组安排 ${actualCount} 个，已达到数量上限。`
})

async function resetCacheData() {
  if (unsupportedCacheVersion) return
  isSaveData && flushStatToStore(practiceData.statStoreData)
  isSaveData = false
  practiceData.practiceData = null
  practiceData.statStoreData = null
  practiceData.sessionSnapshot = undefined
  await wordPersistence.clear()
}

// runtimeStore.globalLoading练习界面，退出时会调用一个保存，可能会卡住。当调用完成再init
//  immediate: true 比 onUmMounted 先执行，只能延时执行
watch(
  [() => store.load, () => runtimeStore.globalLoading],
  debounce(([a, b]) => {
    if (a && !b) {
      init()
      _nextTick(async () => {
        const Shepherd = await loadJsLib('Shepherd', LIB_JS_URL.SHEPHERD)
        const tour = new Shepherd.Tour(TourConfig)
        tour.on('cancel', () => {
          localStorage.setItem('tour-guide', '1')
        })
        tour.addStep({
          id: 'step1',
          text: '点击这里选择一本词典开始学习',
          attachTo: {
            element: '#step1',
            on: 'bottom',
          },
          buttons: [
            {
              text: `下一步（1/${TourConfig.total}）`,
              action() {
                tour.next()
                router.push('/dict-list')
              },
            },
          ],
        })
        const r = localStorage.getItem('tour-guide')
        if (settingStore.first && !r && !isMobile()) tour.start()
      }, 500)
    }
  }),
  { immediate: true }
)

async function onvisibilitychange() {
  if (!document.hidden) {
    //当页面可见时，检查是否需要从缓存恢复
    const d = await loadPracticeCache()
    if (d) {
      practiceData = d
      isSaveData = true
    }
  }
}

async function init() {
  document.removeEventListener('visibilitychange', onvisibilitychange)
  document.addEventListener('visibilitychange', onvisibilitychange)

  let studyIndex = store.word.studyIndex
  if (studyIndex >= 3) {
    if (!store.sdict.custom && !store.sdict.words.length) {
      let dictList = await fetch(resourceWrap(DICT_LIST.WORD.ALL)).then(r => r.json())
      let dict = await _getDictDataByUrl(store.sdict)
      let r = dictList.find(v => [v.enName, v.id].includes(store.sdict.id))
      if (r) {
        store.word.bookList[studyIndex].words = dict.words
        store.word.bookList[studyIndex].id = r.id
        store.word.bookList[studyIndex].enName = r.enName
        store.word.bookList[studyIndex].cover = r.cover
        store.word.bookList[studyIndex].category = r.category
        store.word.bookList[studyIndex].tags = r.tags
        store.word.bookList[studyIndex].url = r.url
        store.word.bookList[studyIndex].description = r.description
        store.word.bookList[studyIndex].name = r.name
      } else {
        store.word.bookList[studyIndex] = dict
      }
      store.word.bookList[studyIndex].length = dict.words.length
      let s = store.word.bookList[studyIndex]
      if (s.lastLearnIndex > s.length) {
        store.word.bookList[studyIndex].lastLearnIndex = s.length
        store.word.bookList[studyIndex].complete = true
        await resetCacheData()
      }
    }
  }

  if (!practiceData?.taskWords.new.length && store.sdict.words.length) {
    const d = await loadPracticeCache()
    if (d) {
      practiceData = d
      isSaveData = true
    } else if (!unsupportedCacheVersion) {
      refreshStudyTask()
    }
  }
  loading = false
}

async function startPractice(practiceMode: WordPracticeMode, resetCache: boolean = false): Promise<void> {
  if (unsupportedCacheVersion) {
    Toast.error('当前客户端无法读取这份练习缓存，请升级后再继续')
    return
  }
  if (practiceMode === WordPracticeMode.Custom) {
    const activeCustomFlowId = getActiveCustomFlowId()
    if (!activeCustomFlowId || !getUserFlow(activeCustomFlowId)) {
      Toast.warning('请先创建并激活一个自定义流程')
      router.push('/practice-flow-editor')
      return
    }
  }
  if (resetCache) await resetCacheData()

  if (shouldShowDialogPracticeMode.includes(practiceMode) && !isSaveData) {
    editingWordPracticeMode = practiceMode
    showShufflePracticeSettingDialog = true
    return
  }

  if (store.sdict.id) {
    if (!store.sdict.words.length) {
      Toast.warning('没有单词可学习！')
      return
    }

    settingStore.wordPracticeMode = practiceMode

    window.umami?.track('startStudyWord', {
      name: store.sdict.name,
      index: String(store.sdict.lastLearnIndex),
      perDayStudyNumber: String(store.sdict.perDayStudyNumber),
      custom: store.sdict.custom,
      complete: store.sdict.complete,
      wordPracticeMode: String(settingStore.wordPracticeMode),
    })
    //把是否是第一次设置为false
    if (settingStore.first) settingStore.first = false
    nav(WordPracticeModeUrlMap[practiceMode] + '/' + store.sdict.id, {}, practiceData)
  } else {
    window.umami?.track('no-dict')
    Toast.warning('请先选择一本词典')
  }
}

function freePractice() {
  startPractice(WordPracticeMode.Free, settingStore.wordPracticeMode !== WordPracticeMode.Free)
}

function systemPractice() {
  const currentMode = settingStore.wordPracticeMode
  const isFree = currentMode === WordPracticeMode.Free
  startPractice(isFree ? WordPracticeMode.System : currentMode, isFree)
}

let editingWordPracticeMode = $ref(0)

let showPracticeSettingDialog = $ref(false)
let showShufflePracticeSettingDialog = $ref(false)
let showChangeLastPracticeIndexDialog = $ref(false)
let showPracticeWordListDialog = $ref(false)

type StudyDayRow = Statistics & { dictName: string }

let showStudyDayDialog = $ref(false)
let selectedStudyDateKey = $ref('')
let studyDayRecords = $ref<StudyDayRow[]>([])

const allWordStatistics = $computed(() => store.word.bookList.flatMap(book => book.statistics ?? []))

const cacheSpendMs = $computed(() => practiceData.statStoreData?.spend ?? 0)

const todayDateKey = $computed(() => dayjs().format('YYYY-MM-DD'))

/**
 * 缓存记录中每一天对应的学习毫秒数 Map<'YYYY-MM-DD', spendMs>
 * 有 segments 时按片段精确分组，否则退回到 startDate + spend 整体归一天
 */
const cacheDaySpendMap = $computed((): Map<string, number> => {
  const st = practiceData.statStoreData
  const map = new Map<string, number>()
  if (!st?.spend) return map
  if (Array.isArray(st.segments) && st.segments.length > 0) {
    for (const [segStart, segEnd] of st.segments) {
      const key = dayjs(segStart).format('YYYY-MM-DD')
      map.set(key, (map.get(key) ?? 0) + (segEnd - segStart))
    }
  } else {
    // 老数据 / 无 segments：全部归到 startDate 那天
    map.set(dayjs(st.startDate).format('YYYY-MM-DD'), st.spend)
  }
  return map
})

const todayCacheMs = $computed(() => cacheDaySpendMap.get(todayDateKey) ?? 0)

const calendarHighlightDates = $computed(() => {
  const set = new Set<string>()
  for (const s of allWordStatistics) {
    set.add(dayjs(s.startDate).format('YYYY-MM-DD'))
  }
  // 把缓存记录中所有出现过的天都高亮（支持跨天）
  for (const key of cacheDaySpendMap.keys()) {
    set.add(key)
  }
  return [...set]
})

/** 已落库统计总毫秒（全 bookList） */
const persistedTotalMs = $computed(() => total(allWordStatistics, 'spend'))

const totalSpend = $computed(() => {
  const sum = persistedTotalMs + cacheSpendMs
  if (!sum) return 0
  return msToHourMinute(sum)
})

const todayTotalSpend = $computed(() => {
  const todayPersistedMs = total(
    allWordStatistics.filter(v => dayjs(v.startDate).isSame(dayjs(), 'day')),
    'spend'
  )
  const sum = todayPersistedMs + todayCacheMs
  if (!sum) return 0
  return msToHourMinute(sum)
})

const totalDay = $computed(() => {
  const set = new Set(allWordStatistics.map(v => dayjs(v.startDate).format('YYYY-MM-DD')))
  // 把缓存记录中所有出现过的天都计入（支持跨天）
  for (const key of cacheDaySpendMap.keys()) {
    set.add(key)
  }
  return set.size
})

const studyDayDialogTitle = $computed(() =>
  selectedStudyDateKey ? `${dayjs(selectedStudyDateKey).format('YYYY年M月D日')} 学习记录` : ''
)

function isStudyDayKeyToday(dateKey: string) {
  return dateKey === dayjs().format('YYYY-MM-DD')
}

function onSelectCalendarDate(dateKey: string) {
  selectedStudyDateKey = dateKey
  const rows: StudyDayRow[] = []
  for (const book of store.word.bookList) {
    for (const stat of book.statistics ?? []) {
      if (dayjs(stat.startDate).format('YYYY-MM-DD') === dateKey) {
        rows.push({ ...stat, dictName: book.name })
      }
    }
  }
  const st = practiceData.statStoreData
  // 缓存记录跨天时，只要该天在 cacheDaySpendMap 中有记录就展示
  if (st?.spend && cacheDaySpendMap.has(dateKey)) {
    const daySpend = cacheDaySpendMap.get(dateKey)!
    const cacheKeys = [...cacheDaySpendMap.keys()]
    const keyIdx = cacheKeys.indexOf(dateKey)
    const isMultiDay = cacheKeys.length > 1
    // 推算该天在整次练习中的角色（练习未结束，最后一天标为"学习中"而非"学习结束"）
    let sessionRole: StudyDayRow['sessionRole']
    if (!isMultiDay) {
      sessionRole = 'single'
    } else if (keyIdx === 0) {
      sessionRole = 'start'
    } else if (keyIdx === cacheKeys.length - 1) {
      sessionRole = 'middle' // 最后一天仍在进行中，用 middle 表示
    } else {
      sessionRole = 'middle'
    }
    rows.push({
      ...st,
      spend: daySpend,
      new: st.newWordNumber,
      review: st.reviewWordNumber,
      dictName: store.sdict.name,
      sessionRole,
    })
  }
  if (!rows.length) return Toast.info('无学习记录')
  studyDayRecords = rows
  showStudyDayDialog = true
}

async function goDictDetail(val: DictResource) {
  if (!val.id) return nav('dict-list')
  runtimeStore.editDict = getDefaultDict(val)
  nav('/dict', {})
}

let isManageDict = $ref(false)
let selectIds = $ref([])

async function handleBatchDel() {
  selectIds.forEach(id => {
    let r = store.word.bookList.findIndex(v => v.id === id)
    if (r !== -1) {
      if (store.word.studyIndex === r) {
        store.word.studyIndex = -1
      }
      if (store.word.studyIndex > r) {
        store.word.studyIndex--
      }
      store.word.bookList.splice(r, 1)
    }
  })
  selectIds = []
  Toast.success('删除成功！')
}

function toggleSelect(item) {
  let rIndex = selectIds.findIndex(v => v === item.id)
  if (rIndex > -1) {
    selectIds.splice(rIndex, 1)
  } else {
    selectIds.push(item.id)
  }
}

const progressTextLeft = $computed(() => {
  if (store.sdict.complete) return '已学完，进入总复习阶段'
  return '当前进度：已学' + store.currentStudyProgress + '%'
})

function check(cb: Function) {
  if (!store.sdict.id) {
    Toast.warning('请先选择一本词典')
  } else {
    runtimeStore.editDict = getDefaultDict(store.sdict)
    cb()
  }
}

async function savePracticeSetting() {
  await resetCacheData()
  await store.changeDict(runtimeStore.editDict)
  refreshStudyTask()
  Toast.success('修改成功')
}

async function onShufflePracticeSettingOk(setting: ShufflePracticeSetting) {
  await dataSync.saveDictState()
  await resetCacheData()
  settingStore.wordPracticeMode = editingWordPracticeMode

  window.umami?.track('startStudyWord', {
    name: store.sdict.name,
    index: store.sdict.lastLearnIndex,
    perDayStudyNumber: store.sdict.perDayStudyNumber,
    custom: store.sdict.custom,
    complete: store.sdict.complete,
    wordPracticeMode: settingStore.wordPracticeMode,
  })

  const result = getShufflePracticeWords(store.sdict.words, setting, store.getIgnoreWordsSet())
  practiceData.taskWords.review = result.words
  nav(
    WordPracticeModeUrlMap[editingWordPracticeMode] + '/' + store.sdict.id,
    {},
    {
      ...practiceData,
      total: result.words.length,
      shuffleRange: result.range,
    }
  )
}

async function saveLastPracticeIndex(e) {
  runtimeStore.editDict.lastLearnIndex = e
  showChangeLastPracticeIndexDialog = false
  await resetCacheData()
  await store.changeDict(runtimeStore.editDict)
  refreshStudyTask()
  Toast.success('修改成功')
}

const { data: recommendDictList, isFetching } = useFetch(resourceWrap(DICT_LIST.WORD.RECOMMENDED)).json()

const systemPracticeText = $computed(() => {
  if (settingStore.wordPracticeMode === WordPracticeMode.Free) {
    return '开始学习'
  } else if (settingStore.wordPracticeMode === WordPracticeMode.Custom) {
    return isSaveData ? '继续自定义练习' : '开始自定义练习'
  } else {
    return isSaveData
      ? '继续' + WordPracticeModeNameMap[settingStore.wordPracticeMode]
      : '开始' + WordPracticeModeNameMap[settingStore.wordPracticeMode]
  }
})

let isOldHost = $ref(false)
onMounted(() => {
  isOldHost = window.location.host === Old_Host
})

onUnmounted(() => {
  document.removeEventListener('visibilitychange', onvisibilitychange)
})
</script>

<template>
  <BasePage>
    <div class="focus-dashboard">
      <div class="focus-old-host" v-if="isOldHost">
        已启用新域名：<a :href="`${Origin}/words?from_old_site=1`">{{ Origin }}</a>
      </div>

      <header class="focus-dashboard__header">
        <div>
          <p class="focus-dashboard__eyebrow">DAILY PRACTICE</p>
          <h1>{{ isSaveData ? $t('continue_learning') : $t('today_task') }}</h1>
          <p>把今天的任务做完，剩下的交给记忆曲线。</p>
        </div>
        <button
          v-if="store.sdict.id"
          type="button"
          class="focus-text-action"
          @click="showPracticeWordListDialog = true"
        >
          {{ $t('word_list') }}
          <IconFluentChevronRight16Regular />
        </button>
      </header>

      <section class="focus-task" :class="{ 'is-empty': !store.sdict.id }">
        <div class="focus-task__context">
          <div class="focus-task__book-icon">
            <IconFluentBookNumber20Filled />
          </div>
          <div class="focus-task__copy">
            <p class="focus-task__label">{{ $t('words') }}</p>
            <button type="button" class="focus-task__title" @click="goDictDetail(store.sdict)">
              {{ store.sdict.name || $t('no_dict_selected') }}
            </button>
            <template v-if="store.sdict.id">
              <p class="focus-task__meta">
                {{ $t('estimated_completion') }}
                {{ _getAccomplishDate(store.sdict.words.length - store.sdict.lastLearnIndex, store.sdict.perDayStudyNumber) }}
              </p>
              <div class="focus-task__progress">
                <Progress size="large" :percentage="store.currentStudyProgress" :show-text="false" />
                <div>
                  <span>{{ progressTextLeft }}</span>
                  <span>{{ store.sdict.lastLearnIndex }} / {{ store.sdict.length }} 词</span>
                </div>
              </div>
            </template>
            <p v-else class="focus-task__meta">{{ $t('select_dict_to_start') }}</p>
          </div>
        </div>

        <div v-if="store.sdict.id" class="focus-task__plan">
          <div class="focus-task__plan-head">
            <div>
              <p class="focus-task__label">{{ isSaveData ? $t('last_task') : $t('today_task') }}</p>
              <p class="focus-task__goal">
                {{ $t('daily_goal') }} <strong>{{ store.sdict.perDayStudyNumber }}</strong> {{ $t('words_count') }}
              </p>
            </div>
            <PopConfirm
              :disabled="!isSaveData"
              title="当前存在未完成的学习任务，修改会重新生成学习任务，是否继续？"
              @confirm="check(() => (showPracticeSettingDialog = true))"
            >
              <button type="button" class="focus-text-action">{{ $t('change') }}</button>
            </PopConfirm>
          </div>

          <div class="focus-task__counts">
            <div>
              <strong>{{ practiceData?.taskWords?.new?.length }}</strong>
              <span>{{ $t('new_words') }}</span>
            </div>
            <div>
              <strong>{{ practiceData?.taskWords?.review?.length }}</strong>
              <span class="focus-inline-label">
                {{ $t('review') }}
                <Tooltip>
                  <IconFluentQuestionCircle20Regular width="17" />
                  <template #reference><div class="whitespace-pre-wrap">{{ reviewWordTip }}</div></template>
                </Tooltip>
              </span>
            </div>
          </div>

          <div v-if="!isSaveData && dueReviewCount === 0" class="focus-task__random-review">
            <span>加入随机复习</span>
            <Switch :model-value="settingStore.autoAddRandomReviewWhenNoDue" @change="toggleAutoAddRandomReview" />
          </div>

          <div class="focus-task__actions btn-no-margin">
            <OptionButton class="focus-task__primary">
              <BaseButton
                size="large"
                :type="settingStore.wordPracticeMode !== WordPracticeMode.Free ? 'orange' : 'primary'"
                :loading="loading"
                @click="systemPractice"
              >
                <span class="focus-button-label">
                  {{ systemPracticeText }}
                  <IconFluentArrowRight16Regular />
                </span>
              </BaseButton>
              <template #options>
                <BaseButton
                  class="w-full"
                  v-if="settingStore.wordPracticeMode !== WordPracticeMode.System && settingStore.wordPracticeMode !== WordPracticeMode.Free"
                  @click="startPractice(WordPracticeMode.System, true)"
                >{{ $t('smart_learning') }}</BaseButton>
                <BaseButton
                  class="w-full"
                  v-if="settingStore.wordPracticeMode !== WordPracticeMode.Review"
                  :disabled="!practiceData?.taskWords?.review?.length"
                  @click="startPractice(WordPracticeMode.Review, true)"
                >{{ $t('review') }}</BaseButton>
                <BaseButton
                  class="w-full"
                  v-if="settingStore.wordPracticeMode !== WordPracticeMode.Shuffle"
                  :disabled="store.sdict.lastLearnIndex < 10 && !store.sdict.complete"
                  @click="startPractice(WordPracticeMode.Shuffle, true)"
                >{{ $t('random_review') }}</BaseButton>
                <BaseButton
                  class="w-full"
                  v-if="settingStore.wordPracticeMode !== WordPracticeMode.ReviewWordsTest"
                  :disabled="store.sdict.lastLearnIndex < 10 && !store.sdict.complete"
                  @click="startPractice(WordPracticeMode.ReviewWordsTest, true)"
                >{{ $t('words') }}{{ $t('test') }}</BaseButton>
                <BaseButton
                  class="w-full"
                  v-if="settingStore.wordPracticeMode !== WordPracticeMode.ShuffleWordsTest"
                  :disabled="store.sdict.lastLearnIndex < 10 && !store.sdict.complete"
                  @click="startPractice(WordPracticeMode.ShuffleWordsTest, true)"
                >{{ $t('random_words_test') }}</BaseButton>
              </template>
            </OptionButton>

            <BaseButton type="info" size="large" :loading="loading" @click="freePractice()">
              <span class="focus-button-label">
                {{ settingStore.wordPracticeMode === WordPracticeMode.Free && isSaveData ? $t('continue_free_practice') : $t('free_practice') }}
                <IconFluentPen20Regular />
              </span>
            </BaseButton>
          </div>
        </div>

        <div v-else class="focus-task__empty-action">
          <BaseButton id="step1" type="primary" size="large" @click="router.push('/dict-list')">
            <span class="focus-button-label"><IconFluentAdd16Regular />{{ $t('select_dict') }}</span>
          </BaseButton>
        </div>
      </section>

      <section class="focus-overview" aria-label="学习统计">
        <div class="focus-overview__metrics">
          <div class="focus-metric">
            <span>{{ $t('today_study_time') }}</span>
            <strong>{{ todayTotalSpend }}</strong>
          </div>
          <div class="focus-metric">
            <span>{{ $t('total_study_days') }}</span>
            <strong>{{ totalDay }}</strong>
          </div>
          <div class="focus-metric">
            <span>{{ $t('total_study_time') }}</span>
            <strong>{{ totalSpend }}</strong>
          </div>
        </div>
        <div class="focus-overview__calendar">
          <Calendar
            :highlighted-dates="calendarHighlightDates"
            @select-date="onSelectCalendarDate"
            :weekHeaderTitle="$t('this_week_record')"
          />
        </div>
      </section>

      <section class="focus-library">
        <div class="focus-section-head">
          <div>
            <p class="focus-dashboard__eyebrow">LIBRARY</p>
            <h2>{{ $t('my_dictionaries') }}</h2>
          </div>
          <div class="focus-section-actions">
            <PopConfirm title="确认删除所有选中词典？" @confirm="handleBatchDel" v-if="selectIds.length">
              <BaseIcon class="del" :title="$t('delete')"><DeleteIcon /></BaseIcon>
            </PopConfirm>
            <button
              v-if="store.word.bookList.length > 3"
              type="button"
              class="focus-text-action"
              @click="isManageDict = !isManageDict; selectIds = []"
            >{{ isManageDict ? $t('cancel') : $t('manage_dict') }}</button>
            <button type="button" class="focus-text-action" @click="nav('/dict', { isAdd: true })">
              {{ $t('create_personal_dict') }}
            </button>
          </div>
        </div>
        <div class="focus-book-grid">
          <Book
            v-for="(item, j) in store.word.bookList"
            :key="item.id"
            :is-add="false"
            quantifier="词"
            :item="item"
            :checked="selectIds.includes(item.id)"
            :show-checkbox="isManageDict && j >= 3"
            @check="() => toggleSelect(item)"
            @click="goDictDetail(item)"
          />
          <Book :is-add="true" @click="router.push('/dict-list')" />
        </div>
      </section>

      <section class="focus-library focus-library--quiet" v-loading="isFetching">
        <div class="focus-section-head">
          <div>
            <p class="focus-dashboard__eyebrow">DISCOVER</p>
            <h2>{{ $t('recommend') }}</h2>
          </div>
          <button type="button" class="focus-text-action" @click="router.push('/dict-list')">
            {{ $t('more') }} <IconFluentChevronRight16Regular />
          </button>
        </div>
        <div class="focus-book-grid focus-book-grid--recommend">
          <Book
            v-for="item in recommendDictList"
            :key="item.id"
            :is-add="false"
            quantifier="词"
            :item="item as any"
            @click="goDictDetail(item as any)"
          />
        </div>
      </section>
    </div>
  </BasePage>

  <PracticeSettingDialog
    :show-left-option="false"
    v-model="showPracticeSettingDialog"
    :onConfirm="savePracticeSetting"
  />

  <ChangeLastPracticeIndexDialog v-model="showChangeLastPracticeIndexDialog" @ok="saveLastPracticeIndex" />

  <PracticeWordListDialog :data="practiceData?.taskWords" v-model="showPracticeWordListDialog" />

  <ShufflePracticeSettingDialog
    v-model="showShufflePracticeSettingDialog"
    :onConfirm="onShufflePracticeSettingOk"
    :wordPracticeMode="editingWordPracticeMode"
  />

  <Dialog v-model="showStudyDayDialog" :title="studyDayDialogTitle" :footer="false" :padding="true">
    <div
      v-if="!studyDayRecords.length && !(isStudyDayKeyToday(selectedStudyDateKey) && todayCacheMs > 0)"
      class="text-gray-500 py-6 text-center"
    >
      当日无学习记录
    </div>
    <ul v-if="studyDayRecords.length" class="study-day-list max-h-70vh overflow-y-auto space-y-3">
      <li v-for="(row, idx) in studyDayRecords" :key="idx" class="border-b border-gray-200 pb-3 last:border-0">
        <div class="flex items-center gap-2">
          <span class="font-medium">{{ row.dictName }}</span>
          <span
            v-if="row.sessionRole && row.sessionRole !== 'single'"
            class="text-xs px-1.5 py-0.5 rounded-full"
            :class="{
              'bg-green-100 text-green-700': row.sessionRole === 'start',
              'bg-blue-100 text-blue-700': row.sessionRole === 'middle',
              'bg-orange-100 text-orange-700': row.sessionRole === 'end',
            }"
          >
            {{ { start: '学习开始', middle: '学习中', end: '学习结束' }[row.sessionRole] }}
          </span>
        </div>
        <div class="text-sm text-gray-600 mt-1">
          时长 {{ msToHourMinute(row.spend) }} · 新学 {{ row.new }} · 复习 {{ row.review }} · 错词 {{ row.wrong }}
          <template v-if="row.total"> · 共 {{ row.total }} 词</template>
        </div>
      </li>
    </ul>
  </Dialog>
</template>

<style scoped lang="scss">
.focus-dashboard {
  padding-bottom: 3rem;

  &__header,
  .focus-section-head {
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
  }

  &__header {
    margin-bottom: 1.5rem;

    h1 {
      margin: 0.125rem 0 0;
      color: var(--focus-ink);
      font-size: clamp(2rem, 4vw, 3.5rem);
      font-weight: 760;
      letter-spacing: -0.055em;
      line-height: 1.05;
    }

    p:last-child {
      margin: 0.75rem 0 0;
      color: var(--focus-ink-secondary);
    }
  }

  &__eyebrow {
    margin: 0;
    color: var(--focus-accent);
    font-size: 0.6875rem;
    font-weight: 760;
    letter-spacing: 0.14em;
  }
}

.focus-old-host {
  margin-bottom: 1rem;
  padding: 0.75rem 1rem;
  border: 1px solid var(--focus-warning);
  border-radius: var(--focus-radius-sm);
  color: var(--focus-warning);
  background: var(--focus-warning-soft);
}

.focus-text-action {
  min-height: 2.75rem;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.25rem;
  padding: 0 0.5rem;
  border: 0;
  color: var(--focus-accent);
  background: transparent;
  font: inherit;
  font-size: 0.875rem;
  font-weight: 650;
  cursor: pointer;
}

.focus-task {
  display: grid;
  grid-template-columns: minmax(0, 1.05fr) minmax(24rem, 0.95fr);
  gap: clamp(1.5rem, 4vw, 4rem);
  padding: clamp(1.5rem, 4vw, 3rem);
  border: 1px solid var(--focus-border);
  border-radius: var(--focus-radius-lg);
  background: var(--focus-surface);
  box-shadow: var(--focus-shadow-sm);

  &__context {
    min-width: 0;
    display: flex;
    gap: 1rem;
  }

  &__book-icon {
    width: 3rem;
    height: 3rem;
    flex: 0 0 auto;
    display: grid;
    place-items: center;
    border-radius: 0.875rem;
    color: var(--focus-accent);
    background: var(--focus-accent-soft);

    svg {
      width: 1.375rem;
      height: 1.375rem;
    }
  }

  &__copy {
    min-width: 0;
    flex: 1;
  }

  &__label {
    margin: 0;
    color: var(--focus-ink-tertiary);
    font-size: 0.75rem;
    font-weight: 720;
    letter-spacing: 0.08em;
    text-transform: uppercase;
  }

  &__title {
    display: block;
    max-width: 100%;
    margin-top: 0.25rem;
    padding: 0;
    border: 0;
    color: var(--focus-ink);
    background: transparent;
    font: inherit;
    font-size: clamp(1.5rem, 3vw, 2.25rem);
    font-weight: 740;
    letter-spacing: -0.035em;
    text-align: left;
    cursor: pointer;
  }

  &__meta {
    margin: 0.5rem 0 0;
    color: var(--focus-ink-secondary);
    font-size: 0.875rem;
  }

  &__progress {
    margin-top: 2rem;

    > div:last-child {
      display: flex;
      justify-content: space-between;
      gap: 1rem;
      margin-top: 0.625rem;
      color: var(--focus-ink-secondary);
      font-size: 0.8125rem;
    }
  }

  &__plan {
    padding-left: clamp(1.5rem, 4vw, 3rem);
    border-left: 1px solid var(--focus-border);
  }

  &__plan-head {
    display: flex;
    justify-content: space-between;
    gap: 1rem;
  }

  &__goal {
    margin: 0.25rem 0 0;
    color: var(--focus-ink-secondary);

    strong {
      color: var(--focus-ink);
      font-size: 1.25rem;
    }
  }

  &__counts {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    margin-top: 1.5rem;
    border-block: 1px solid var(--focus-border);

    > div {
      display: flex;
      flex-direction: column;
      gap: 0.125rem;
      padding: 1rem 0;

      & + div {
        padding-left: 1.5rem;
        border-left: 1px solid var(--focus-border);
      }
    }

    strong {
      color: var(--focus-ink);
      font-size: 2rem;
      font-weight: 720;
      letter-spacing: -0.04em;
    }

    span {
      color: var(--focus-ink-secondary);
      font-size: 0.8125rem;
    }
  }

  &__random-review {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    gap: 0.625rem;
    margin-top: 0.75rem;
    color: var(--focus-ink-secondary);
    font-size: 0.8125rem;
  }

  &__actions {
    display: flex;
    align-items: stretch;
    gap: 0.75rem;
    margin-top: 1.5rem;
  }

  &__primary {
    min-width: 0;
    flex: 1;

    :deep(.base-button) {
      width: 100%;
    }
  }

  &__empty-action {
    display: flex;
    align-items: center;
    justify-content: flex-end;
  }
}

.focus-button-label,
.focus-inline-label {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
}

.focus-overview {
  display: grid;
  grid-template-columns: minmax(0, 1fr) auto;
  gap: 2rem;
  align-items: center;
  margin-top: 1.25rem;
  padding: 1.25rem 0;
  border-bottom: 1px solid var(--focus-border);

  &__metrics {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
  }

  &__calendar {
    padding-left: 2rem;
    border-left: 1px solid var(--focus-border);
  }
}

.focus-metric {
  min-width: 0;
  display: flex;
  flex-direction: column-reverse;
  gap: 0.25rem;
  padding: 0.5rem 1.5rem;

  &:first-child {
    padding-left: 0;
  }

  & + & {
    border-left: 1px solid var(--focus-border);
  }

  span {
    color: var(--focus-ink-secondary);
    font-size: 0.8125rem;
  }

  strong {
    color: var(--focus-ink);
    font-size: clamp(1.25rem, 2.5vw, 1.75rem);
    font-weight: 700;
    letter-spacing: -0.035em;
  }
}

.focus-library {
  margin-top: 3rem;

  &--quiet {
    margin-top: 2.5rem;
    padding-top: 2.5rem;
    border-top: 1px solid var(--focus-border);
  }
}

.focus-section-head {
  h2 {
    margin: 0.125rem 0 0;
    color: var(--focus-ink);
    font-size: 1.5rem;
    font-weight: 720;
    letter-spacing: -0.03em;
  }
}

.focus-section-actions {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.focus-book-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(var(--book-width), 1fr));
  gap: 1rem;
  margin-top: 1.25rem;

  :deep(> div) {
    width: 100% !important;
  }

  :deep(.book) {
    width: 100% !important;
  }

  &--recommend {
    max-height: 31rem;
    overflow: hidden;
  }
}

@media (max-width: 1024px) {
  .focus-task {
    grid-template-columns: 1fr;

    &__plan {
      padding-top: 1.5rem;
      padding-left: 0;
      border-top: 1px solid var(--focus-border);
      border-left: 0;
    }
  }

  .focus-overview {
    grid-template-columns: 1fr;

    &__calendar {
      padding-top: 1.25rem;
      padding-left: 0;
      border-top: 1px solid var(--focus-border);
      border-left: 0;
    }
  }
}

@media (max-width: 640px) {
  .focus-dashboard {
    &__header {
      align-items: flex-start;

      h1 {
        font-size: 2.25rem;
      }

      p:last-child {
        max-width: 18rem;
      }
    }
  }

  .focus-task {
    padding: 1.25rem;

    &__context {
      flex-direction: column;
    }

    &__book-icon {
      width: 2.5rem;
      height: 2.5rem;
    }

    &__actions {
      flex-direction: column;
    }

    &__empty-action {
      justify-content: flex-start;
    }
  }

  .focus-overview__metrics {
    grid-template-columns: 1fr;
  }

  .focus-metric {
    padding: 0.75rem 0;

    & + & {
      border-top: 1px solid var(--focus-border);
      border-left: 0;
    }
  }

  .focus-section-head {
    align-items: flex-start !important;
  }

  .focus-section-actions {
    max-width: 11rem;
    flex-wrap: wrap;
    justify-content: flex-end;
  }

  .focus-book-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}
</style>
