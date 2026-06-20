<script setup lang="ts">
import { ref, computed } from 'vue'
import type { ScoreEvent, ScoreEventCategory } from '@/types/game'
import { WEATHER_NAMES } from '@/utils/constants'

const props = defineProps<{
  events: ScoreEvent[]
}>()

const isExpanded = ref(false)
const activeFilter = ref<ScoreEventCategory | 'all'>('all')

const CATEGORY_CONFIG: Record<ScoreEventCategory, { icon: string; label: string; color: string }> = {
  hatch: { icon: '🥚', label: '孵化', color: 'from-yellow-500/20 to-amber-500/20 border-yellow-400/30' },
  growth: { icon: '🌟', label: '成长', color: 'from-green-500/20 to-emerald-500/20 border-green-400/30' },
  feed: { icon: '🍒', label: '喂食', color: 'from-red-500/20 to-pink-500/20 border-red-400/30' },
  calm: { icon: '🤗', label: '安抚', color: 'from-purple-500/20 to-violet-500/20 border-purple-400/30' },
  death: { icon: '💔', label: '离世', color: 'from-gray-500/20 to-slate-500/20 border-gray-400/30' },
  sick: { icon: '🤒', label: '生病', color: 'from-orange-500/20 to-red-500/20 border-orange-400/30' },
  weather: { icon: '🌤️', label: '天气', color: 'from-blue-500/20 to-cyan-500/20 border-blue-400/30' },
  breed: { icon: '💝', label: '繁殖', color: 'from-pink-500/20 to-rose-500/20 border-pink-400/30' },
}

const filteredEvents = computed(() => {
  if (activeFilter.value === 'all') return props.events
  return props.events.filter(e => e.category === activeFilter.value)
})

const categoryStats = computed(() => {
  const stats: Record<string, number> = {}
  for (const cat of Object.keys(CATEGORY_CONFIG) as ScoreEventCategory[]) {
    stats[cat] = props.events.filter(e => e.category === cat).length
  }
  return stats
})

const scoreTrend = computed(() => {
  if (props.events.length < 2) return []
  const points: { day: number; score: number }[] = []
  for (const evt of props.events) {
    points.push({ day: evt.day, score: evt.snapshot.totalScore })
  }
  return points
})

const getPrevSnapshot = (index: number) => {
  if (index <= 0) return null
  return filteredEvents.value[index - 1]?.snapshot ?? null
}

const deltaText = (curr: number, prev: number | undefined) => {
  if (prev === undefined || prev === null) return ''
  const d = curr - prev
  if (d > 0) return `+${d}`
  if (d < 0) return `${d}`
  return ''
}

const deltaClass = (curr: number, prev: number | undefined) => {
  if (prev === undefined || prev === null) return ''
  const d = curr - prev
  if (d > 0) return 'text-green-400'
  if (d < 0) return 'text-red-400'
  return 'text-white/30'
}

const formatTime = (ts: number) => {
  const d = new Date(ts)
  return `${d.getHours().toString().padStart(2, '0')}:${d.getMinutes().toString().padStart(2, '0')}:${d.getSeconds().toString().padStart(2, '0')}`
}

const toggleExpand = () => {
  isExpanded.value = !isExpanded.value
}

const setFilter = (cat: ScoreEventCategory | 'all') => {
  activeFilter.value = cat
}

const allCategories = Object.keys(CATEGORY_CONFIG) as ScoreEventCategory[]
</script>

