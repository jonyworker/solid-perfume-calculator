<template>
  <div class="min-h-screen bg-slate-100/60 p-6 text-slate-950">
    <div class="mx-auto max-w-7xl space-y-6">
      <div class="space-y-1">
        <h1 class="text-3xl font-semibold tracking-tight">體香膏製作計算機</h1>
        <p class="text-sm text-slate-500">
          可設定三條產線配方、材料價格、容器數量與單罐克數，並依不可拆賣規則自動估算採購成本。
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
              <h2 class="text-base font-semibold tracking-tight">製作比例 (%)</h2>
              <p class="mt-1 text-sm text-slate-500">
                三條產線可各自設定不同配方比例，系統會分別驗證是否為 100%。
              </p>
            </div>

            <div class="space-y-4 p-6">
              <div
                v-for="line in productionLines"
                :key="line.key"
                class="rounded-xl border border-slate-200 bg-slate-50/50 p-4"
              >
                <div class="flex items-center justify-between gap-4">
                  <div>
                    <h3 class="text-sm font-semibold text-slate-900">{{ line.label }}</h3>
                    <p class="mt-1 text-xs text-slate-500">
                      這條產線的基底與精油比例會獨立影響備料、成本與採購結果。
                    </p>
                  </div>

                  <div class="text-sm text-slate-500">
                    比例總和：
                    <span
                      class="font-semibold"
                      :class="getLineRatioTotal(line.key) === 100 ? 'text-emerald-600' : 'text-rose-600'"
                    >
                      {{ formatNumber(getLineRatioTotal(line.key)) }}%
                    </span>
                  </div>
                </div>

                <div
                  v-if="getLineRatioError(line.key)"
                  class="mt-4 rounded-lg border border-rose-200 bg-rose-50/80 px-4 py-3 text-sm text-rose-700"
                >
                  {{ getLineRatioError(line.key) }}
                </div>

                <div class="mt-4 overflow-hidden rounded-full bg-slate-100">
                  <div class="flex h-2.5 w-full">
                    <div
                      v-for="segment in getLineRatioSegments(line.key)"
                      :key="`${line.key}-${segment.key}`"
                      class="h-full transition-all"
                      :class="segment.barClass"
                      :style="{ width: `${segment.width}%` }"
                    ></div>
                    <div
                      v-if="getLineRatioGap(line.key) > 0"
                      class="h-full bg-rose-300/70"
                      :style="{ width: `${getLineRatioGap(line.key)}%` }"
                    ></div>
                  </div>
                </div>

                <div class="mt-4 grid gap-4 md:grid-cols-2">
                  <label
                    v-for="material in materials"
                    :key="`${line.key}-${material.key}`"
                    class="rounded-lg border border-slate-200 bg-white p-4"
                  >
                    <div class="flex items-center justify-between gap-3">
                      <span class="text-sm font-medium">{{ material.label }}</span>
                      <span
                        class="inline-flex items-center rounded-md border px-2.5 py-1 text-xs font-medium"
                        :class="material.badgeClass"
                      >
                        {{ formatNumber(ratiosByLine[line.key][material.key]) }}%
                      </span>
                    </div>

                    <div class="mt-3 flex items-center gap-3 rounded-md border border-slate-200 bg-white px-3 py-2.5 transition-colors focus-within:border-slate-300 focus-within:ring-2 focus-within:ring-slate-950/5">
                      <input
                        v-model.number="ratiosByLine[line.key][material.key]"
                        type="number"
                        min="0"
                        step="1"
                        class="w-full bg-transparent text-xl font-semibold outline-none"
                      />
                      <span class="text-sm font-medium text-slate-500">%</span>
                    </div>
                  </label>
                </div>
              </div>
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
                          單位成本：NT$ {{ formatCurrency(unitPricePerMl(pack.price, pack.size)) }}/{{ material.unitLabel }}
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
                依三條產線各自的配方比例，顯示實際要準備的基底原料與精油用量。
              </p>
            </div>

            <div class="p-6 text-sm text-emerald-900">
              <div class="space-y-4">
                <div class="rounded-xl border border-emerald-200 bg-emerald-50/50 p-4">
                  <div class="text-sm font-semibold text-emerald-900">產線分配總覽</div>
                  <div class="mt-1 text-xs text-emerald-700">
                    必買複方精油固定使用 1 瓶，剩餘罐數由尤加利與薰衣草平分。
                  </div>

                  <div class="mt-3 grid gap-3 md:grid-cols-3">
                    <div class="rounded-lg border border-emerald-100 bg-white px-3 py-3">
                      <div class="text-xs text-emerald-700">🧪 必買複方</div>
                      <div class="mt-1 text-base font-semibold">
                        {{ formatNumber(oilUsageSummary.specialCount) }} 罐
                      </div>
                    </div>

                    <div class="rounded-lg border border-emerald-100 bg-white px-3 py-3">
                      <div class="text-xs text-emerald-700">🌿 尤加利</div>
                      <div class="mt-1 text-base font-semibold">
                        {{ formatNumber(oilUsageSummary.eucalyptusCount) }} 罐
                      </div>
                    </div>

                    <div class="rounded-lg border border-emerald-100 bg-white px-3 py-3">
                      <div class="text-xs text-emerald-700">🌸 薰衣草</div>
                      <div class="mt-1 text-base font-semibold">
                        {{ formatNumber(oilUsageSummary.lavenderCount) }} 罐
                      </div>
                    </div>
                  </div>
                </div>

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
                        {{ formatNumber(line.essentialOilMl) }} ml <span class="text-xs text-slate-500"> ≈ {{ formatNumber(mlToGram(line.essentialOilMl)) }} g</span>
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
              <div class="text-sm text-slate-400">平均單罐成本</div>
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
  ratiosByLine: {
    special: {
      sheaButter: 56,
      jojobaOil: 17,
      beeswax: 22,
      essentialOil: 5,
    },
    eucalyptus: {
      sheaButter: 57,
      jojobaOil: 18,
      beeswax: 22,
      essentialOil: 3,
    },
    lavender: {
      sheaButter: 56,
      jojobaOil: 18,
      beeswax: 22,
      essentialOil: 4,
    },
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
const ratiosByLine = reactive(structuredClone(defaultState.ratiosByLine))
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

const productionLines = [
  { key: 'special', label: '🧪 必買複方配方' },
  { key: 'eucalyptus', label: '🌿 尤加利配方' },
  { key: 'lavender', label: '🌸 薰衣草配方' },
]

const lineDisplayMap = {
  special: '🧪 必買複方精油',
  eucalyptus: '🌿 尤加利精油',
  lavender: '🌸 薰衣草精油',
}

const lineOilPackMap = {
  special: 'specialEssentialOil',
  eucalyptus: 'eucalyptusEssentialOil',
  lavender: 'lavenderEssentialOil',
}

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

const totalWeight = computed(() => {
  return Number(settings.containerCount || 0) * Number(settings.gramsPerContainer || 0)
})

const essentialOilDensity = computed(() => {
  return materials.find((item) => item.key === 'essentialOil')?.density || 0.9
})

const specialEssentialOilBottleMl = computed(() => {
  return Number(packOptions.specialEssentialOil?.[0]?.size || 15)
})

function getLineRatioTotal(lineKey) {
  return Object.values(ratiosByLine[lineKey]).reduce((sum, value) => sum + Number(value || 0), 0)
}

function getLineRatioGap(lineKey) {
  return Math.max(0, 100 - Math.min(getLineRatioTotal(lineKey), 100))
}

function getLineRatioSegments(lineKey) {
  return materials.map((material) => ({
    key: material.key,
    width: Math.max(0, Math.min(Number(ratiosByLine[lineKey][material.key] || 0), 100)),
    barClass: material.barClass,
  }))
}

function getLineRatioError(lineKey) {
  const total = getLineRatioTotal(lineKey)

  if (total > 100) {
    return `${lineDisplayMap[lineKey]}比例總和不可超過 100%，目前為 ${formatNumber(total)}%。`
  }

  if (total < 100) {
    return `${lineDisplayMap[lineKey]}比例總和必須等於 100%，目前為 ${formatNumber(total)}%。`
  }

  return ''
}

const hasAnyRatioError = computed(() => {
  return productionLines.some((line) => Boolean(getLineRatioError(line.key)))
})

function getLineEssentialOilRatio(lineKey) {
  return Number(ratiosByLine[lineKey].essentialOil || 0) / 100
}

function getLineEssentialOilPerContainerMl(lineKey) {
  const gramsPerContainer = Number(settings.gramsPerContainer || 0)
  const essentialRatio = getLineEssentialOilRatio(lineKey)
  const perContainerGrams = gramsPerContainer * essentialRatio
  return essentialOilDensity.value > 0 ? perContainerGrams / essentialOilDensity.value : 0
}

const maxSpecialEssentialOilContainers = computed(() => {
  const perContainerMl = getLineEssentialOilPerContainerMl('special')
  if (perContainerMl <= 0) return 0
  return Math.floor(specialEssentialOilBottleMl.value / perContainerMl)
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

const productionPreparationDetails = computed(() => {
  const gramsPerContainer = Number(settings.gramsPerContainer || 0)

  const lineCounts = {
    special: oilUsageSummary.value.specialCount,
    eucalyptus: oilUsageSummary.value.eucalyptusCount,
    lavender: oilUsageSummary.value.lavenderCount,
  }

  return productionLines.map((line) => {
    const containers = lineCounts[line.key]
    const totalLineWeight = containers * gramsPerContainer

    const sheaRatio = Number(ratiosByLine[line.key].sheaButter || 0) / 100
    const jojobaRatio = Number(ratiosByLine[line.key].jojobaOil || 0) / 100
    const beeswaxRatio = Number(ratiosByLine[line.key].beeswax || 0) / 100

    const essentialOilMl = containers * getLineEssentialOilPerContainerMl(line.key)

    return {
      key: line.key,
      label: lineDisplayMap[line.key],
      containers,
      essentialOilMl,
      sheaButterGrams: totalLineWeight * sheaRatio,
      jojobaOilGrams: totalLineWeight * jojobaRatio,
      beeswaxGrams: totalLineWeight * beeswaxRatio,
    }
  })
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

function calculateLeftoverValue(item) {
  if (item.purchasedAmount <= 0 || item.purchaseCost <= 0 || item.leftoverAmount <= 0) return 0
  return (item.purchaseCost / item.purchasedAmount) * item.leftoverAmount
}

const materialResults = computed(() => {
  const rows = []

  const totalBaseRequirements = productionPreparationDetails.value.reduce(
    (acc, line) => {
      acc.sheaButter += line.sheaButterGrams
      acc.jojobaOil += line.jojobaOilGrams
      acc.beeswax += line.beeswaxGrams
      return acc
    },
    {
      sheaButter: 0,
      jojobaOil: 0,
      beeswax: 0,
    },
  )

  const baseMaterialMeta = materials.filter((item) => item.key !== 'essentialOil')

  baseMaterialMeta.forEach((material) => {
    const requiredAmount = totalBaseRequirements[material.key]
    const options = normalizePackOptions(packOptions[material.key])

    const purchase = hasAnyRatioError.value
      ? { totalAmount: 0, totalCost: 0, combination: [] }
      : findBestCombination(requiredAmount, options)

    rows.push({
      key: material.key,
      label: material.label,
      unitLabel: material.unitLabel,
      barClass: material.barClass,
      requiredAmount,
      formula: productionPreparationDetails.value
        .map((line) => `${lineDisplayMap[line.key]} ${formatNumber(line[`${material.key}Grams`] || 0)}g`)
        .join(' + '),
      purchasedAmount: purchase.totalAmount,
      purchaseCost: purchase.totalCost,
      leftoverAmount: Math.max(0, purchase.totalAmount - requiredAmount),
      leftoverWarning: requiredAmount > 0 ? purchase.totalAmount - requiredAmount > requiredAmount * 0.1 : false,
      combination: purchase.combination,
      lineKey: null,
      perContainerAmount: Number(settings.containerCount || 0) > 0 ? requiredAmount / Number(settings.containerCount || 0) : 0,
    })
  })

  productionPreparationDetails.value.forEach((line) => {
    const options = normalizePackOptions(packOptions[lineOilPackMap[line.key]])
    const purchase = hasAnyRatioError.value
      ? { totalAmount: 0, totalCost: 0, combination: [] }
      : findBestCombination(line.essentialOilMl, options)

    rows.push({
      key: `essentialOil-${line.key}`,
      label: line.label,
      unitLabel: 'ml',
      barClass:
        line.key === 'special'
          ? 'bg-slate-400'
          : line.key === 'eucalyptus'
            ? 'bg-slate-600'
            : 'bg-slate-500',
      requiredAmount: line.essentialOilMl,
      formula: `${formatNumber(line.containers)}罐 × ${formatNumber(getLineEssentialOilPerContainerMl(line.key))}ml/罐`,
      purchasedAmount: purchase.totalAmount,
      purchaseCost: purchase.totalCost,
      leftoverAmount: Math.max(0, purchase.totalAmount - line.essentialOilMl),
      leftoverWarning: line.essentialOilMl > 0
        ? purchase.totalAmount - line.essentialOilMl > line.essentialOilMl * 0.1
        : false,
      combination: purchase.combination,
      lineKey: line.key,
      perContainerAmount: line.containers > 0 ? line.essentialOilMl / line.containers : 0,
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
  if (hasAnyRatioError.value) return 0
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
  const totalCount = Number(settings.containerCount || 0)

  if (totalCount <= 0) {
    return materialResults.value.map((item) => ({
      ...item,
      requiredAmountPerContainer: 0,
      costPerContainer: 0,
    }))
  }

  return materialResults.value.map((item) => ({
    ...item,
    requiredAmountPerContainer: item.perContainerAmount,
    costPerContainer: item.purchaseCost / totalCount,
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

function unitPricePerMl(price, size) {
  const normalizedPrice = Number(price || 0)
  const normalizedSize = Number(size || 0)
  if (normalizedPrice <= 0 || normalizedSize <= 0) return 0
  return normalizedPrice / normalizedSize
}

function resetDefaults() {
  Object.assign(settings, structuredClone(defaultState.settings))

  Object.keys(ratiosByLine).forEach((lineKey) => {
    Object.assign(ratiosByLine[lineKey], structuredClone(defaultState.ratiosByLine[lineKey]))
  })

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
function mlToGram(ml) {
  const density = essentialOilDensity.value || 0.9
  return ml * density
}
</script>