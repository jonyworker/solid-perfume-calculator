<template>
  <div class="min-h-screen bg-slate-100/60 p-6 text-slate-950">
    <div class="mx-auto max-w-7xl space-y-6">
      <div class="space-y-1">
        <h1 class="text-3xl font-semibold tracking-tight">固態香水製作計算機</h1>
        <p class="text-sm text-slate-500">
          可設定材料價格、製作比例、容器數量與單罐克數，並依不可拆賣規則自動估算採購成本。
        </p>
      </div>

      <div class="grid gap-6 xl:grid-cols-[1.15fr_0.85fr]">
        <section class="space-y-6">
          <div class="rounded-xl border border-slate-200 bg-white shadow-sm">
            <div class="flex items-center justify-between border-b border-slate-100 px-6 py-4">
              <h2 class="text-base font-semibold tracking-tight">基本設定</h2>
              <button
                class="inline-flex h-9 items-center justify-center rounded-md border border-slate-200 bg-white px-4 text-sm font-medium transition-colors hover:bg-slate-50 focus:outline-none focus:ring-2 focus:ring-slate-950/10"
                @click="resetDefaults"
              >
                還原預設
              </button>
            </div>

            <div class="grid gap-4 p-6 md:grid-cols-3">
              <label class="rounded-lg border border-slate-200 bg-white p-4">
                <span class="text-sm font-medium text-slate-600">容器數量</span>
                <div class="mt-3 flex items-center gap-3 rounded-md border border-slate-200 bg-white px-3 py-2.5 transition-colors focus-within:border-slate-300 focus-within:ring-2 focus-within:ring-slate-950/5">
                  <input
                    v-model.number="settings.containerCount"
                    type="number"
                    min="1"
                    class="w-full bg-transparent text-2xl font-semibold outline-none"
                  />
                  <span class="text-sm font-medium text-slate-500">個</span>
                </div>
              </label>

              <label class="rounded-lg border border-slate-200 bg-white p-4">
                <span class="text-sm font-medium text-slate-600">每個容器內容物容量</span>
                <div class="mt-3 flex items-center gap-3 rounded-md border border-slate-200 bg-white px-3 py-2.5 transition-colors focus-within:border-slate-300 focus-within:ring-2 focus-within:ring-slate-950/5">
                  <input
                    v-model.number="settings.gramsPerContainer"
                    type="number"
                    min="0.1"
                    step="0.1"
                    class="w-full bg-transparent text-2xl font-semibold outline-none"
                  />
                  <span class="text-sm font-medium text-slate-500">g</span>
                </div>
              </label>

              <label class="rounded-lg border border-slate-200 bg-white p-4">
                <span class="text-sm font-medium text-slate-600">每個容器成本</span>
                <div class="mt-3 flex items-center gap-3 rounded-md border border-slate-200 bg-white px-3 py-2.5 transition-colors focus-within:border-slate-300 focus-within:ring-2 focus-within:ring-slate-950/5">
                  <input
                    v-model.number="settings.containerUnitCost"
                    type="number"
                    min="0"
                    step="0.1"
                    class="w-full bg-transparent text-2xl font-semibold outline-none"
                  />
                  <span class="text-sm font-medium text-slate-500">元</span>
                </div>
              </label>
            </div>
          </div>

          <div class="rounded-xl border border-slate-200 bg-white shadow-sm">
            <div class="border-b border-slate-100 px-6 py-4">
              <div class="flex items-center justify-between gap-4">
                <h2 class="text-base font-semibold tracking-tight">製作比例 (%)</h2>
                <div class="text-sm text-slate-500">
                  比例總和：
                  <span class="font-semibold" :class="ratioTotal === 100 ? 'text-emerald-600' : 'text-rose-600'">
                    {{ formatNumber(ratioTotal) }}%
                  </span>
                </div>
              </div>

              <div
                v-if="ratioError"
                class="mt-4 rounded-lg border border-rose-200 bg-rose-50/80 px-4 py-3 text-sm text-rose-700"
              >
                {{ ratioError }}
              </div>

              <div class="mt-4 overflow-hidden rounded-full bg-slate-100">
                <div class="flex h-2.5 w-full">
                  <div
                    v-for="segment in ratioSegments"
                    :key="segment.key"
                    class="h-full transition-all"
                    :class="segment.barClass"
                    :style="{ width: `${segment.width}%` }"
                  ></div>
                  <div
                    v-if="ratioGap > 0"
                    class="h-full bg-rose-300/70"
                    :style="{ width: `${ratioGap}%` }"
                  ></div>
                </div>
              </div>
            </div>

            <div class="grid gap-4 p-6 md:grid-cols-2">
              <label
                v-for="material in materials"
                :key="material.key"
                class="rounded-lg border border-slate-200 bg-white p-4"
              >
                <div class="flex items-center justify-start gap-3">
                  <span class="text-sm font-medium">{{ material.label }}</span>
                </div>
                <div class="mt-3 flex items-center gap-3 rounded-md border border-slate-200 bg-white px-3 py-2.5 transition-colors focus-within:border-slate-300 focus-within:ring-2 focus-within:ring-slate-950/5">
                  <input
                    v-model.number="ratios[material.key]"
                    type="number"
                    min="0"
                    step="0.1"
                    class="w-full bg-transparent text-xl font-semibold outline-none"
                  />
                  <span class="text-sm font-medium text-slate-500">%</span>
                </div>
              </label>
            </div>
          </div>

          <div class="rounded-xl border border-slate-200 bg-white shadow-sm">
            <button
              type="button"
              class="flex w-full items-center justify-between gap-4 px-6 py-4 text-left"
              @click="isPriceSettingsOpen = !isPriceSettingsOpen"
            >
              <div>
                <h2 class="text-base font-semibold tracking-tight">原料價格設定</h2>
                <p class="mt-1 text-sm text-slate-500">
                  {{ materials.length }} 種原料已設定，{{ totalPackOptionCount }} 個採購規格
                </p>
              </div>

              <span
                class="inline-flex h-9 w-9 items-center justify-center rounded-md border border-slate-200 bg-slate-50 text-slate-600 transition-transform duration-200"
                :class="isPriceSettingsOpen ? 'rotate-180' : ''"
              >
                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" class="h-4 w-4">
                  <path stroke-linecap="round" stroke-linejoin="round" d="m6 9 6 6 6-6" />
                </svg>
              </span>
            </button>

            <div v-if="isPriceSettingsOpen" class="space-y-4 border-t border-slate-100 p-6">
              <div
                v-for="material in materials"
                :key="material.key"
                class="overflow-hidden rounded-lg border border-slate-200 bg-white"
              >
                <button
                  type="button"
                  class="flex w-full items-center justify-between gap-4 px-4 py-4 text-left hover:bg-slate-50/80"
                  @click="toggleMaterialSection(material.key)"
                >
                  <div>
                    <h3 class="text-sm font-semibold">{{ material.label }}</h3>
                    <p class="mt-1 text-xs text-slate-500">
                      {{ packOptions[material.key].length }} 個規格，{{ material.unitLabel }} 為採購單位
                    </p>
                  </div>

                  <span
                    class="inline-flex h-8 w-8 items-center justify-center rounded-md border border-slate-200 bg-slate-50 text-slate-600 transition-transform duration-200"
                    :class="openMaterialSections[material.key] ? 'rotate-180' : ''"
                  >
                    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" class="h-4 w-4">
                      <path stroke-linecap="round" stroke-linejoin="round" d="m6 9 6 6 6-6" />
                    </svg>
                  </span>
                </button>

                <div v-if="openMaterialSections[material.key]" class="border-t border-slate-100 p-4">
                  <div class="grid gap-3 md:grid-cols-3">
                    <div
                      v-for="(pack, index) in packOptions[material.key]"
                      :key="`${material.key}-${index}`"
                      class="rounded-lg border border-slate-200 bg-slate-50/50 p-3"
                    >
                      <div class="space-y-2">
                        <label class="block space-y-1">
                          <span class="text-xs text-slate-500">規格 ({{ material.unitLabel }})</span>
                          <input
                            v-model.number="pack.size"
                            type="number"
                            min="0.1"
                            step="0.1"
                            class="w-full rounded-md border border-slate-200 bg-white px-3 py-2 text-sm outline-none transition-colors focus:border-slate-300 focus:ring-2 focus:ring-slate-950/5"
                          />
                        </label>

                        <label class="block space-y-1">
                          <span class="text-xs text-slate-500">價格 (元)</span>
                          <input
                            v-model.number="pack.price"
                            type="number"
                            min="0"
                            step="0.1"
                            class="w-full rounded-md border border-slate-200 bg-white px-3 py-2 text-sm outline-none transition-colors focus:border-slate-300 focus:ring-2 focus:ring-slate-950/5"
                          />
                        </label>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </section>

        <aside class="space-y-6 xl:sticky xl:top-6 self-start">
          <div class="rounded-xl border border-slate-200 bg-slate-950 text-white shadow-sm">
            <div class="p-6">
              <div class="text-sm text-slate-400">單罐成本</div>
              <div class="mt-3 text-4xl font-semibold tracking-tight">NT$ {{ formatCurrency(costPerContainer) }}</div>
              <div class="mt-4 rounded-lg border border-white/10 bg-white/5">
                <button
                  type="button"
                  class="flex w-full items-center justify-between gap-4 px-4 py-3 text-left"
                  @click="isUnitCostBreakdownOpen = !isUnitCostBreakdownOpen"
                >
                  <div>
                    <div class="text-sm font-medium text-white">單罐成本明細</div>
                    <div class="mt-1 text-xs text-slate-400">查看每個成分分攤到單罐的成本</div>
                  </div>

                  <span
                    class="inline-flex h-8 w-8 items-center justify-center rounded-md border border-white/10 bg-white/5 text-slate-300 transition-transform duration-200"
                    :class="isUnitCostBreakdownOpen ? 'rotate-180' : ''"
                  >
                    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" class="h-4 w-4">
                      <path stroke-linecap="round" stroke-linejoin="round" d="m6 9 6 6 6-6" />
                    </svg>
                  </span>
                </button>

                <div v-if="isUnitCostBreakdownOpen" class="space-y-3 border-t border-white/10 px-4 py-4">
                  <div
                    v-for="item in unitCostBreakdown"
                    :key="`unit-cost-${item.key}`"
                    class="rounded-md border border-white/10 bg-black/10 px-3 py-3"
                  >
                    <div class="flex items-center justify-between gap-3">
                      <div>
                        <div class="text-sm font-medium text-white">{{ item.label }}</div>
                        <div class="mt-1 text-xs text-slate-400">
                          {{ formatNumber(item.requiredAmountPerContainer) }} {{ item.unitLabel }} / 單罐
                        </div>
                      </div>
                      <div class="text-right">
                        <div class="text-xs text-slate-400">單罐分攤</div>
                        <div class="text-sm font-semibold text-white">NT$ {{ formatCurrency(item.costPerContainer) }}</div>
                      </div>
                    </div>
                  </div>

                  <div class="flex items-center justify-between border-t border-white/10 pt-3 text-sm">
                    <span class="text-slate-300">原料單罐成本合計</span>
                    <span class="font-semibold text-white">NT$ {{ formatCurrency(unitMaterialCostPerContainer) }}</span>
                  </div>
                  <div class="flex items-center justify-between text-sm">
                    <span class="text-slate-300">容器單罐成本</span>
                    <span class="font-semibold text-white">NT$ {{ formatCurrency(settings.containerUnitCost) }}</span>
                  </div>
                </div>
              </div>
              <div class="mt-4 grid gap-3 sm:grid-cols-2 xl:grid-cols-1 2xl:grid-cols-2">
                <div class="rounded-lg border border-white/10 bg-white/5 px-4 py-3">
                  <div class="text-xs text-slate-400">總成本</div>
                  <div class="mt-1 text-lg font-semibold">NT$ {{ formatCurrency(grandTotalCost) }}</div>
                </div>
                <div class="rounded-lg border border-white/10 bg-white/5 px-4 py-3">
                  <div class="text-xs text-slate-400">總產量</div>
                  <div class="mt-1 text-lg font-semibold">{{ formatNumber(totalWeight) }} g</div>
                </div>
              </div>


            </div>
          </div>

          <div class="rounded-xl border border-slate-200 bg-white shadow-sm">
            <div class="border-b border-slate-100 px-6 py-4">
              <h2 class="text-base font-semibold tracking-tight">成本摘要</h2>
            </div>

            <div class="space-y-4 p-6 text-sm">
              <div class="flex items-center justify-between">
                <span class="text-slate-600">容器成本</span>
                <span class="font-semibold">NT$ {{ formatCurrency(containerCost) }}</span>
              </div>
              <div>
                <div class="flex items-center justify-between">
                  <span class="text-slate-600">原料成本</span>
                  <span class="font-semibold">NT$ {{ formatCurrency(totalMaterialCost) }}</span>
                </div>

                <div class="mt-3 space-y-1 rounded-lg border border-slate-200 bg-slate-50/70 px-3 py-2 text-xs text-slate-600">
                  <div
                    v-for="item in materialResults"
                    :key="`summary-${item.key}`"
                    class="flex items-center justify-between gap-3"
                  >
                    <span>{{ item.label }}</span>
                    <span class="font-medium text-slate-700">
                      {{ formatNumber(item.requiredAmount) }} {{ item.unitLabel }}
                    </span>
                  </div>
                </div>

                <div class="mt-4 space-y-3">
                  <div v-for="item in materialResults" :key="item.key" class="text-xs">
                    <div class="mb-1.5 flex justify-between gap-3 text-slate-600">
                      <span>{{ item.label }}</span>
                      <span>{{ materialCostShare(item.purchaseCost) }}%</span>
                    </div>
                    <div class="h-2 overflow-hidden rounded-full bg-slate-100">
                      <div
                        class="h-full rounded-full transition-all"
                        :class="item.barClass"
                        :style="{ width: `${materialCostShare(item.purchaseCost)}%` }"
                      ></div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div class="rounded-xl border border-slate-200 bg-white shadow-sm">
            <div class="border-b border-slate-100 px-6 py-4">
              <h2 class="text-base font-semibold tracking-tight">採購明細</h2>
            </div>

            <div class="space-y-4 p-6">
              <div
                v-for="item in materialResults"
                :key="item.key"
                class="rounded-lg border border-slate-200 bg-white p-4"
              >
                <div class="mb-3 flex items-start justify-between gap-4">
                  <div>
                    <div class="flex items-center gap-2">
                      <h3 class="text-sm font-semibold">{{ item.label }}</h3>
                      <span
                        v-if="item.isHighestCost"
                        class="inline-flex items-center rounded-md border border-slate-200 bg-slate-100 px-2 py-0.5 text-[11px] font-medium text-slate-700"
                      >
                        成本最高
                      </span>
                      <span
                        v-else-if="item.leftoverWarning"
                        class="inline-flex items-center rounded-md border border-amber-200 bg-amber-50 px-2 py-0.5 text-[11px] font-medium text-amber-700"
                      >
                        剩餘偏高
                      </span>
                      <span
                        v-else-if="item.unitLabel === 'ml'"
                        class="inline-flex items-center rounded-md border border-slate-200 bg-slate-50 px-2 py-0.5 text-[11px] font-medium text-slate-700"
                      >
                        已換算
                      </span>
                    </div>
                    <p class="mt-1 text-xs text-slate-500">
                      需求：{{ formatNumber(item.requiredAmount) }} {{ item.unitLabel }}
                      <span class="ml-1 text-[11px] text-slate-400">
                        ({{ item.formula }})
                      </span>
                    </p>
                  </div>
                  <div class="text-right">
                    <div class="text-sm text-slate-500">採購成本</div>
                    <div class="font-semibold">NT$ {{ formatCurrency(item.purchaseCost) }}</div>
                  </div>
                </div>

                <div class="space-y-2 text-sm">
                  <div class="flex items-center justify-between">
                    <span class="text-slate-600">實際買入</span>
                    <span class="font-medium">
                      {{ formatNumber(item.purchasedAmount) }} {{ item.unitLabel }}
                    </span>
                  </div>
                  <div class="flex items-center justify-between">
                    <span class="text-slate-600">剩餘</span>
                    <span class="font-medium" :class="item.leftoverWarning ? 'text-amber-700' : ''">
                      {{ formatNumber(item.leftoverAmount) }} {{ item.unitLabel }}
                    </span>
                  </div>
                </div>

                <div class="mt-3">
                  <div class="mb-2 text-xs font-medium text-slate-500">購買組合</div>
                  <div class="flex flex-wrap gap-2">
                    <span
                      v-for="combo in item.combination"
                      :key="`${item.key}-${combo.size}`"
                      class="inline-flex items-center rounded-md border border-slate-200 bg-slate-50 px-2.5 py-1 text-xs font-medium text-slate-700"
                    >
                      {{ combo.count }} × {{ combo.size }}{{ item.unitLabel }}
                    </span>
                    <span
                      v-if="item.combination.length === 0"
                      class="inline-flex items-center rounded-md border border-slate-200 bg-slate-50 px-2.5 py-1 text-xs font-medium text-slate-700"
                    >
                      無需購買
                    </span>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </aside>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, reactive, ref } from 'vue'