<template>
  <div class="score-replay">
    <button
      class="w-full flex items-center justify-between px-5 py-4 bg-gradient-to-r from-indigo-500/10 to-purple-500/10
             rounded-2xl border border-indigo-400/20 hover:border-indigo-400/40 transition-all"
      @click="toggleExpand"
    >
      <div class="flex items-center gap-3">
        <span class="text-2xl">📋</span>
        <div class="text-left">
          <div class="text-white/90 font-medium">评分明细回放</div>
          <div class="text-white/50 text-xs">追溯 {{ events.length }} 个关键决策对评分的影响</div>
        </div>
      </div>
      <span
        class="text-white/60 transition-transform duration-300"
        :class="isExpanded ? 'rotate-180' : ''"
      >▼</span>
    </button>

    <transition name="slide">
      <div v-if="isExpanded" class="mt-4 space-y-4">
        <div class="flex flex-wrap gap-2">
          <button
            class="px-3 py-1.5 rounded-xl text-xs font-medium transition-all border"
            :class="activeFilter === 'all'
              ? 'bg-white/20 border-white/30 text-white'
              : 'bg-white/5 border-white/10 text-white/50 hover:bg-white/10'"
            @click="setFilter('all')"
          >
            全部 ({{ events.length }})
          </button>
          <button
            v-for="cat in allCategories"
            :key="cat"
            v-show="categoryStats[cat] > 0"
            class="px-3 py-1.5 rounded-xl text-xs font-medium transition-all border"
            :class="activeFilter === cat
              ? 'bg-white/20 border-white/30 text-white'
              : 'bg-white/5 border-white/10 text-white/50 hover:bg-white/10'"
            @click="setFilter(cat)"
          >
            {{ CATEGORY_CONFIG[cat].icon }} {{ CATEGORY_CONFIG[cat].label }} ({{ categoryStats[cat] }})
          </button>
        </div>

        <div v-if="scoreTrend.length >= 2" class="bg-white/5 rounded-2xl p-4 border border-white/10">
          <div class="text-white/60 text-xs mb-3 flex items-center gap-1.5">
            <span>📈</span> 综合得分趋势
          </div>
          <div class="flex items-end gap-[2px] h-20">
            <div
              v-for="(point, idx) in scoreTrend"
              :key="idx"
              class="flex-1 rounded-t-sm transition-all min-w-[3px]"
              :style="{
                height: Math.max(4, point.score) + '%',
                background: point.score >= 60
                  ? 'linear-gradient(to top, #22c55e, #4ade80)'
                  : point.score >= 30
                    ? 'linear-gradient(to top, #eab308, #facc15)'
                    : 'linear-gradient(to top, #ef4444, #f87171)',
              }"
              :title="`第${point.day}天: ${point.score}分`"
            />
          </div>
          <div class="flex justify-between text-white/30 text-[10px] mt-1">
            <span>第{{ scoreTrend[0]?.day }}天</span>
            <span>第{{ scoreTrend[scoreTrend.length - 1]?.day }}天</span>
          </div>
        </div>

        <div class="space-y-2 max-h-[400px] overflow-y-auto scrollbar-hide pr-1">
          <div
            v-for="(event, idx) in filteredEvents"
            :key="event.id"
            class="bg-gradient-to-r rounded-2xl p-4 border transition-all hover:scale-[1.01]"
            :class="CATEGORY_CONFIG[event.category].color"
          >
            <div class="flex items-start gap-3">
              <div class="text-2xl mt-0.5 shrink-0">{{ CATEGORY_CONFIG[event.category].icon }}</div>
              <div class="flex-1 min-w-0">
                <div class="flex items-center gap-2 mb-1">
                  <span class="text-white/90 text-sm font-medium truncate">{{ event.message }}</span>
                </div>
                <div class="flex items-center gap-3 text-white/40 text-xs">
                  <span>📅 第{{ event.day }}天</span>
                  <span>🕐 {{ formatTime(event.timestamp) }}</span>
                  <span
                    class="px-1.5 py-0.5 rounded-md text-[10px] font-medium"
                    :class="CATEGORY_CONFIG[event.category].color"
                  >{{ CATEGORY_CONFIG[event.category].label }}</span>
                </div>

                <div class="mt-2 grid grid-cols-4 gap-2">
                  <div class="text-center">
                    <div class="text-white/40 text-[10px]">成活率</div>
                    <div class="text-sm font-bold" :class="deltaClass(event.snapshot.survivalRate, getPrevSnapshot(idx)?.survivalRate)">
                      {{ event.snapshot.survivalRate }}%
                      <span v-if="deltaText(event.snapshot.survivalRate, getPrevSnapshot(idx)?.survivalRate)" class="text-[10px]">
                        {{ deltaText(event.snapshot.survivalRate, getPrevSnapshot(idx)?.survivalRate) }}
                      </span>
                    </div>
                  </div>
                  <div class="text-center">
                    <div class="text-white/40 text-[10px]">健康</div>
                    <div class="text-sm font-bold" :class="deltaClass(event.snapshot.avgHealth, getPrevSnapshot(idx)?.avgHealth)">
                      {{ event.snapshot.avgHealth }}
                      <span v-if="deltaText(event.snapshot.avgHealth, getPrevSnapshot(idx)?.avgHealth)" class="text-[10px]">
                        {{ deltaText(event.snapshot.avgHealth, getPrevSnapshot(idx)?.avgHealth) }}
                      </span>
                    </div>
                  </div>
                  <div class="text-center">
                    <div class="text-white/40 text-[10px]">繁殖</div>
                    <div class="text-sm font-bold text-pink-400">
                      +{{ event.snapshot.breedingBonus }}
                      <span v-if="getPrevSnapshot(idx) && event.snapshot.breedingBonus > getPrevSnapshot(idx)!.breedingBonus" class="text-[10px] text-green-400">
                        +{{ event.snapshot.breedingBonus - getPrevSnapshot(idx)!.breedingBonus }}
                      </span>
                    </div>
                  </div>
                  <div class="text-center">
                    <div class="text-white/40 text-[10px]">照料</div>
                    <div class="text-sm font-bold text-amber-400">
                      +{{ event.snapshot.personalityBonus }}
                      <span v-if="getPrevSnapshot(idx) && event.snapshot.personalityBonus > getPrevSnapshot(idx)!.personalityBonus" class="text-[10px] text-green-400">
                        +{{ event.snapshot.personalityBonus - getPrevSnapshot(idx)!.personalityBonus }}
                      </span>
                    </div>
                  </div>
                </div>

                <div class="mt-2 flex items-center gap-2">
                  <div class="flex-1 h-1.5 bg-white/5 rounded-full overflow-hidden">
                    <div
                      class="h-full rounded-full transition-all duration-500"
                      :class="event.snapshot.totalScore >= 60
                        ? 'progress-gradient-good'
                        : event.snapshot.totalScore >= 30
                          ? 'progress-gradient-mid'
                          : 'progress-gradient-bad'"
                      :style="{ width: event.snapshot.totalScore + '%' }"
                    />
                  </div>
                  <span class="text-xs font-bold" :class="event.snapshot.totalScore >= 60 ? 'text-green-400' : event.snapshot.totalScore >= 30 ? 'text-yellow-400' : 'text-red-400'">
                    {{ event.snapshot.totalScore }}
                  </span>
                </div>
              </div>
            </div>
          </div>

          <div v-if="filteredEvents.length === 0" class="text-center text-white/30 py-8 text-sm">
            暂无此类事件记录
          </div>
        </div>
      </div>
    </transition>
  </div>
</template>

<style scoped>
.slide-enter-active,
.slide-leave-active {
  transition: all 0.3s ease;
  overflow: hidden;
}

.slide-enter-from,
.slide-leave-to {
  opacity: 0;
  max-height: 0;
  transform: translateY(-8px);
}

.slide-enter-to,
.slide-leave-from {
  opacity: 1;
  max-height: 2000px;
  transform: translateY(0);
}
</style>
