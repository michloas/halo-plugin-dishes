<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { VButton, VCard, VEmpty, VLoading, VPageHeader, VPagination } from '@halo-dev/components'
import { getApiErrorMessage, getDishStatistics, getOrderStatistics } from '@/api/client'
import { utils } from '@halo-dev/ui-shared'
import RiBarChartBoxLine from '~icons/ri/bar-chart-box-line'

type DishStat = {
  dishId: number
  dishName: string
  categoryName: string
  imageUrl?: string | null
  orderCount: number
  totalQuantity: number
}

type StatisticsResp = {
  overall: DishStat[]
  breakfast: DishStat[]
  lunch: DishStat[]
  dinner: DishStat[]
  from: string
  to: string
}

type OrderStatisticsResp = {
  summary: {
    totalOrders: number
    totalItems: number
    activeDays: number
    avgItemsPerOrder: number
  }
  dateTrend: Array<{
    date: string
    count: number
  }>
  periodDistribution: Array<{
    code: string
    name: string
    count: number
  }>
  categoryStats: Array<{
    categoryId: number
    categoryName: string
    count: number
  }>
  from: string
  to: string
}

const route = useRoute()
const router = useRouter()

const loading = ref(true)
const error = ref<string | null>(null)
const statistics = ref<StatisticsResp>({ overall: [], breakfast: [], lunch: [], dinner: [], from: '', to: '' })
const orderStats = ref<OrderStatisticsResp | null>(null)

const queryFrom = ref('')
const queryTo = ref('')
const topN = ref(20)
const page = ref(1)
const size = ref(20)
const viewMode = ref<'list' | 'chart'>('list')
const activeTab = ref<'overall' | 'breakfast' | 'lunch' | 'dinner'>('overall')

function ymd(d: Date): string {
  const y = d.getFullYear()
  const m = String(d.getMonth() + 1).padStart(2, '0')
  const day = String(d.getDate()).padStart(2, '0')
  return `${y}-${m}-${day}`
}

function getCurrentMonthRange() {
  const now = new Date()
  const firstDay = new Date(now.getFullYear(), now.getMonth(), 1)
  const lastDay = new Date(now.getFullYear(), now.getMonth() + 1, 0)
  return { from: ymd(firstDay), to: ymd(lastDay) }
}

function setFromQuery() {
  const q = route.query
  const qFrom = typeof q.from === 'string' ? q.from : ''
  const qTo = typeof q.to === 'string' ? q.to : ''
  const qTop = typeof q.top === 'string' ? parseInt(q.top, 10) : NaN
  const qPage = typeof q.page === 'string' ? parseInt(q.page, 10) : NaN
  const qSize = typeof q.size === 'string' ? parseInt(q.size, 10) : NaN
  const qMode = typeof q.mode === 'string' ? q.mode : 'list'

  queryFrom.value = qFrom || ''
  queryTo.value = qTo || ''
  if (!Number.isNaN(qTop) && qTop > 0) topN.value = qTop
  if (!Number.isNaN(qPage) && qPage > 0) page.value = qPage
  if (!Number.isNaN(qSize) && qSize > 0) size.value = qSize
  if (qMode === 'chart') viewMode.value = 'chart'
  else viewMode.value = 'list'
}

function syncQuery() {
  const nextQuery: Record<string, string> = {
    page: String(page.value),
    size: String(size.value),
    top: String(topN.value),
    mode: viewMode.value,
  }
  if (queryFrom.value) nextQuery.from = queryFrom.value
  if (queryTo.value) nextQuery.to = queryTo.value
  void router.replace({ query: nextQuery })
}