const defaultState = {
  settings: {
    containerCount: 300,
    gramsPerContainer: 15,
    containerUnitCost: 5,
  },
  ratios: {
    sheaButter: 62,
    jojobaOil: 5,
    beeswax: 28,
    essentialOil: 5,
  },
  packOptions: {
    sheaButter: [
      { size: 250, price: 90 },
      { size: 500, price: 160 },
      { size: 1000, price: 290 },
    ],
    jojobaOil: [
      { size: 250, price: 260 },
      { size: 500, price: 490 },
      { size: 1000, price: 950 },
    ],
    beeswax: [
      { size: 250, price: 124 },
      { size: 500, price: 214 },
      { size: 1000, price: 380 },
    ],
    essentialOil: [
      { size: 15, price: 1280 },
    ],
  },
}

const settings = reactive(structuredClone(defaultState.settings))
const ratios = reactive(structuredClone(defaultState.ratios))
const packOptions = reactive(structuredClone(defaultState.packOptions))
const isPriceSettingsOpen = ref(false)
const isUnitCostBreakdownOpen = ref(false)
const openMaterialSections = reactive({
  sheaButter: true,
  jojobaOil: false,
  beeswax: false,
  essentialOil: false,
})

const materials = [
  {
    key: 'sheaButter',
    label: '乳油木果油',
    unitLabel: 'g',
    density: 1,
    badgeClass: 'border-slate-200 bg-slate-50 text-slate-700',
    barClass: 'bg-slate-900',
  },
  {
    key: 'jojobaOil',
    label: '荷荷芭油',
    unitLabel: 'g',
    density: 1,
    badgeClass: 'border-slate-200 bg-slate-50 text-slate-700',
    barClass: 'bg-slate-700',
  },
  {
    key: 'beeswax',
    label: '蜂蠟',
    unitLabel: 'g',
    density: 1,
    badgeClass: 'border-slate-200 bg-slate-50 text-slate-700',
    barClass: 'bg-slate-500',
  },
  {
    key: 'essentialOil',
    label: '精油',
    unitLabel: 'ml',
    density: 0.9,
    badgeClass: 'border-slate-200 bg-slate-50 text-slate-700',
    barClass: 'bg-slate-300',
  },
]

