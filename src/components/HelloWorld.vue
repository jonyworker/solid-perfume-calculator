<template>
  <div class="min-h-screen bg-slate-100/60 p-6 text-slate-950">
    <div class="mx-auto max-w-7xl space-y-6">
      <div class="space-y-1">
        <h1 class="text-3xl font-semibold tracking-tight">香膏製作計算機</h1>
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
                <span class="text-sm font-medium text-slate-600">香膏容器數量</span>
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
                <span class="text-sm font-medium text-slate-600">每個容器內容物</span>
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
                <div class="flex items-center justify-between gap-3">
                  <span class="text-sm font-medium">{{ material.label }}</span>
                  <span class="inline-flex items-center rounded-md border px-2.5 py-1 text-xs font-medium" :class="material.badgeClass">
                    {{ formatNumber(ratios[material.key]) }}%
                  </span>
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
                      <template v-if="material.key === 'essentialOil'">
                        3 組精油設定，{{ totalEssentialOilPackCount }} 個採購規格
                      </template>
                      <template v-else>
                        {{ packOptions[material.key].length }} 個規格，{{ material.unitLabel }} 為採購單位
                      </template>
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
                  <template v-if="material.key === 'essentialOil'">
                    <div class="space-y-3 rounded-lg border border-slate-200 bg-slate-50/70 px-4 py-3 text-xs text-slate-600">
                      <div>
                        必買複方精油固定使用 1 瓶，系統會自動換算最多可做幾罐；剩餘罐數再由尤加利與薰衣草精油平分。
                      </div>
                      <div class="grid gap-2 md:grid-cols-3">
                        <div class="rounded-md border border-slate-200 bg-white px-3 py-2">
                          必買複方：{{ formatNumber(oilUsageSummary.specialCount) }} 罐
                        </div>
                        <div class="rounded-md border border-slate-200 bg-white px-3 py-2">
                          尤加利：{{ formatNumber(oilUsageSummary.eucalyptusCount) }} 罐
                        </div>
                        <div class="rounded-md border border-slate-200 bg-white px-3 py-2">
                          薰衣草：{{ formatNumber(oilUsageSummary.lavenderCount) }} 罐
                        </div>
                      </div>
                    </div>

                    <div class="space-y-4">
                      <div class="rounded-lg border border-slate-200 bg-slate-50/50 p-4">
                        <div class="mb-3">
                          <h4 class="text-sm font-semibold text-slate-800">必買複方精油</h4>
                          <p class="mt-1 text-xs text-slate-500">固定使用 1 瓶，可編輯容量與價格。</p>
                        </div>
                        <div class="grid gap-3 md:grid-cols-3">
                          <div
                            v-for="(pack, index) in packOptions.specialEssentialOil"
                            :key="`special-essential-oil-${index}`"
                            class="rounded-lg border border-slate-200 bg-white p-3"
                          >
                            <div class="space-y-2">
                              <label class="block space-y-1">
                                <span class="text-xs text-slate-500">規格 (ml)</span>
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
                              <div class="rounded-md border border-slate-200 bg-slate-50 px-3 py-2 text-xs text-slate-600">
                                單位成本：NT$ {{ formatCurrency(unitPricePerMl(pack.price, pack.size)) }}/ml
                              </div>
                            </div>
                          </div>
                        </div>
                      </div>

                      <div class="rounded-lg border border-slate-200 bg-slate-50/50 p-4">
                        <div class="mb-3">
                          <h4 class="text-sm font-semibold text-slate-800">尤加利精油</h4>
                          <p class="mt-1 text-xs text-slate-500">剩餘罐數的一半會分配到這裡，可設定 10 / 30 / 50 / 100 ml 價格。</p>
                        </div>
                        <div class="grid gap-3 md:grid-cols-2 xl:grid-cols-4">
                          <div
                            v-for="(pack, index) in packOptions.eucalyptusEssentialOil"
                            :key="`eucalyptus-essential-oil-${index}`"
                            class="rounded-lg border border-slate-200 bg-white p-3"
                          >
                            <div class="space-y-2">
                              <label class="block space-y-1">
                                <span class="text-xs text-slate-500">規格 (ml)</span>
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
                              <div class="rounded-md border border-slate-200 bg-slate-50 px-3 py-2 text-xs text-slate-600">
                                單位成本：NT$ {{ formatCurrency(unitPricePerMl(pack.price, pack.size)) }}/ml
                              </div>
                            </div>
                          </div>
                        </div>
                      </div>

                      <div class="rounded-lg border border-slate-200 bg-slate-50/50 p-4">
                        <div class="mb-3">
                          <h4 class="text-sm font-semibold text-slate-800">薰衣草精油</h4>
                          <p class="mt-1 text-xs text-slate-500">剩餘罐數的另一半會分配到這裡，可設定 10 / 30 / 50 / 100 ml 價格。</p>
                        </div>
                        <div class="grid gap-3 md:grid-cols-2 xl:grid-cols-4">
                          <div
                            v-for="(pack, index) in packOptions.lavenderEssentialOil"
                            :key="`lavender-essential-oil-${index}`"
                            class="rounded-lg border border-slate-200 bg-white p-3"
                          >
                            <div class="space-y-2">
                              <label class="block space-y-1">
                                <span class="text-xs text-slate-500">規格 (ml)</span>
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
                              <div class="rounded-md border border-slate-200 bg-slate-50 px-3 py-2 text-xs text-slate-600">
                                單位成本：NT$ {{ formatCurrency(unitPricePerMl(pack.price, pack.size)) }}/ml
                              </div>
                            </div>
                          </div>
                        </div>
                      </div>
                    </div>
                  </template>

                  <div v-else class="grid gap-3 md:grid-cols-3">
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
                        <div class="rounded-md border border-slate-200 bg-slate-50 px-3 py-2 text-xs text-slate-600">
                          單位成本：NT$ {{ formatCurrency(unitPricePerMl(pack.price, pack.size)) }}/ml
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div class="rounded-xl border border-emerald-200 bg-white shadow-sm">
            <div class="border-b border-emerald-100 px-6 py-4">
              <h2 class="text-base font-semibold tracking-tight text-emerald-900">
                製作準備（實際操作）
              </h2>
              <p class="mt-1 text-sm text-emerald-700/80">
                依三條產線分別顯示實際要準備的基底原料與精油用量。
              </p>
            </div>

            <div class="p-6 text-sm text-emerald-900">
              <div class="space-y-4">
                <div
                  v-for="line in productionPreparationDetails"
                  :key="line.key"
                  class="rounded-xl border border-emerald-200 bg-emerald-50/50 p-4"
                >
                  <div class="flex flex-col gap-2 md:flex-row md:items-center md:justify-between">
                    <div class="font-semibold">
                      {{ line.label }}
                    </div>
                    <div class="text-sm text-emerald-800">
                      精油 {{ formatNumber(line.essentialOilMl) }} ml
                      ・約 {{ formatNumber(line.containers) }} 罐
                    </div>
                  </div>

                  <div class="mt-3 grid gap-3 md:grid-cols-2 xl:grid-cols-4">
                    <div class="rounded-lg border border-emerald-100 bg-white px-3 py-3">
                      <div class="text-xs text-emerald-700">乳油木果油</div>
                      <div class="mt-1 text-base font-semibold">
                        {{ formatNumber(line.sheaButterGrams) }} g
                      </div>
                    </div>

                    <div class="rounded-lg border border-emerald-100 bg-white px-3 py-3">
                      <div class="text-xs text-emerald-700">荷荷芭油</div>
                      <div class="mt-1 text-base font-semibold">
                        {{ formatNumber(line.jojobaOilGrams) }} g
                      </div>
                    </div>

                    <div class="rounded-lg border border-emerald-100 bg-white px-3 py-3">
                      <div class="text-xs text-emerald-700">蜂蠟</div>
                      <div class="mt-1 text-base font-semibold">
                        {{ formatNumber(line.beeswaxGrams) }} g
                      </div>
                    </div>

                    <div class="rounded-lg border border-emerald-100 bg-white px-3 py-3">
                      <div class="text-xs text-emerald-700">精油</div>
                      <div class="mt-1 text-base font-semibold">
                        {{ formatNumber(line.essentialOilMl) }} ml
                      </div>
                    </div>
                  </div>
                </div>

                <div class="rounded-lg border border-dashed border-emerald-200 bg-emerald-50/40 px-4 py-3">
                  <span class="text-emerald-700">容器準備：</span>
                  <span class="font-semibold text-emerald-900">
                    {{ formatNumber(settings.containerCount) }} 個
                  </span>
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
                  <div class="flex items-center justify-between text-xs text-slate-500">
                    <span>剩料約值</span>
                    <span>NT$ {{ formatCurrency(item.leftoverValue) }}</span>
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
    sheaButter: 57,
    jojobaOil: 18,
    beeswax: 22,
    essentialOil: 3,
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
    specialEssentialOil: [
      { size: 15, price: 1280 },
    ],
    eucalyptusEssentialOil: [
      { size: 10, price: 120 },
      { size: 30, price: 160 },
      { size: 50, price: 240 },
      { size: 100, price: 410 },
    ],
    lavenderEssentialOil: [
      { size: 10, price: 130 },
      { size: 30, price: 190 },
      { size: 50, price: 280 },
      { size: 100, price: 510 },
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

const totalEssentialOilPackCount = computed(() => {
  return (
    (packOptions.specialEssentialOil?.length || 0) +
    (packOptions.eucalyptusEssentialOil?.length || 0) +
    (packOptions.lavenderEssentialOil?.length || 0)
  )
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

const specialEssentialOilBottleMl = computed(() => {
  return Number(packOptions.specialEssentialOil?.[0]?.size || 15)
})

const essentialOilDensity = computed(() => {
  return materials.find((item) => item.key === 'essentialOil')?.density || 0.9
})

const essentialOilPerContainerMl = computed(() => {
  const ratio = Number(ratios.essentialOil || 0) / 100
  const gramsPerContainer = Number(settings.gramsPerContainer || 0)
  const perContainerGrams = gramsPerContainer * ratio
  return essentialOilDensity.value > 0 ? perContainerGrams / essentialOilDensity.value : 0
})

const maxSpecialEssentialOilContainers = computed(() => {
  const perContainerMl = essentialOilPerContainerMl.value
  if (perContainerMl <= 0) return 0
  return Math.floor(specialEssentialOilBottleMl.value / perContainerMl)
})

const materialResults = computed(() => {
  const rows = []

  materials.forEach((material) => {
    const ratio = Number(ratios[material.key] || 0) / 100
    const totalRequiredGrams = totalWeight.value * ratio

    if (material.key === 'essentialOil') {
      const perContainerMl = essentialOilPerContainerMl.value

      const maxSpecialCount = Math.min(
        Number(settings.containerCount || 0),
        maxSpecialEssentialOilContainers.value,
      )

      const totalCount = Number(settings.containerCount || 0)
      const remainingCount = Math.max(0, totalCount - maxSpecialCount)

      const specialRequiredMl = maxSpecialCount * perContainerMl
      const remainingRequiredMl = remainingCount * perContainerMl

      const eachStandardMl = remainingRequiredMl / 2

      const specialOptions = normalizePackOptions(packOptions.specialEssentialOil)
      const eucalyptusOptions = normalizePackOptions(packOptions.eucalyptusEssentialOil)
      const lavenderOptions = normalizePackOptions(packOptions.lavenderEssentialOil)

      const specialPurchase = ratioError.value
        ? { totalAmount: 0, totalCost: 0, combination: [] }
        : findBestCombination(specialRequiredMl, specialOptions)
      const eucalyptusPurchase = ratioError.value
        ? { totalAmount: 0, totalCost: 0, combination: [] }
        : findBestCombination(eachStandardMl, eucalyptusOptions)
      const lavenderPurchase = ratioError.value
        ? { totalAmount: 0, totalCost: 0, combination: [] }
        : findBestCombination(eachStandardMl, lavenderOptions)

      rows.push({
        key: 'essentialOil-special',
        label: '必買複方精油',
        unitLabel: 'ml',
        barClass: 'bg-slate-400',
        requiredAmount: specialRequiredMl,
        formula: `${formatNumber(specialEssentialOilBottleMl.value)}ml ÷ ${formatNumber(perContainerMl)}ml/罐 = 最多 ${maxSpecialCount}罐`,
        purchasedAmount: specialPurchase.totalAmount,
        purchaseCost: specialPurchase.totalCost,
        leftoverAmount: Math.max(0, specialPurchase.totalAmount - specialRequiredMl),
        leftoverWarning: specialPurchase.totalAmount - specialRequiredMl > perContainerMl * 0.1,
        combination: specialPurchase.combination,
      })

      rows.push({
        key: 'essentialOil-eucalyptus',
        label: '尤加利精油',
        unitLabel: 'ml',
        barClass: 'bg-slate-600',
        requiredAmount: eachStandardMl,
        formula: `剩餘 ${remainingCount}罐 × ${formatNumber(perContainerMl)}ml ÷ 2`,
        purchasedAmount: eucalyptusPurchase.totalAmount,
        purchaseCost: eucalyptusPurchase.totalCost,
        leftoverAmount: Math.max(0, eucalyptusPurchase.totalAmount - eachStandardMl),
        leftoverWarning: false,
        combination: eucalyptusPurchase.combination,
      })

      rows.push({
        key: 'essentialOil-lavender',
        label: '薰衣草精油',
        unitLabel: 'ml',
        barClass: 'bg-slate-500',
        requiredAmount: eachStandardMl,
        formula: `剩餘 ${remainingCount}罐 × ${formatNumber(perContainerMl)}ml ÷ 2`,
        purchasedAmount: lavenderPurchase.totalAmount,
        purchaseCost: lavenderPurchase.totalCost,
        leftoverAmount: Math.max(0, lavenderPurchase.totalAmount - eachStandardMl),
        leftoverWarning: false,
        combination: lavenderPurchase.combination,
      })

      return
    }

    const requiredAmount = material.unitLabel === 'ml'
      ? totalRequiredGrams / material.density
      : totalRequiredGrams

    const options = normalizePackOptions(packOptions[material.key])
    const purchase = ratioError.value
      ? { totalAmount: 0, totalCost: 0, combination: [] }
      : findBestCombination(requiredAmount, options)

    rows.push({
      key: material.key,
      label: material.label,
      unitLabel: material.unitLabel,
      barClass: material.barClass,
      requiredAmount,
      formula: `${formatNumber(totalWeight.value)}g × ${formatNumber(ratio * 100)}%`,
      purchasedAmount: purchase.totalAmount,
      purchaseCost: purchase.totalCost,
      leftoverAmount: Math.max(0, purchase.totalAmount - requiredAmount),
      leftoverWarning: purchase.totalAmount - requiredAmount > requiredAmount * 0.1,
      combination: purchase.combination,
    })
  })

  const highestCost = rows.reduce((max, item) => Math.max(max, item.purchaseCost), 0)

  return rows.map((item) => ({
    ...item,
    leftoverValue: calculateLeftoverValue(item),
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

  return materialResults.value.map((item) => {
    if (item.key.includes('essentialOil')) {
      return {
        ...item,
        requiredAmountPerContainer: essentialOilPerContainerMl.value,
        costPerContainer: item.purchaseCost / count,
      }
    }

    return {
      ...item,
      requiredAmountPerContainer: item.requiredAmount / count,
      costPerContainer: item.purchaseCost / count,
    }
  })
})

const unitMaterialCostPerContainer = computed(() => {
  const count = Number(settings.containerCount || 0)
  if (count <= 0) return 0
  return totalMaterialCost.value / count
})

const oilUsageSummary = computed(() => {
  const totalCount = Number(settings.containerCount || 0)
  const specialCount = Math.min(totalCount, maxSpecialEssentialOilContainers.value)
  const remainingCount = Math.max(0, totalCount - specialCount)
  return {
    specialCount,
    remainingCount,
    eucalyptusCount: remainingCount / 2,
    lavenderCount: remainingCount / 2,
  }
})

const oilPreparation = computed(() => {
  const perContainerMl = essentialOilPerContainerMl.value

  const specialCount = oilUsageSummary.value.specialCount
  const eucalyptusCount = oilUsageSummary.value.eucalyptusCount
  const lavenderCount = oilUsageSummary.value.lavenderCount

  return {
    specialMl: specialCount * perContainerMl,
    eucalyptusMl: eucalyptusCount * perContainerMl,
    lavenderMl: lavenderCount * perContainerMl,

    specialContainers: specialCount,
    eucalyptusContainers: eucalyptusCount,
    lavenderContainers: lavenderCount,
  }
})

const productionPreparationDetails = computed(() => {
  const gramsPerContainer = Number(settings.gramsPerContainer || 0)

  const sheaRatio = Number(ratios.sheaButter || 0) / 100
  const jojobaRatio = Number(ratios.jojobaOil || 0) / 100
  const beeswaxRatio = Number(ratios.beeswax || 0) / 100

  const lines = [
    {
      key: 'special',
      label: '必買複方精油',
      containers: oilPreparation.value.specialContainers,
      essentialOilMl: oilPreparation.value.specialMl,
    },
    {
      key: 'eucalyptus',
      label: '尤加利精油',
      containers: oilPreparation.value.eucalyptusContainers,
      essentialOilMl: oilPreparation.value.eucalyptusMl,
    },
    {
      key: 'lavender',
      label: '薰衣草精油',
      containers: oilPreparation.value.lavenderContainers,
      essentialOilMl: oilPreparation.value.lavenderMl,
    },
  ]

  return lines.map((line) => {
    const totalBaseWeight = line.containers * gramsPerContainer

    return {
      ...line,
      sheaButterGrams: totalBaseWeight * sheaRatio,
      jojobaOilGrams: totalBaseWeight * jojobaRatio,
      beeswaxGrams: totalBaseWeight * beeswaxRatio,
    }
  })
})

function calculateLeftoverValue(item) {
  if (item.purchasedAmount <= 0 || item.purchaseCost <= 0 || item.leftoverAmount <= 0) return 0
  return (item.purchaseCost / item.purchasedAmount) * item.leftoverAmount
}

function unitPricePerMl(price, size) {
  const normalizedPrice = Number(price || 0)
  const normalizedSize = Number(size || 0)
  if (normalizedPrice <= 0 || normalizedSize <= 0) return 0
  return normalizedPrice / normalizedSize
}

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