async function load() {
  loading.value = true
  error.value = null
  try {
    const params: { from?: string; to?: string; top: number } = { top: topN.value }
    if (queryFrom.value) params.from = queryFrom.value
    if (queryTo.value) params.to = queryTo.value

    // 并行加载菜品统计和订单统计
    const [dishData, orderData] = await Promise.all([
      getDishStatistics(params),
      getOrderStatistics({ from: params.from, to: params.to })
    ])

    statistics.value = dishData
    orderStats.value = orderData

    const currentItems = getCurrentItems()
    const maxPage = Math.max(1, Math.ceil(currentItems.length / size.value))
    if (page.value > maxPage) page.value = maxPage
    syncQuery()
  } catch (e) {
    error.value = getApiErrorMessage(e, '加载统计数据失败，请稍后重试', {
      DISHES_ACCESS_DENIED: '当前账号无权访问统计数据',
    })
  } finally {
    loading.value = false
  }
}

function onSearch() {
  page.value = 1
  syncQuery()
  void load()
}

function clearDateFilter() {
  queryFrom.value = ''
  queryTo.value = ''
  page.value = 1
  syncQuery()
  void load()
}

function setCurrentMonth() {
  const { from, to } = getCurrentMonthRange()
  queryFrom.value = from
  queryTo.value = to
  page.value = 1
  syncQuery()
  void load()
}

function toggleViewMode() {
  viewMode.value = viewMode.value === 'list' ? 'chart' : 'list'
  syncQuery()
}

function getCurrentItems(): DishStat[] {
  return statistics.value[activeTab.value] || []
}

const pagedItems = computed(() => {
  const currentItems = getCurrentItems()
  const start = (page.value - 1) * size.value
  return currentItems.slice(start, start + size.value)
})

function dishThumbSrc(url: string | null | undefined) {
  const raw = (url || '').trim()
  if (!raw) return ''
  return utils.attachment.getThumbnailUrl(raw, 'M')
}

function getMaxOrderCount() {
  const currentItems = getCurrentItems()
  if (currentItems.length === 0) return 1
  return Math.max(...currentItems.map((item) => item.orderCount))
}

function getTabLabel(tab: string): string {
  const labels: Record<string, string> = {
    overall: '全部',
    breakfast: '早餐',
    lunch: '午餐',
    dinner: '晚餐',
  }
  return labels[tab] || tab
}

function switchTab(tab: 'overall' | 'breakfast' | 'lunch' | 'dinner') {
  activeTab.value = tab
  page.value = 1
}

onMounted(() => {
  setFromQuery()
  if (!queryFrom.value && !queryTo.value) {
    const { from, to } = getCurrentMonthRange()
    queryFrom.value = from
    queryTo.value = to
  }
  void load()
})

watch(
  () => route.query,
  () => {
    setFromQuery()
  },
)
</script>