const totalPackOptionCount = computed(() => {
  return Object.values(packOptions).reduce((sum, packs) => sum + packs.length, 0)
})

const ratioTotal = computed(() => {
  return Object.values(ratios).reduce((sum, value) => sum + Number(value || 0), 0)
})

const ratioGap = computed(() => {
  return Math.max(0, 100 - Math.min(ratioTotal.value, 100))
})

const ratioSegments = computed(() => {
  return materials.map((material) => ({
    key: material.key,
    width: Math.max(0, Math.min(Number(ratios[material.key] || 0), 100)),
    barClass: material.barClass,
  }))
})

const ratioError = computed(() => {
  if (ratioTotal.value > 100) {
    return `比例總和不可超過 100%，目前為 ${formatNumber(ratioTotal.value)}%。`
  }

  if (ratioTotal.value < 100) {
    return `比例總和必須等於 100%，目前為 ${formatNumber(ratioTotal.value)}%。`
  }

  return ''
})

const totalWeight = computed(() => {
  return Number(settings.containerCount || 0) * Number(settings.gramsPerContainer || 0)
})

function normalizePackOptions(options) {
  return [...options]
    .map((item) => ({
      size: Number(item.size || 0),
      price: Number(item.price || 0),
    }))
    .filter((item) => item.size > 0 && item.price >= 0)
    .sort((a, b) => a.size - b.size)
}

