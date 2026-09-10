<script setup lang="ts">
import { ref } from 'vue'
import { useI18n } from 'vue-i18n'
import type { UserScore } from '@/types'

const { t } = useI18n()
const kinokoUserid = ref(localStorage.getItem('kinokoExportUserid') || '')
const exportError = ref('')

const KINOKO_MAX_SCORES = 5000
const KINOKO_MAX_BYTES = 5 * 1024 * 1024

function getStoredScores(): UserScore[] | null {
  const scores = localStorage.getItem('taikoScoreData')
  if (!scores) return null

  try {
    const parsed = JSON.parse(scores)
    return Array.isArray(parsed) ? parsed as UserScore[] : null
  } catch {
    return null
  }
}

const scoreCount = getStoredScores()?.length || 0
const hasScores = ref(scoreCount > 0)

function downloadJson(content: string, filename: string) {
  const blob = new Blob([content], { type: 'application/json;charset=utf-8' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')

  link.href = url
  link.download = filename
  document.body.appendChild(link)
  link.click()
  link.remove()
  URL.revokeObjectURL(url)
}

function exportRawScores() {
  const scores = localStorage.getItem('taikoScoreData')
  if (!scores) return

  const date = new Date().toISOString().slice(0, 10)
  downloadJson(scores, `taiko-scores-${date}.json`)
}

function exportKinokoScores() {
  exportError.value = ''
  const userid = kinokoUserid.value.trim()
  if (!/^cm\d+$/.test(userid)) {
    exportError.value = t('scoreExport.kinokoInvalidUserid')
    return
  }

  const scores = getStoredScores()
  if (!scores) {
    exportError.value = t('scoreExport.exportDataError')
    return
  }

  const exportScores = scores
    .filter(score => Number.isInteger(score.level) && score.level >= 1 && score.level <= 5)
    .slice(0, KINOKO_MAX_SCORES)
  const payload = {
    playerInfo: { userid },
    scoreInfo: exportScores.map(score => ({
      song_no: score.id,
      level: score.level,
      high_score: score.score,
      best_score_rank: score.scoreRank,
      good_cnt: score.great,
      ok_cnt: score.good,
      ng_cnt: score.bad,
      pound_cnt: score.drumroll,
      combo_cnt: score.combo,
      stage_cnt: score.playCount,
      clear_cnt: score.clearCount,
      full_combo_cnt: score.fullcomboCount,
      dondaful_combo_cnt: score.perfectCount,
      highscore_datetime: score.updatedAt,
      update_datetime: score.updatedAt
    }))
  }
  const content = JSON.stringify(payload, null, 2)

  if (new TextEncoder().encode(content).byteLength > KINOKO_MAX_BYTES) {
    exportError.value = t('scoreExport.kinokoFileTooLarge')
    return
  }

  localStorage.setItem('kinokoExportUserid', userid)
  const date = new Date().toISOString().slice(0, 10)
  downloadJson(content, `kinoko-scores-${userid}-${date}.json`)
}

function clearExportError() {
  exportError.value = ''
}
</script>

<template>
  <section class="bg-white/70 shadow-sm backdrop-blur-xl my-8 border border-white/40 rounded-[24px] overflow-hidden">
    <header class="px-6 sm:px-8 py-6 border-black/5 border-b">
      <div class="flex sm:flex-row flex-col sm:justify-between sm:items-baseline gap-2">
        <h2 class="m-0 font-bold text-[#1D1D1F] text-xl">{{ t('scoreExport.title') }}</h2>
        <span v-if="hasScores" class="text-[#86868B] text-sm">
          {{ t('scoreExport.scoreCount', { count: scoreCount }) }}
        </span>
      </div>
      <p class="m-0 mt-2 text-[#86868B] text-sm leading-relaxed">{{ t('scoreExport.description') }}</p>
    </header>

    <div v-if="hasScores" class="grid md:grid-cols-2">
      <div class="flex flex-col gap-4 p-6 sm:p-8 md:border-black/5 md:border-r">
        <h3 class="m-0 font-semibold text-[#1D1D1F] text-base">{{ t('scoreExport.rawTitle') }}</h3>
        <p class="flex-1 m-0 text-[#6E6E73] text-sm leading-relaxed">{{ t('scoreExport.rawDescription') }}</p>
        <button
          class="bg-black/5 hover:bg-black/10 px-4 py-3 border-none rounded-xl w-full font-semibold text-[#1D1D1F] active:scale-[0.98] transition-all cursor-pointer"
          @click="exportRawScores"
        >
          {{ t('scoreExport.exportRaw') }}
        </button>
      </div>

      <div class="flex flex-col gap-4 p-6 sm:p-8 border-black/5 border-t md:border-t-0">
        <h3 class="m-0 font-semibold text-[#1D1D1F] text-base">{{ t('scoreExport.kinokoTitle') }}</h3>
        <p class="m-0 text-[#6E6E73] text-sm leading-relaxed">{{ t('scoreExport.kinokoDescription') }}</p>
        <div class="space-y-2">
          <label for="kinoko-export-userid" class="block font-semibold text-[#1D1D1F] text-sm">
            {{ t('scoreExport.kinokoUserid') }}
          </label>
          <input
            id="kinoko-export-userid"
            v-model="kinokoUserid"
            type="text"
            autocomplete="off"
            spellcheck="false"
            :placeholder="t('scoreExport.kinokoUseridPlaceholder')"
            class="box-border bg-black/5 focus:bg-white px-4 py-3 border border-transparent focus:border-[#007AFF]/30 rounded-xl outline-none focus:ring-[#007AFF]/10 focus:ring-4 w-full font-mono text-[#1D1D1F] transition-all"
            @input="clearExportError"
            @keyup.enter="exportKinokoScores"
          />
          <p class="m-0 text-[#86868B] text-xs leading-relaxed">{{ t('scoreExport.useridHint') }}</p>
          <p v-if="exportError" class="m-0 text-[#D70015] text-sm">{{ exportError }}</p>
        </div>
        <button
          class="bg-[#007AFF] hover:bg-[#0071E3] px-4 py-3 border-none rounded-xl w-full font-semibold text-white active:scale-[0.98] transition-all cursor-pointer"
          @click="exportKinokoScores"
        >
          {{ t('scoreExport.exportKinoko') }}
        </button>
      </div>
    </div>

    <div v-else class="px-6 py-10 text-center">
      <p class="m-0 font-semibold text-[#1D1D1F]">{{ t('scoreExport.noScoresTitle') }}</p>
      <p class="m-0 mt-1 text-[#86868B] text-sm">{{ t('scoreExport.noScores') }}</p>
    </div>
  </section>
</template>