<template>
  <VPageHeader title="点餐统计">
    <template #icon>
      <RiBarChartBoxLine />
    </template>
  </VPageHeader>

  <div class="statistics-page :uno: p-4">
    <!-- 数据概览卡片 -->
    <div v-if="!loading && !error && orderStats" class="statistics-summary-cards">
      <div class="summary-card">
        <div class="summary-card-icon summary-card-icon--blue">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
            <polyline points="14 2 14 8 20 8"></polyline>
            <line x1="16" y1="13" x2="8" y2="13"></line>
            <line x1="16" y1="17" x2="8" y2="17"></line>
            <polyline points="10 9 9 9 8 9"></polyline>
          </svg>
        </div>
        <div class="summary-card-content">
          <div class="summary-card-label">总餐数</div>
          <div class="summary-card-value">{{ orderStats.summary.totalOrders }}</div>
        </div>
      </div>
      <div class="summary-card">
        <div class="summary-card-icon summary-card-icon--green">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <circle cx="9" cy="21" r="1"></circle>
            <circle cx="20" cy="21" r="1"></circle>
            <path d="M1 1h4l2.68 13.39a2 2 0 0 0 2 1.61h9.72a2 2 0 0 0 2-1.61L23 6H6"></path>
          </svg>
        </div>
        <div class="summary-card-content">
          <div class="summary-card-label">总点餐菜品数</div>
          <div class="summary-card-value">{{ orderStats.summary.totalItems }}</div>
        </div>
      </div>
      <div class="summary-card">
        <div class="summary-card-icon summary-card-icon--purple">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <rect x="3" y="4" width="18" height="18" rx="2" ry="2"></rect>
            <line x1="16" y1="2" x2="16" y2="6"></line>
            <line x1="8" y1="2" x2="8" y2="6"></line>
            <line x1="3" y1="10" x2="21" y2="10"></line>
          </svg>
        </div>
        <div class="summary-card-content">
          <div class="summary-card-label">活跃天数</div>
          <div class="summary-card-value">{{ orderStats.summary.activeDays }}</div>
        </div>
      </div>
      <div class="summary-card">
        <div class="summary-card-icon summary-card-icon--orange">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <line x1="12" y1="20" x2="12" y2="10"></line>
            <line x1="18" y1="20" x2="18" y2="4"></line>
            <line x1="6" y1="20" x2="6" y2="16"></line>
          </svg>
        </div>
        <div class="summary-card-content">
          <div class="summary-card-label">平均每单菜品数</div>
          <div class="summary-card-value">{{ orderStats.summary.avgItemsPerOrder }}</div>
        </div>
      </div>
    </div>

    <VCard class="statistics-main-card" :body-class="[':uno: !p-0']">
      <template #header>
        <div class=":uno: block w-full bg-gray-50 px-4 py-3">
          <div class=":uno: flex flex-wrap items-end justify-between gap-3">
            <div class=":uno: flex flex-wrap items-end gap-3">
              <div>
                <div class="statistics-label">开始日期</div>
                <input v-model="queryFrom" type="date" class="statistics-input" />
              </div>
              <div>
                <div class="statistics-label">结束日期</div>
                <input v-model="queryTo" type="date" class="statistics-input" />
              </div>
              <div>
                <div class="statistics-label">Top N</div>
                <input v-model.number="topN" type="number" min="1" max="100" class="statistics-input statistics-input-sm" />
              </div>
              <VButton type="primary" class="statistics-search-btn" :disabled="loading" @click="onSearch">查询</VButton>
              <VButton :disabled="loading" @click="setCurrentMonth">本月</VButton>
              <VButton :disabled="loading || (!queryFrom && !queryTo)" @click="clearDateFilter">清空</VButton>
            </div>

            <div class=":uno: flex items-center gap-2">
              <div class="statistics-meta">{{ getTabLabel(activeTab) }} {{ getCurrentItems().length }} 道菜品</div>
              <VButton :disabled="loading" @click="toggleViewMode">
                {{ viewMode === 'list' ? '图表视图' : '列表视图' }}
              </VButton>
              <VButton :disabled="loading" @click="load">刷新</VButton>
            </div>
          </div>
        </div>
      </template>

      <div class=":uno: p-4">
        <div v-if="error" class="statistics-error">{{ error }}</div>
        <VLoading v-else-if="loading" />

        <template v-else>
          <!-- Tab 切换 -->
          <div class="statistics-tabs">
            <button
              v-for="tab in ['overall', 'breakfast', 'lunch', 'dinner']"
              :key="tab"
              class="statistics-tab-btn"
              :class="{ 'statistics-tab-btn--active': activeTab === tab }"
              @click="switchTab(tab as any)"
            >
              {{ getTabLabel(tab) }} ({{ statistics[tab as keyof StatisticsResp].length }})
            </button>
          </div>

          <VEmpty v-if="getCurrentItems().length === 0" title="暂无统计数据" description="当前日期范围内没有可展示的数据。" />

          <div v-else>
            <div v-if="viewMode === 'list'" class="statistics-list-view">
              <table class="statistics-table">
                <thead>
                  <tr>
                    <th>排名</th>
                    <th>菜品</th>
                    <th>分类</th>
                    <th>点餐次数</th>
                    <th>总数量</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(item, index) in pagedItems" :key="item.dishId">
                    <td class="statistics-rank-cell">
                      <span class="statistics-rank-badge" :class="{ 'statistics-rank-badge--top3': index < 3 }">
                        {{ (page - 1) * size + index + 1 }}
                      </span>
                    </td>
                    <td>
                      <div class="statistics-dish-cell">
                        <img
                          v-if="item.imageUrl"
                          :src="dishThumbSrc(item.imageUrl)"
                          :alt="item.dishName"
                          class="statistics-dish-thumb"
                        />
                        <div v-else class="statistics-dish-thumb statistics-dish-thumb--placeholder">
                          {{ item.dishName.slice(0, 1) }}
                        </div>
                        <div class="statistics-dish-name">{{ item.dishName }}</div>
                      </div>
                    </td>
                    <td>{{ item.categoryName }}</td>
                    <td class="statistics-count-cell">{{ item.orderCount }}</td>
                    <td class="statistics-count-cell">{{ item.totalQuantity }}</td>
                  </tr>
                </tbody>
              </table>
            </div>

            <div v-else class="statistics-chart-view">
              <div
                v-for="(item, index) in getCurrentItems()"
                :key="item.dishId"
                class="statistics-bar-item"
                :class="{ 'statistics-bar-item--top1': index === 0, 'statistics-bar-item--top2': index === 1, 'statistics-bar-item--top3': index === 2 }"
              >
                <div class="statistics-bar-label">
                  <span class="statistics-bar-rank" :class="{ 'statistics-bar-rank--top3': index < 3 }">{{ index + 1 }}</span>
                  <div class="statistics-bar-dish-info">
                    <img
                      v-if="item.imageUrl"
                      :src="dishThumbSrc(item.imageUrl)"
                      :alt="item.dishName"
                      class="statistics-bar-thumb"
                    />
                    <div v-else class="statistics-bar-thumb statistics-bar-thumb--placeholder">
                      {{ item.dishName.slice(0, 1) }}
                    </div>
                    <div class="statistics-bar-dish-name" :title="item.dishName">{{ item.dishName }}</div>
                  </div>
                </div>
                <div class="statistics-bar-container">
                  <div
                    class="statistics-bar-fill"
                    :class="{ 'statistics-bar-fill--top1': index === 0, 'statistics-bar-fill--top2': index === 1, 'statistics-bar-fill--top3': index === 2 }"
                    :style="{ width: `${(item.orderCount / getMaxOrderCount()) * 100}%` }"
                  ></div>
                  <div class="statistics-bar-value">
                    <span class="statistics-bar-count">{{ item.orderCount }}</span>
                    <span class="statistics-bar-qty">({{ item.totalQuantity }})</span>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </template>
      </div>

      <template #footer>
        <div class=":uno: px-4 py-3">
          <VPagination
            v-show="viewMode === 'list' && getCurrentItems().length > 0"
            v-model:page="page"
            v-model:size="size"
            :total="getCurrentItems().length"
            :size-options="[20, 30, 50, 100]"
            @change="syncQuery"
          />
        </div>
      </template>
    </VCard>
  </div>