function findBestCombination(requiredAmount, options) {
  const normalizedRequired = Math.max(0, Math.ceil(requiredAmount))

  if (normalizedRequired <= 0 || options.length === 0) {
    return {
      totalAmount: 0,
      totalCost: 0,
      combination: [],
    }
  }

  const smallestPack = options[0]?.size || 1
  const largestPack = options[options.length - 1]?.size || 1
  const searchLimit = normalizedRequired + Math.max(largestPack * 3, smallestPack * 5)

  const dp = Array(searchLimit + 1).fill(null)
  dp[0] = {
    cost: 0,
    counts: Array(options.length).fill(0),
  }

  for (let amount = 0; amount <= searchLimit; amount += 1) {
    if (!dp[amount]) continue

    options.forEach((pack, index) => {
      const nextAmount = amount + pack.size
      if (nextAmount > searchLimit) return

      const nextCost = dp[amount].cost + pack.price
      const nextCounts = [...dp[amount].counts]
      nextCounts[index] += 1

      const existing = dp[nextAmount]
      const nextPackCount = nextCounts.reduce((sum, count) => sum + count, 0)
      const existingPackCount = existing
        ? existing.counts.reduce((sum, count) => sum + count, 0)
        : Infinity

      const shouldReplace =
        !existing ||
        nextCost < existing.cost ||
        (nextCost === existing.cost && nextPackCount < existingPackCount)

      if (shouldReplace) {
        dp[nextAmount] = {
          cost: nextCost,
          counts: nextCounts,
        }
      }
    })
  }

  let bestAmount = -1
  let bestState = null

  for (let amount = normalizedRequired; amount <= searchLimit; amount += 1) {
    const state = dp[amount]
    if (!state) continue

    if (!bestState) {
      bestAmount = amount
      bestState = state
      continue
    }

    const currentLeftover = amount - normalizedRequired
    const bestLeftover = bestAmount - normalizedRequired
    const currentPackCount = state.counts.reduce((sum, count) => sum + count, 0)
    const bestPackCount = bestState.counts.reduce((sum, count) => sum + count, 0)

    const better =
      state.cost < bestState.cost ||
      (state.cost === bestState.cost && currentLeftover < bestLeftover) ||
      (state.cost === bestState.cost && currentLeftover === bestLeftover && currentPackCount < bestPackCount)

    if (better) {
      bestAmount = amount
      bestState = state
    }
  }

  if (!bestState) {
    return {
      totalAmount: 0,
      totalCost: 0,
      combination: [],
    }
  }

  const combination = bestState.counts
    .map((count, index) => ({
      count,
      size: options[index].size,
      price: options[index].price,
    }))
    .filter((item) => item.count > 0)
    .sort((a, b) => b.size - a.size)

  return {
    totalAmount: bestAmount,
    totalCost: bestState.cost,
    combination,
  }
}