</template>

<style scoped>
.statistics-page :deep(.halo-card) {
  border-radius: 12px;
}

.statistics-summary-cards {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 1rem;
  margin-bottom: 1rem;
}

.summary-card {
  background: white;
  border-radius: 12px;
  padding: 1.25rem;
  display: flex;
  align-items: center;
  gap: 1rem;
  box-shadow: 0 2px 8px rgb(15 23 42 / 0.08);
  transition: all 0.2s;
}

.summary-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 16px rgb(15 23 42 / 0.12);
}

.summary-card-icon {
  width: 3rem;
  height: 3rem;
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex: 0 0 auto;
}

.summary-card-icon--blue {
  background: linear-gradient(135deg, rgb(96 165 250) 0%, rgb(59 130 246) 100%);
  color: white;
}

.summary-card-icon--green {
  background: linear-gradient(135deg, rgb(74 222 128) 0%, rgb(34 197 94) 100%);
  color: white;
}

.summary-card-icon--purple {
  background: linear-gradient(135deg, rgb(167 139 250) 0%, rgb(139 92 246) 100%);
  color: white;
}

.summary-card-icon--orange {
  background: linear-gradient(135deg, rgb(251 146 60) 0%, rgb(249 115 22) 100%);
  color: white;
}

.summary-card-content {
  flex: 1;
  min-width: 0;
}