const materialResults = computed(() => {
  const rows = materials.map((material) => {
    const ratio = Number(ratios[material.key] || 0) / 100
    const requiredGrams = totalWeight.value * ratio
    const requiredAmount = material.unitLabel === 'ml'
      ? requiredGrams / material.density
      : requiredGrams

    const options = normalizePackOptions(packOptions[material.key])
    const purchase = ratioError.value
      ? { totalAmount: 0, totalCost: 0, combination: [] }
      : findBestCombination(requiredAmount, options)

    const ratioPercent = ratio * 100
    const baseFormula = `${formatNumber(totalWeight.value)}g × ${formatNumber(ratioPercent)}%`
    const formula = material.unitLabel === 'ml'
      ? `${baseFormula} → ${formatNumber(requiredGrams)}g ÷ ${material.density}`
      : baseFormula

    return {
      key: material.key,
      label: material.label,
      unitLabel: material.unitLabel,
      barClass: material.barClass,
      requiredAmount,
      formula,
      purchasedAmount: purchase.totalAmount,
      purchaseCost: purchase.totalCost,
      leftoverAmount: Math.max(0, purchase.totalAmount - requiredAmount),
      leftoverWarning: purchase.totalAmount - requiredAmount > requiredAmount * 0.1,
      combination: purchase.combination,
    }
  })

  const highestCost = rows.reduce((max, item) => Math.max(max, item.purchaseCost), 0)

  return rows.map((item) => ({
    ...item,
    isHighestCost: highestCost > 0 && item.purchaseCost === highestCost,
  }))
})