.summary-card-label {
  font-size: 0.875rem;
  color: rgb(113 113 122);
  margin-bottom: 0.25rem;
}

.summary-card-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: rgb(24 24 27);
  line-height: 1;
}

@media (max-width: 1024px) {
  .statistics-summary-cards {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

@media (max-width: 640px) {
  .statistics-summary-cards {
    grid-template-columns: 1fr;
  }
}

.statistics-main-card {
  box-shadow: 0 4px 18px rgb(15 23 42 / 0.06);
}

.statistics-main-card :deep(.card-header),
.statistics-main-card :deep(.card-footer) {
  padding: 0;
}

.statistics-label {
  margin-bottom: 0.25rem;
  font-size: 0.875rem;
  font-weight: 500;
  color: rgb(63 63 70);
}

.statistics-input {
  height: 2.25rem;
  border: 1px solid rgb(212 212 216);
  border-radius: 6px;
  padding: 0 0.75rem;
  font-size: 0.875rem;
}

.statistics-input-sm {
  width: 5rem;
}

.statistics-search-btn {
  background: rgb(24 24 27) !important;
  border-color: rgb(24 24 27) !important;
  color: #fff !important;
}

.statistics-meta {
  font-size: 0.875rem;
  color: rgb(113 113 122);
}

.statistics-error {
  margin-bottom: 0.75rem;
  border: 1px solid rgb(254 202 202);
  background: rgb(254 242 242);
  color: rgb(153 27 27);
  border-radius: 8px;
  padding: 0.75rem;
  font-size: 0.875rem;
}

.statistics-tabs {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
  border-bottom: 1px solid rgb(228 228 231);
  padding-bottom: 0.5rem;
}

.statistics-tab-btn {
  padding: 0.5rem 1rem;
  border: none;
  background: transparent;
  color: rgb(113 113 122);
  font-size: 0.875rem;
  font-weight: 500;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
}

.statistics-tab-btn:hover {
  background: rgb(244 244 245);
  color: rgb(24 24 27);
}

.statistics-tab-btn--active {
  background: rgb(24 24 27);
  color: white;
}

.statistics-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.875rem;
}

.statistics-table thead tr {
  border-bottom: 1px solid rgb(228 228 231);
  color: rgb(113 113 122);
}

.statistics-table th,
.statistics-table td {
  text-align: left;
  padding: 0.75rem 0.5rem;
}

.statistics-table tbody tr {
  border-bottom: 1px solid rgb(244 244 245);
}

.statistics-table tbody tr:hover {
  background: rgb(249 250 251);
}

.statistics-rank-cell {
  text-align: center;
}

.statistics-rank-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 1.75rem;
  height: 1.75rem;
  border-radius: 999px;
  font-size: 0.75rem;
  font-weight: 600;
  color: rgb(113 113 122);
  background: rgb(244 244 245);
}

.statistics-rank-badge--top3 {
  background: linear-gradient(135deg, rgb(251 191 36) 0%, rgb(245 158 11) 100%);
  color: white;
}

.statistics-dish-cell {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  min-width: 0;
}

.statistics-dish-thumb {
  width: 2.5rem;
  height: 2.5rem;
  border-radius: 0.375rem;
  object-fit: cover;
  background: rgb(244 244 245);
  flex: 0 0 auto;
}

.statistics-dish-thumb--placeholder {
  display: flex;
  align-items: center;
  justify-content: center;
  color: rgb(113 113 122);
  font-size: 0.875rem;
  font-weight: 600;
}

.statistics-dish-name {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-weight: 500;
}

.statistics-count-cell {
  font-weight: 600;
  color: rgb(24 24 27);
}

.statistics-chart-view {
  display: flex;
  flex-direction: column;
  gap: 0.875rem;
}

.statistics-bar-item {
  display: flex;
  align-items: center;
  gap: 1rem;
  animation: slideInLeft 0.4s ease-out backwards;
  transition: all 0.2s ease;
}

.statistics-bar-item:hover {
  transform: translateX(4px);
}

.statistics-bar-item--top1 {
  animation-delay: 0.05s;
}

.statistics-bar-item--top2 {
  animation-delay: 0.1s;
}

.statistics-bar-item--top3 {
  animation-delay: 0.15s;
}

@keyframes slideInLeft {
  from {
    opacity: 0;
    transform: translateX(-20px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.statistics-bar-label {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  min-width: 12rem;
  flex: 0 0 auto;
}

.statistics-bar-rank {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 1.5rem;
  height: 1.5rem;
  border-radius: 999px;
  font-size: 0.75rem;
  font-weight: 600;
  color: rgb(113 113 122);
  background: rgb(244 244 245);
  flex: 0 0 auto;
  transition: all 0.2s;
}

.statistics-bar-rank--top3 {
  background: linear-gradient(135deg, rgb(251 191 36) 0%, rgb(245 158 11) 100%);
  color: white;
  box-shadow: 0 2px 8px rgb(251 191 36 / 0.3);
}

.statistics-bar-dish-info {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  min-width: 0;
  flex: 1;
}

.statistics-bar-thumb {
  width: 2rem;
  height: 2rem;
  border-radius: 0.375rem;
  object-fit: cover;
  background: rgb(244 244 245);
  flex: 0 0 auto;
}

.statistics-bar-thumb--placeholder {
  display: flex;
  align-items: center;
  justify-content: center;
  color: rgb(113 113 122);
  font-size: 0.75rem;
  font-weight: 600;
}

.statistics-bar-dish-name {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 0.875rem;
}

.statistics-bar-container {
  flex: 1;
  display: flex;
  align-items: center;
  gap: 0.75rem;
  height: 2rem;
  background: rgb(244 244 245);
  border-radius: 0.375rem;
  padding: 0 0.75rem;
  position: relative;
}

.statistics-bar-fill {
  height: 100%;
  background: linear-gradient(90deg, rgb(96 165 250) 0%, rgb(59 130 246) 100%);
  border-radius: 0.25rem;
  transition: width 0.6s cubic-bezier(0.4, 0, 0.2, 1);
  min-width: 2px;
  position: relative;
  overflow: hidden;
}

.statistics-bar-fill::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(
    90deg,
    transparent 0%,
    rgba(255, 255, 255, 0.2) 50%,
    transparent 100%
  );
  animation: shimmer 2s infinite;
}

@keyframes shimmer {
  0% {
    transform: translateX(-100%);
  }
  100% {
    transform: translateX(100%);
  }
}

.statistics-bar-fill--top1 {
  background: linear-gradient(90deg, rgb(251 191 36) 0%, rgb(245 158 11) 100%);
  box-shadow: 0 2px 12px rgb(251 191 36 / 0.4);
}

.statistics-bar-fill--top2 {
  background: linear-gradient(90deg, rgb(156 163 175) 0%, rgb(107 114 128) 100%);
  box-shadow: 0 2px 12px rgb(156 163 175 / 0.3);
}

.statistics-bar-fill--top3 {
  background: linear-gradient(90deg, rgb(217 119 6) 0%, rgb(180 83 9) 100%);
  box-shadow: 0 2px 12px rgb(217 119 6 / 0.3);
}

.statistics-bar-value {
  position: absolute;
  right: 0.75rem;
  font-size: 0.875rem;
  font-weight: 600;
  color: rgb(24 24 27);
  display: flex;
  align-items: baseline;
  gap: 0.25rem;
}

.statistics-bar-count {
  font-size: 1rem;
  font-weight: 700;
}

.statistics-bar-qty {
  font-size: 0.75rem;
  color: rgb(113 113 122);
  font-weight: 500;
}

@media (max-width: 768px) {
  .statistics-bar-item {
    flex-direction: column;
    align-items: stretch;
    gap: 0.5rem;
  }

  .statistics-bar-label {
    min-width: auto;
    width: 100%;
  }

  .statistics-bar-container {
    height: 1.75rem;
  }

  .statistics-bar-value {
    right: 0.5rem;
  }

  .statistics-bar-count {
    font-size: 0.875rem;
  }

  .statistics-bar-qty {
    font-size: 0.7rem;
  }
}
</style>