const totalMaterialCost = computed(() => {
  if (ratioError.value) return 0
  return materialResults.value.reduce((sum, item) => sum + item.purchaseCost, 0)
})

const containerCost = computed(() => {
  return Number(settings.containerCount || 0) * Number(settings.containerUnitCost || 0)
})

const grandTotalCost = computed(() => {
  return totalMaterialCost.value + containerCost.value
})

const costPerContainer = computed(() => {
  const count = Number(settings.containerCount || 0)
  if (count <= 0) return 0
  return grandTotalCost.value / count
})

const unitCostBreakdown = computed(() => {
  const count = Number(settings.containerCount || 0)

  if (count <= 0) {
    return materialResults.value.map((item) => ({
      ...item,
      requiredAmountPerContainer: 0,
      costPerContainer: 0,
    }))
  }

  return materialResults.value.map((item) => ({
    ...item,
    requiredAmountPerContainer: item.requiredAmount / count,
    costPerContainer: item.purchaseCost / count,
  }))
})

const unitMaterialCostPerContainer = computed(() => {
  const count = Number(settings.containerCount || 0)
  if (count <= 0) return 0
  return totalMaterialCost.value / count
})

function toggleMaterialSection(key) {
  openMaterialSections[key] = !openMaterialSections[key]
}

function materialCostShare(cost) {
  if (totalMaterialCost.value <= 0) return 0
  return Math.round((cost / totalMaterialCost.value) * 100)
}

function resetDefaults() {
  Object.assign(settings, structuredClone(defaultState.settings))
  Object.assign(ratios, structuredClone(defaultState.ratios))
  Object.keys(packOptions).forEach((key) => {
    packOptions[key] = structuredClone(defaultState.packOptions[key])
  })

  isPriceSettingsOpen.value = false
  isUnitCostBreakdownOpen.value = false
  Object.assign(openMaterialSections, {
    sheaButter: true,
    jojobaOil: false,
    beeswax: false,
    essentialOil: false,
  })
}

function formatCurrency(value) {
  return Number(value || 0).toLocaleString('zh-TW', {
    minimumFractionDigits: 0,
    maximumFractionDigits: 2,
  })
}

function formatNumber(value) {
  return Number(value || 0).toLocaleString('zh-TW', {
    minimumFractionDigits: 0,
    maximumFractionDigits: 2,
  })
}
</script>
