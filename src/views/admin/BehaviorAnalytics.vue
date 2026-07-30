<script setup lang="ts">
import { computed, onMounted, reactive, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import {
  Activity,
  AlertTriangle,
  ChevronDown,
  ChevronRight,
  Clock3,
  MousePointerClick,
  RefreshCw,
  Route,
  SearchX,
  TicketCheck,
  UserRoundSearch,
} from 'lucide-vue-next'
import { adminAPI } from '@/api/admin'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { getLocalizedText } from '@/utils/format'

interface BehaviorOverview {
  range: string
  from: string
  to: string
  timezone: string
  kpi: {
    visitors: number
    sessions: number
    page_views: number
    product_view_sessions: number
    cart_action_sessions: number
    checkout_sessions: number
    coupon_applied_sessions: number
    order_sessions: number
    paid_sessions: number
    in_progress_sessions: number
    checkout_to_paid_rate: string
  }
  funnel: Array<{ key: string; sessions: number; conversion_from_prior: string }>
  dropoffs: Array<{ reason: string; sessions: number; share: string }>
  coupon: {
    entered_sessions: number
    applied_sessions: number
    rejected_sessions: number
    ordered_sessions: number
    paid_sessions: number
    abandoned_sessions: number
    pending_sessions: number
    coupons: Array<{
      coupon_id: number
      coupon_code: string
      applied_sessions: number
      ordered_sessions: number
      paid_sessions: number
      abandoned_sessions: number
    }>
  }
  top_clicks: Array<{
    element_key: string
    element_text: string
    element_selector: string
    page_path: string
    clicks: number
    sessions: number
  }>
  top_products: Array<{
    product_id: number
    title: Record<string, unknown>
    view_sessions: number
    intent_sessions: number
    checkout_sessions: number
    order_sessions: number
    paid_sessions: number
  }>
}

interface BehaviorSessionUser {
  id: number
  email: string
  display_name: string
}

interface BehaviorSessionSummary {
  session_id: string
  visitor_id: string
  user?: BehaviorSessionUser
  guest_email?: string
  client_ip?: string
  user_agent?: string
  started_at: string
  last_at: string
  duration_ms: number
  event_count: number
  last_page: string
  max_scroll_depth: number
  stage: string
  dropoff_reason?: string
  is_paid: boolean
  product_ids: number[]
  coupon_codes: string[]
  order_nos: string[]
  order_statuses: Record<string, string>
}

interface BehaviorSessionEvent {
  id: number
  event_name: string
  page_path?: string
  page_url?: string
  referrer?: string
  element_key?: string
  element_text?: string
  element_selector?: string
  product_id?: number
  sku_id?: number
  order_no?: string
  payment_id?: number
  coupon_id?: number
  coupon_code?: string
  reason?: string
  duration_ms?: number
  scroll_depth?: number
  properties?: Record<string, unknown>
  occurred_at: string
}

interface BehaviorSessionDetail {
  summary: BehaviorSessionSummary
  events: BehaviorSessionEvent[]
}

const zhCopy = {
  title: '用户行为分析',
  subtitle: '从点击到支付，定位优惠券、结算和支付环节的真实流失点。',
  evidence: '第一方行为证据',
  evidenceHint: '登录用户、游客邮箱、IP、设备、点击选择器、停留时长与原始错误均可下钻查看。',
  range: '统计周期',
  today: '今天',
  last7: '最近 7 天',
  last30: '最近 30 天',
  custom: '自定义',
  from: '开始日期',
  to: '结束日期',
  refresh: '刷新数据',
  loading: '正在拼接用户轨迹…',
  fetchFailed: '获取行为分析数据失败',
  sessions: '会话',
  visitors: '访客',
  pageViews: '页面浏览',
  checkoutPaid: '结算 → 支付',
  inProgress: '仍在购买中',
  funnelTitle: '购买漏斗',
  funnelHint: '按唯一会话计数；同一用户重复点击不会放大漏斗。',
  priorConversion: '较上一步',
  dropoffTitle: '最可能的流失原因',
  dropoffHint: '仅统计连续 30 分钟无新动作的会话，避免误判正在付款的人。',
  noDropoff: '当前周期还没有可归因的流失会话。',
  couponTitle: '优惠券诊断',
  couponHint: '重点区分“输入了券”“券可用”“下了单”和“最终付款”。',
  entered: '输入优惠码',
  applied: '成功抵扣',
  rejected: '券被拒绝',
  ordered: '用券下单',
  paid: '用券付款',
  abandoned: '抵扣后未买',
  pending: '仍在进行',
  couponCode: '优惠码',
  noCoupons: '暂时没有优惠券行为数据。',
  clicksTitle: '点击热点',
  clicksHint: '显示用户实际点击的文字、页面和 DOM 选择器。',
  clicks: '点击',
  clickSessions: '涉及会话',
  noClicks: '暂无点击数据。',
  productsTitle: '商品漏斗',
  product: '商品',
  viewed: '浏览',
  intent: '意向',
  checkout: '结算',
  noProducts: '暂无商品行为数据。',
  journeysTitle: '最近用户轨迹',
  journeysHint: '展开任意会话，可按时间查看每一次点击、错误、优惠券和支付动作。',
  identity: '用户 / 游客',
  stage: '最后阶段',
  lastSeen: '最后活动',
  events: '事件',
  evidenceColumn: '关联证据',
  viewJourney: '查看轨迹',
  hideJourney: '收起轨迹',
  noSessions: '当前周期尚未采集到会话。上线后数据会从零开始积累。',
  anonymous: '匿名访客',
  visitor: '访客',
  duration: '停留',
  scroll: '最大滚动',
  referrer: '来源',
  selector: '选择器',
  rawData: '原始属性',
  page: '页面',
  reason: '原因',
  previous: '上一页',
  next: '下一页',
  pageInfo: '第 {page} / {total} 页，共 {count} 个会话',
}

const enCopy = {
  ...zhCopy,
  title: 'Behavior Analytics',
  subtitle: 'Trace the real drop-off points from click through payment.',
  evidence: 'First-party evidence',
  evidenceHint: 'Drill into users, guest emails, IPs, devices, selectors, dwell time, and raw errors.',
  range: 'Range', today: 'Today', last7: 'Last 7 days', last30: 'Last 30 days', custom: 'Custom',
  from: 'From', to: 'To', refresh: 'Refresh', loading: 'Reconstructing journeys…', fetchFailed: 'Failed to load behavior analytics',
  sessions: 'Sessions', visitors: 'Visitors', pageViews: 'Page views', checkoutPaid: 'Checkout → paid', inProgress: 'In progress',
  funnelTitle: 'Purchase funnel', funnelHint: 'Unique sessions; repeated clicks do not inflate the funnel.', priorConversion: 'from prior step',
  dropoffTitle: 'Most likely drop-off reasons', dropoffHint: 'Only sessions inactive for 30 minutes are classified as drop-offs.', noDropoff: 'No attributable drop-offs in this period.',
  couponTitle: 'Coupon diagnosis', couponHint: 'Separate entered, accepted, ordered, and ultimately paid.', entered: 'Code entered', applied: 'Discount applied', rejected: 'Rejected', ordered: 'Ordered', paid: 'Paid', abandoned: 'Applied, not paid', pending: 'In progress', couponCode: 'Coupon', noCoupons: 'No coupon activity yet.',
  clicksTitle: 'Click hotspots', clicksHint: 'Actual text, page, and DOM selector clicked by users.', clicks: 'Clicks', clickSessions: 'Sessions', noClicks: 'No click data yet.',
  productsTitle: 'Product funnel', product: 'Product', viewed: 'Viewed', intent: 'Intent', checkout: 'Checkout', noProducts: 'No product behavior yet.',
  journeysTitle: 'Recent journeys', journeysHint: 'Expand a session to inspect every click, error, coupon, and payment action.', identity: 'User / guest', stage: 'Last stage', lastSeen: 'Last activity', events: 'Events', evidenceColumn: 'Evidence', viewJourney: 'View journey', hideJourney: 'Hide journey', noSessions: 'No sessions collected in this period yet.', anonymous: 'Anonymous visitor', visitor: 'Visitor', duration: 'Duration', scroll: 'Max scroll', referrer: 'Referrer', selector: 'Selector', rawData: 'Raw properties', page: 'Page', reason: 'Reason', previous: 'Previous', next: 'Next', pageInfo: 'Page {page} / {total}, {count} sessions',
}

const { locale } = useI18n()
const copy = computed(() => String(locale.value).toLowerCase().startsWith('en') ? enCopy : zhCopy)

const overview = ref<BehaviorOverview | null>(null)
const sessions = ref<BehaviorSessionSummary[]>([])
const loading = ref(false)
const error = ref('')
const sessionPage = ref(1)
const sessionPageSize = 20
const sessionTotal = ref(0)
const sessionTotalPage = ref(1)
const expandedSessionID = ref('')
const detailLoading = ref('')
const sessionDetails = reactive<Record<string, BehaviorSessionDetail>>({})

const filters = reactive({
  range: '7d',
  from: '',
  to: '',
})

const rangeOptions = computed(() => [
  { value: 'today', label: copy.value.today },
  { value: '7d', label: copy.value.last7 },
  { value: '30d', label: copy.value.last30 },
  { value: 'custom', label: copy.value.custom },
])

const isCustomRange = computed(() => filters.range === 'custom')
const maxFunnelSessions = computed(() => Math.max(overview.value?.funnel?.[0]?.sessions || 0, 1))
const maxDropoffSessions = computed(() => Math.max(...(overview.value?.dropoffs || []).map(item => item.sessions), 1))

const funnelLabels: Record<string, string> = {
  sessions: '进入站点',
  product_view: '查看商品',
  purchase_intent: '加购 / 立即买',
  checkout: '进入结算',
  order_created: '创建订单',
  paid: '支付成功',
}

const funnelLabelsEN: Record<string, string> = {
  sessions: 'Site sessions', product_view: 'Product viewed', purchase_intent: 'Purchase intent', checkout: 'Checkout', order_created: 'Order created', paid: 'Paid',
}

const stageLabels: Record<string, string> = {
  browsing: '仅浏览', product: '商品页', cart: '购物车', checkout: '结算页', ordered: '已下单', payment: '支付页', paid: '已支付',
}

const stageLabelsEN: Record<string, string> = {
  browsing: 'Browsing', product: 'Product', cart: 'Cart', checkout: 'Checkout', ordered: 'Ordered', payment: 'Payment', paid: 'Paid',
}

const reasonLabels: Record<string, string> = {
  in_progress: '仍在操作',
  browse_only: '浏览后离开',
  product_no_action: '看了商品但没有购买动作',
  cart_no_checkout: '加购后未进入结算',
  checkout_no_submit: '进入结算但未提交',
  checkout_submit_no_order: '点击提交但未创建订单',
  order_created_no_payment: '已创建订单但未发起支付',
  payment_not_completed: '已发起支付但未完成',
  order_expired: '订单倒计时过期',
  order_canceled_or_expired: '订单取消或超时',
  coupon_rejected: '优惠券被拒绝',
  checkout_blocked: '结算条件阻止提交',
  order_create_failed: '创建订单失败',
  payment_create_failed: '创建支付失败',
  payment_failed: '支付确认失败',
}

const reasonLabelsEN: Record<string, string> = {
  in_progress: 'Still active', browse_only: 'Left after browsing', product_no_action: 'Viewed product, no action', cart_no_checkout: 'Cart, no checkout', checkout_no_submit: 'Checkout, no submit', checkout_submit_no_order: 'Submitted, no order', order_created_no_payment: 'Order, no payment attempt', payment_not_completed: 'Payment started, not completed', order_expired: 'Order expired', order_canceled_or_expired: 'Order canceled or expired', coupon_rejected: 'Coupon rejected', checkout_blocked: 'Checkout blocked', order_create_failed: 'Order creation failed', payment_create_failed: 'Payment creation failed', payment_failed: 'Payment confirmation failed',
}

const eventLabels: Record<string, string> = {
  page_view: '浏览页面', page_exit: '离开页面', ui_click: '点击控件', scroll_depth: '滚动页面', form_field_focus: '开始填写', form_field_blur: '离开字段', frontend_error: '前端错误', api_error: '接口错误', product_view: '查看商品', product_click: '点击商品', quick_buy_open: '打开快速购买', add_to_cart: '加入购物车', add_to_cart_blocked: '加购受阻', buy_now: '立即购买', buy_now_blocked: '立即购买受阻', cart_view: '查看购物车', cart_remove: '移除商品', cart_restore: '恢复商品', cart_quantity_change: '修改数量', checkout_view: '进入结算', checkout_mode_change: '切换结算身份', checkout_submit: '提交结算', checkout_blocked: '结算受阻', coupon_entered: '输入优惠码', coupon_applied: '优惠券生效', coupon_rejected: '优惠券被拒绝', order_created: '订单创建成功', order_create_failed: '订单创建失败', payment_view: '进入支付页', payment_channel_selected: '选择支付渠道', payment_submit: '提交支付', payment_started: '支付已发起', payment_create_failed: '支付创建失败', payment_link_opened: '打开支付链接', payment_link_copied: '复制支付链接', payment_status: '支付状态变化', payment_success: '支付成功', payment_failed: '支付失败', order_canceled: '订单取消', order_expired: '订单过期',
}

const eventLabelsEN: Record<string, string> = {
  page_view: 'Page view', page_exit: 'Page exit', ui_click: 'UI click', scroll_depth: 'Scroll depth', form_field_focus: 'Field focus', form_field_blur: 'Field blur', frontend_error: 'Frontend error', api_error: 'API error', product_view: 'Product view', product_click: 'Product click', quick_buy_open: 'Quick buy opened', add_to_cart: 'Added to cart', add_to_cart_blocked: 'Add to cart blocked', buy_now: 'Buy now', buy_now_blocked: 'Buy now blocked', cart_view: 'Cart view', cart_remove: 'Cart item removed', cart_restore: 'Cart item restored', cart_quantity_change: 'Quantity changed', checkout_view: 'Checkout view', checkout_mode_change: 'Checkout mode changed', checkout_submit: 'Checkout submitted', checkout_blocked: 'Checkout blocked', coupon_entered: 'Coupon entered', coupon_applied: 'Coupon applied', coupon_rejected: 'Coupon rejected', order_created: 'Order created', order_create_failed: 'Order creation failed', payment_view: 'Payment view', payment_channel_selected: 'Payment channel selected', payment_submit: 'Payment submitted', payment_started: 'Payment started', payment_create_failed: 'Payment creation failed', payment_link_opened: 'Payment link opened', payment_link_copied: 'Payment link copied', payment_status: 'Payment status', payment_success: 'Payment success', payment_failed: 'Payment failed', order_canceled: 'Order canceled', order_expired: 'Order expired',
}

const makeRangeDate = (raw: string, endOfDay: boolean) => {
  if (!raw) return undefined
  const date = new Date(`${raw}${endOfDay ? 'T23:59:59' : 'T00:00:00'}`)
  if (Number.isNaN(date.getTime())) return undefined
  return date.toISOString()
}

const buildQuery = () => {
  const params: Record<string, unknown> = {
    range: filters.range,
    tz: Intl.DateTimeFormat().resolvedOptions().timeZone,
  }
  if (isCustomRange.value) {
    const from = makeRangeDate(filters.from, false)
    const to = makeRangeDate(filters.to, true)
    if (!from || !to) return null
    params.from = from
    params.to = to
  }
  return params
}

const loadData = async () => {
  const query = buildQuery()
  if (!query) return
  loading.value = true
  error.value = ''
  expandedSessionID.value = ''
  try {
    const [overviewResponse, sessionResponse] = await Promise.all([
      adminAPI.getBehaviorAnalyticsOverview(query),
      adminAPI.getBehaviorAnalyticsSessions({ ...query, page: sessionPage.value, page_size: sessionPageSize }),
    ])
    overview.value = overviewResponse.data.data as BehaviorOverview
    sessions.value = (sessionResponse.data.data as BehaviorSessionSummary[]) || []
    sessionTotal.value = Number(sessionResponse.data.pagination?.total || 0)
    sessionTotalPage.value = Math.max(Number(sessionResponse.data.pagination?.total_page || 1), 1)
  } catch (err: any) {
    error.value = err?.message || copy.value.fetchFailed
  } finally {
    loading.value = false
  }
}

const handleRangeChange = (value: unknown) => {
  filters.range = String(value || '7d')
  sessionPage.value = 1
  if (filters.range === 'custom') {
    const today = new Date()
    const start = new Date(today)
    start.setDate(start.getDate() - 6)
    filters.from = start.toISOString().slice(0, 10)
    filters.to = today.toISOString().slice(0, 10)
  } else {
    filters.from = ''
    filters.to = ''
  }
  void loadData()
}

const refresh = () => void loadData()

const goToPage = (page: number) => {
  const next = Math.max(1, Math.min(page, sessionTotalPage.value))
  if (next === sessionPage.value) return
  sessionPage.value = next
  void loadData()
}

const toggleSession = async (session: BehaviorSessionSummary) => {
  if (expandedSessionID.value === session.session_id) {
    expandedSessionID.value = ''
    return
  }
  expandedSessionID.value = session.session_id
  if (sessionDetails[session.session_id]) return
  detailLoading.value = session.session_id
  try {
    const response = await adminAPI.getBehaviorAnalyticsSession(session.session_id)
    sessionDetails[session.session_id] = response.data.data as BehaviorSessionDetail
  } finally {
    detailLoading.value = ''
  }
}

const funnelLabel = (key: string) => (String(locale.value).startsWith('en') ? funnelLabelsEN : funnelLabels)[key] || key
const stageLabel = (stage: string) => (String(locale.value).startsWith('en') ? stageLabelsEN : stageLabels)[stage] || stage
const eventLabel = (event: string) => (String(locale.value).startsWith('en') ? eventLabelsEN : eventLabels)[event] || event

const reasonParts = (reason?: string) => {
  const raw = String(reason || '')
  const parts = raw.split(':')
  const base = parts[0] || ''
  const rest = parts.slice(1)
  const labels = String(locale.value).startsWith('en') ? reasonLabelsEN : reasonLabels
  return { label: labels[base] || base || '-', detail: rest.join(':') }
}

const funnelWidth = (sessions: number) => `${Math.max(3, Math.round((sessions / maxFunnelSessions.value) * 100))}%`
const dropoffWidth = (sessions: number) => `${Math.max(4, Math.round((sessions / maxDropoffSessions.value) * 100))}%`

const formatDate = (value?: string) => {
  if (!value) return '-'
  const date = new Date(value)
  if (Number.isNaN(date.getTime())) return value
  return date.toLocaleString()
}

const formatTime = (value?: string) => {
  if (!value) return '-'
  const date = new Date(value)
  if (Number.isNaN(date.getTime())) return value
  return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' })
}

const formatDuration = (milliseconds?: number) => {
  const totalSeconds = Math.max(Math.round(Number(milliseconds || 0) / 1000), 0)
  if (totalSeconds < 60) return `${totalSeconds}s`
  const minutes = Math.floor(totalSeconds / 60)
  const seconds = totalSeconds % 60
  if (minutes < 60) return `${minutes}m ${seconds}s`
  const hours = Math.floor(minutes / 60)
  return `${hours}h ${minutes % 60}m`
}

const shortID = (value?: string) => {
  const text = String(value || '')
  if (text.length <= 16) return text
  return `${text.slice(0, 8)}…${text.slice(-6)}`
}

const sessionIdentity = (session: BehaviorSessionSummary) => {
  if (session.user) return session.user.display_name || session.user.email || `#${session.user.id}`
  return session.guest_email || copy.value.anonymous
}

const stageClass = (stage: string) => {
  if (stage === 'paid') return 'bg-emerald-500/10 text-emerald-700 dark:text-emerald-300'
  if (stage === 'payment' || stage === 'ordered') return 'bg-amber-500/10 text-amber-700 dark:text-amber-300'
  if (stage === 'checkout') return 'bg-orange-500/10 text-orange-700 dark:text-orange-300'
  return 'bg-muted text-muted-foreground'
}

const reasonClass = (reason?: string) => {
  if (reason === 'in_progress') return 'text-emerald-700 dark:text-emerald-300'
  if (reason?.includes('failed') || reason?.includes('blocked') || reason?.includes('rejected')) return 'text-rose-700 dark:text-rose-300'
  return 'text-amber-700 dark:text-amber-300'
}

const eventTone = (event: string) => {
  if (event === 'payment_success' || event === 'order_created' || event === 'coupon_applied') return 'bg-emerald-500'
  if (event.includes('failed') || event.includes('blocked') || event.includes('rejected') || event === 'api_error') return 'bg-rose-500'
  if (event.includes('payment') || event.includes('checkout') || event.includes('coupon')) return 'bg-amber-500'
  return 'bg-slate-400 dark:bg-slate-500'
}

const sessionEvidence = (session: BehaviorSessionSummary) => {
  const values: string[] = []
  if (session.coupon_codes?.length) values.push(`券 ${session.coupon_codes.join(', ')}`)
  if (session.order_nos?.length) values.push(`订单 ${session.order_nos.join(', ')}`)
  if (session.product_ids?.length) values.push(`商品 #${session.product_ids.join(', #')}`)
  return values
}

const hasRawProperties = (event: BehaviorSessionEvent) => event.properties && Object.keys(event.properties).length > 0
const sessionDetailEvents = (sessionID: string) => sessionDetails[sessionID]?.events || []

onMounted(() => {
  void loadData()
})
</script>

<template>
  <div class="behavior-page mx-auto max-w-[1600px] space-y-8 pb-12">
    <header class="relative overflow-hidden border-b border-border pb-7 pt-1">
      <div class="behavior-grid absolute inset-0 -z-10 opacity-50"></div>
      <div class="flex flex-col gap-6 xl:flex-row xl:items-end xl:justify-between">
        <div class="max-w-3xl">
          <div class="mb-3 flex items-center gap-2 text-xs font-semibold uppercase tracking-[0.18em] text-emerald-700 dark:text-emerald-300">
            <Activity class="h-4 w-4" />
            {{ copy.evidence }}
          </div>
          <h1 class="text-3xl font-semibold tracking-tight sm:text-4xl">{{ copy.title }}</h1>
          <p class="mt-3 max-w-2xl text-sm leading-6 text-muted-foreground">{{ copy.subtitle }}</p>
          <p class="mt-2 max-w-3xl text-xs leading-5 text-muted-foreground/80">{{ copy.evidenceHint }}</p>
        </div>

        <div class="flex flex-col gap-3 sm:flex-row sm:items-end">
          <div class="min-w-[180px]">
            <label class="mb-1.5 block text-xs font-medium text-muted-foreground">{{ copy.range }}</label>
            <Select :model-value="filters.range" @update:model-value="handleRangeChange">
              <SelectTrigger><SelectValue /></SelectTrigger>
              <SelectContent>
                <SelectItem v-for="option in rangeOptions" :key="option.value" :value="option.value">
                  {{ option.label }}
                </SelectItem>
              </SelectContent>
            </Select>
          </div>
          <template v-if="isCustomRange">
            <div>
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">{{ copy.from }}</label>
              <Input v-model="filters.from" type="date" class="w-[160px]" @change="sessionPage = 1; loadData()" />
            </div>
            <div>
              <label class="mb-1.5 block text-xs font-medium text-muted-foreground">{{ copy.to }}</label>
              <Input v-model="filters.to" type="date" class="w-[160px]" @change="sessionPage = 1; loadData()" />
            </div>
          </template>
          <Button variant="outline" class="gap-2" :disabled="loading" @click="refresh">
            <RefreshCw class="h-4 w-4" :class="loading ? 'animate-spin' : ''" />
            {{ copy.refresh }}
          </Button>
        </div>
      </div>
    </header>

    <div v-if="error" class="flex items-start gap-3 rounded-xl border border-rose-500/30 bg-rose-500/10 px-4 py-3 text-sm text-rose-700 dark:text-rose-300">
      <AlertTriangle class="mt-0.5 h-4 w-4 shrink-0" />
      {{ error }}
    </div>

    <div v-if="loading && !overview" class="flex min-h-[320px] items-center justify-center text-sm text-muted-foreground">
      <RefreshCw class="mr-2 h-4 w-4 animate-spin" /> {{ copy.loading }}
    </div>

    <template v-else-if="overview">
      <section class="overflow-hidden rounded-2xl border border-border bg-card">
        <div class="grid border-b border-border lg:grid-cols-[1.35fr_.65fr]">
          <div class="p-6 sm:p-8">
            <div class="flex items-start justify-between gap-5">
              <div>
                <div class="text-xs font-medium uppercase tracking-[0.16em] text-muted-foreground">{{ copy.checkoutPaid }}</div>
                <div class="mt-2 text-5xl font-semibold tracking-[-0.05em] text-emerald-700 dark:text-emerald-300">
                  {{ overview.kpi.checkout_to_paid_rate }}<span class="ml-1 text-2xl">%</span>
                </div>
              </div>
              <Route class="h-8 w-8 text-emerald-600/70" />
            </div>
            <div class="mt-8 grid grid-cols-2 gap-x-8 gap-y-5 sm:grid-cols-4">
              <div>
                <div class="text-2xl font-semibold">{{ overview.kpi.sessions }}</div>
                <div class="mt-1 text-xs text-muted-foreground">{{ copy.sessions }}</div>
              </div>
              <div>
                <div class="text-2xl font-semibold">{{ overview.kpi.visitors }}</div>
                <div class="mt-1 text-xs text-muted-foreground">{{ copy.visitors }}</div>
              </div>
              <div>
                <div class="text-2xl font-semibold">{{ overview.kpi.page_views }}</div>
                <div class="mt-1 text-xs text-muted-foreground">{{ copy.pageViews }}</div>
              </div>
              <div>
                <div class="text-2xl font-semibold text-amber-700 dark:text-amber-300">{{ overview.kpi.in_progress_sessions }}</div>
                <div class="mt-1 text-xs text-muted-foreground">{{ copy.inProgress }}</div>
              </div>
            </div>
          </div>

          <div class="border-t border-border bg-muted/25 p-6 sm:p-8 lg:border-l lg:border-t-0">
            <div class="flex items-center gap-2 text-sm font-semibold">
              <UserRoundSearch class="h-4 w-4" />
              {{ copy.journeysTitle }}
            </div>
            <p class="mt-2 text-xs leading-5 text-muted-foreground">{{ copy.journeysHint }}</p>
            <div class="mt-6 space-y-3 text-sm">
              <div class="flex items-center justify-between border-b border-border/70 pb-3">
                <span class="text-muted-foreground">{{ funnelLabel('product_view') }}</span>
                <span class="font-semibold">{{ overview.kpi.product_view_sessions }}</span>
              </div>
              <div class="flex items-center justify-between border-b border-border/70 pb-3">
                <span class="text-muted-foreground">{{ funnelLabel('checkout') }}</span>
                <span class="font-semibold">{{ overview.kpi.checkout_sessions }}</span>
              </div>
              <div class="flex items-center justify-between">
                <span class="text-muted-foreground">{{ funnelLabel('paid') }}</span>
                <span class="font-semibold text-emerald-700 dark:text-emerald-300">{{ overview.kpi.paid_sessions }}</span>
              </div>
            </div>
          </div>
        </div>

        <div class="p-6 sm:p-8">
          <div class="mb-6 flex flex-col gap-2 sm:flex-row sm:items-end sm:justify-between">
            <div>
              <h2 class="text-lg font-semibold">{{ copy.funnelTitle }}</h2>
              <p class="mt-1 text-xs text-muted-foreground">{{ copy.funnelHint }}</p>
            </div>
          </div>
          <div class="space-y-4">
            <div v-for="(step, index) in overview.funnel" :key="step.key" class="grid items-center gap-3 sm:grid-cols-[150px_1fr_150px]">
              <div class="flex items-center justify-between gap-3 text-sm sm:block">
                <span class="font-medium">{{ funnelLabel(step.key) }}</span>
                <span class="font-semibold sm:hidden">{{ step.sessions }}</span>
              </div>
              <div class="h-9 overflow-hidden rounded-md bg-muted">
                <div
                  class="flex h-full items-center rounded-md px-3 text-xs font-semibold text-white transition-[width] duration-500"
                  :class="index === overview.funnel.length - 1 ? 'bg-emerald-600' : index >= 3 ? 'bg-amber-600' : 'bg-slate-700 dark:bg-slate-500'"
                  :style="{ width: funnelWidth(step.sessions) }"
                >
                  <span class="hidden sm:inline">{{ step.sessions }}</span>
                </div>
              </div>
              <div class="text-xs text-muted-foreground sm:text-right">
                <template v-if="index > 0">{{ copy.priorConversion }} {{ step.conversion_from_prior }}%</template>
                <template v-else>{{ step.sessions }} {{ copy.sessions }}</template>
              </div>
            </div>
          </div>
        </div>
      </section>

      <section class="grid gap-8 xl:grid-cols-[1.05fr_.95fr]">
        <div class="border-t border-border pt-6">
          <div class="flex items-start justify-between gap-4">
            <div>
              <h2 class="text-lg font-semibold">{{ copy.dropoffTitle }}</h2>
              <p class="mt-1 text-xs leading-5 text-muted-foreground">{{ copy.dropoffHint }}</p>
            </div>
            <SearchX class="h-5 w-5 text-rose-500/80" />
          </div>
          <div v-if="overview.dropoffs.length" class="mt-6 space-y-5">
            <div v-for="item in overview.dropoffs" :key="item.reason">
              <div class="mb-2 flex items-start justify-between gap-4 text-sm">
                <div>
                  <div class="font-medium">{{ reasonParts(item.reason).label }}</div>
                  <div v-if="reasonParts(item.reason).detail" class="mt-0.5 max-w-2xl break-words text-xs text-muted-foreground">
                    {{ reasonParts(item.reason).detail }}
                  </div>
                </div>
                <div class="shrink-0 text-right">
                  <div class="font-semibold">{{ item.sessions }}</div>
                  <div class="text-xs text-muted-foreground">{{ item.share }}%</div>
                </div>
              </div>
              <div class="h-1.5 overflow-hidden rounded-full bg-muted">
                <div class="h-full rounded-full bg-rose-500" :style="{ width: dropoffWidth(item.sessions) }"></div>
              </div>
            </div>
          </div>
          <p v-else class="mt-6 text-sm text-muted-foreground">{{ copy.noDropoff }}</p>
        </div>

        <div class="rounded-2xl border border-amber-500/25 bg-amber-500/[0.04] p-6">
          <div class="flex items-start justify-between gap-4">
            <div>
              <h2 class="flex items-center gap-2 text-lg font-semibold">
                <TicketCheck class="h-5 w-5 text-amber-600" />
                {{ copy.couponTitle }}
              </h2>
              <p class="mt-1 text-xs leading-5 text-muted-foreground">{{ copy.couponHint }}</p>
            </div>
          </div>
          <div class="mt-6 grid grid-cols-2 gap-x-6 gap-y-5 sm:grid-cols-3">
            <div><div class="text-2xl font-semibold">{{ overview.coupon.entered_sessions }}</div><div class="mt-1 text-xs text-muted-foreground">{{ copy.entered }}</div></div>
            <div><div class="text-2xl font-semibold text-emerald-700 dark:text-emerald-300">{{ overview.coupon.applied_sessions }}</div><div class="mt-1 text-xs text-muted-foreground">{{ copy.applied }}</div></div>
            <div><div class="text-2xl font-semibold text-rose-700 dark:text-rose-300">{{ overview.coupon.rejected_sessions }}</div><div class="mt-1 text-xs text-muted-foreground">{{ copy.rejected }}</div></div>
            <div><div class="text-2xl font-semibold">{{ overview.coupon.ordered_sessions }}</div><div class="mt-1 text-xs text-muted-foreground">{{ copy.ordered }}</div></div>
            <div><div class="text-2xl font-semibold text-emerald-700 dark:text-emerald-300">{{ overview.coupon.paid_sessions }}</div><div class="mt-1 text-xs text-muted-foreground">{{ copy.paid }}</div></div>
            <div><div class="text-2xl font-semibold text-amber-700 dark:text-amber-300">{{ overview.coupon.abandoned_sessions }}</div><div class="mt-1 text-xs text-muted-foreground">{{ copy.abandoned }}</div></div>
          </div>
          <div class="mt-6 border-t border-amber-500/20 pt-4 text-xs text-muted-foreground">
            {{ copy.pending }}：<span class="font-semibold text-foreground">{{ overview.coupon.pending_sessions }}</span>
          </div>
        </div>
      </section>

      <section v-if="overview.coupon.coupons.length" class="overflow-hidden rounded-2xl border border-border">
        <div class="border-b border-border px-5 py-4">
          <h2 class="font-semibold">{{ copy.couponTitle }}</h2>
        </div>
        <div class="overflow-x-auto">
          <table class="w-full min-w-[720px] text-sm">
            <thead class="bg-muted/40 text-left text-xs text-muted-foreground">
              <tr>
                <th class="px-5 py-3 font-medium">{{ copy.couponCode }}</th>
                <th class="px-5 py-3 text-right font-medium">{{ copy.applied }}</th>
                <th class="px-5 py-3 text-right font-medium">{{ copy.ordered }}</th>
                <th class="px-5 py-3 text-right font-medium">{{ copy.paid }}</th>
                <th class="px-5 py-3 text-right font-medium">{{ copy.abandoned }}</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-border">
              <tr v-for="coupon in overview.coupon.coupons" :key="coupon.coupon_id" class="hover:bg-muted/25">
                <td class="px-5 py-3 font-mono text-xs">{{ coupon.coupon_code || `#${coupon.coupon_id}` }}</td>
                <td class="px-5 py-3 text-right">{{ coupon.applied_sessions }}</td>
                <td class="px-5 py-3 text-right">{{ coupon.ordered_sessions }}</td>
                <td class="px-5 py-3 text-right text-emerald-700 dark:text-emerald-300">{{ coupon.paid_sessions }}</td>
                <td class="px-5 py-3 text-right text-amber-700 dark:text-amber-300">{{ coupon.abandoned_sessions }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>

      <section class="grid gap-8 xl:grid-cols-2">
        <div class="border-t border-border pt-6">
          <div class="flex items-start justify-between gap-4">
            <div>
              <h2 class="text-lg font-semibold">{{ copy.clicksTitle }}</h2>
              <p class="mt-1 text-xs text-muted-foreground">{{ copy.clicksHint }}</p>
            </div>
            <MousePointerClick class="h-5 w-5 text-slate-500" />
          </div>
          <div v-if="overview.top_clicks.length" class="mt-5 divide-y divide-border">
            <div v-for="item in overview.top_clicks" :key="`${item.page_path}:${item.element_key}:${item.element_selector}`" class="py-4 first:pt-0">
              <div class="flex items-start justify-between gap-5">
                <div class="min-w-0">
                  <div class="truncate text-sm font-medium">{{ item.element_text || item.element_key || item.element_selector }}</div>
                  <div class="mt-1 truncate font-mono text-[11px] text-muted-foreground">{{ item.page_path }} · {{ item.element_selector || item.element_key }}</div>
                </div>
                <div class="shrink-0 text-right text-xs">
                  <div class="font-semibold">{{ item.clicks }} {{ copy.clicks }}</div>
                  <div class="mt-1 text-muted-foreground">{{ item.sessions }} {{ copy.clickSessions }}</div>
                </div>
              </div>
            </div>
          </div>
          <p v-else class="mt-5 text-sm text-muted-foreground">{{ copy.noClicks }}</p>
        </div>

        <div class="border-t border-border pt-6">
          <h2 class="text-lg font-semibold">{{ copy.productsTitle }}</h2>
          <div v-if="overview.top_products.length" class="mt-5 overflow-x-auto">
            <table class="w-full min-w-[620px] text-sm">
              <thead class="text-left text-xs text-muted-foreground">
                <tr class="border-b border-border">
                  <th class="pb-3 font-medium">{{ copy.product }}</th>
                  <th class="pb-3 text-right font-medium">{{ copy.viewed }}</th>
                  <th class="pb-3 text-right font-medium">{{ copy.intent }}</th>
                  <th class="pb-3 text-right font-medium">{{ copy.checkout }}</th>
                  <th class="pb-3 text-right font-medium">{{ copy.paid }}</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-border">
                <tr v-for="item in overview.top_products" :key="item.product_id">
                  <td class="py-3 pr-4"><span class="font-medium">{{ getLocalizedText(item.title) || `#${item.product_id}` }}</span><span class="ml-2 text-xs text-muted-foreground">#{{ item.product_id }}</span></td>
                  <td class="py-3 text-right">{{ item.view_sessions }}</td>
                  <td class="py-3 text-right">{{ item.intent_sessions }}</td>
                  <td class="py-3 text-right">{{ item.checkout_sessions }}</td>
                  <td class="py-3 text-right font-semibold text-emerald-700 dark:text-emerald-300">{{ item.paid_sessions }}</td>
                </tr>
              </tbody>
            </table>
          </div>
          <p v-else class="mt-5 text-sm text-muted-foreground">{{ copy.noProducts }}</p>
        </div>
      </section>

      <section class="overflow-hidden rounded-2xl border border-border bg-card">
        <div class="flex flex-col gap-3 border-b border-border px-5 py-5 sm:flex-row sm:items-end sm:justify-between">
          <div>
            <h2 class="text-lg font-semibold">{{ copy.journeysTitle }}</h2>
            <p class="mt-1 text-xs text-muted-foreground">{{ copy.journeysHint }}</p>
          </div>
          <div class="text-xs text-muted-foreground">
            {{ copy.pageInfo.replace('{page}', String(sessionPage)).replace('{total}', String(sessionTotalPage)).replace('{count}', String(sessionTotal)) }}
          </div>
        </div>

        <div v-if="sessions.length" class="divide-y divide-border">
          <div v-for="session in sessions" :key="session.session_id">
            <button
              type="button"
              class="grid w-full gap-4 px-5 py-4 text-left transition-colors hover:bg-muted/25 lg:grid-cols-[1.25fr_.7fr_.9fr_.45fr_1.35fr_auto] lg:items-center"
              @click="toggleSession(session)"
            >
              <div class="min-w-0">
                <div class="flex items-center gap-2">
                  <ChevronDown v-if="expandedSessionID === session.session_id" class="h-4 w-4 shrink-0" />
                  <ChevronRight v-else class="h-4 w-4 shrink-0 text-muted-foreground" />
                  <div class="truncate font-medium">{{ sessionIdentity(session) }}</div>
                </div>
                <div class="ml-6 mt-1 truncate font-mono text-[11px] text-muted-foreground">{{ shortID(session.session_id) }} · {{ session.client_ip || '-' }}</div>
              </div>
              <div>
                <span class="inline-flex rounded-full px-2.5 py-1 text-xs font-medium" :class="stageClass(session.stage)">{{ stageLabel(session.stage) }}</span>
                <div class="mt-1 text-xs" :class="reasonClass(session.dropoff_reason)">{{ reasonParts(session.dropoff_reason).label }}</div>
              </div>
              <div class="text-xs text-muted-foreground">
                <div>{{ formatDate(session.last_at) }}</div>
                <div class="mt-1 flex items-center gap-3"><span>{{ copy.duration }} {{ formatDuration(session.duration_ms) }}</span><span>{{ copy.scroll }} {{ session.max_scroll_depth }}%</span></div>
              </div>
              <div class="text-sm font-semibold">{{ session.event_count }}</div>
              <div class="min-w-0 text-xs text-muted-foreground">
                <div class="truncate">{{ session.last_page || '-' }}</div>
                <div v-for="evidence in sessionEvidence(session).slice(0, 2)" :key="evidence" class="mt-1 truncate">{{ evidence }}</div>
              </div>
              <div class="text-xs font-medium text-primary">{{ expandedSessionID === session.session_id ? copy.hideJourney : copy.viewJourney }}</div>
            </button>

            <div v-if="expandedSessionID === session.session_id" class="border-t border-border bg-muted/15 px-5 py-6">
              <div v-if="detailLoading === session.session_id" class="flex items-center py-8 text-sm text-muted-foreground">
                <RefreshCw class="mr-2 h-4 w-4 animate-spin" /> {{ copy.loading }}
              </div>
              <template v-else-if="sessionDetails[session.session_id]">
                <div class="grid gap-4 border-b border-border pb-5 text-xs sm:grid-cols-2 xl:grid-cols-4">
                  <div><div class="text-muted-foreground">{{ copy.identity }}</div><div class="mt-1 break-all font-medium">{{ sessionIdentity(session) }}</div></div>
                  <div><div class="text-muted-foreground">IP / User-Agent</div><div class="mt-1 break-all font-mono text-[11px]">{{ session.client_ip || '-' }}<br>{{ session.user_agent || '-' }}</div></div>
                  <div><div class="text-muted-foreground">{{ copy.visitor }}</div><div class="mt-1 break-all font-mono text-[11px]">{{ session.visitor_id }}</div></div>
                  <div><div class="text-muted-foreground">Session</div><div class="mt-1 break-all font-mono text-[11px]">{{ session.session_id }}</div></div>
                </div>

                <div class="relative mt-6 space-y-0">
                  <div class="absolute bottom-3 left-[88px] top-3 w-px bg-border"></div>
                  <div v-for="event in sessionDetailEvents(session.session_id)" :key="event.id" class="relative grid grid-cols-[72px_20px_1fr] gap-3 py-3">
                    <div class="pt-0.5 text-right font-mono text-[11px] text-muted-foreground">{{ formatTime(event.occurred_at) }}</div>
                    <div class="relative z-10 mt-1 h-2.5 w-2.5 rounded-full ring-4 ring-background" :class="eventTone(event.event_name)"></div>
                    <div class="min-w-0 rounded-lg border border-border bg-background px-4 py-3">
                      <div class="flex flex-col gap-2 sm:flex-row sm:items-start sm:justify-between">
                        <div>
                          <div class="text-sm font-semibold">{{ eventLabel(event.event_name) }}</div>
                          <div v-if="event.element_text || event.element_key" class="mt-1 text-xs text-foreground/80">{{ event.element_text || event.element_key }}</div>
                        </div>
                        <div class="flex flex-wrap gap-1.5 text-[10px] text-muted-foreground">
                          <span v-if="event.product_id" class="rounded bg-muted px-2 py-1">商品 #{{ event.product_id }}</span>
                          <span v-if="event.order_no" class="rounded bg-muted px-2 py-1">订单 {{ event.order_no }}</span>
                          <span v-if="event.coupon_code" class="rounded bg-amber-500/10 px-2 py-1 text-amber-700 dark:text-amber-300">券 {{ event.coupon_code }}</span>
                          <span v-if="event.scroll_depth" class="rounded bg-muted px-2 py-1">滚动 {{ event.scroll_depth }}%</span>
                          <span v-if="event.duration_ms" class="rounded bg-muted px-2 py-1">{{ formatDuration(event.duration_ms) }}</span>
                        </div>
                      </div>
                      <div class="mt-3 grid gap-2 text-[11px] text-muted-foreground">
                        <div v-if="event.page_url || event.page_path"><span class="font-medium text-foreground/70">{{ copy.page }}：</span><span class="break-all font-mono">{{ event.page_url || event.page_path }}</span></div>
                        <div v-if="event.referrer"><span class="font-medium text-foreground/70">{{ copy.referrer }}：</span><span class="break-all font-mono">{{ event.referrer }}</span></div>
                        <div v-if="event.element_selector"><span class="font-medium text-foreground/70">{{ copy.selector }}：</span><span class="break-all font-mono">{{ event.element_selector }}</span></div>
                        <div v-if="event.reason"><span class="font-medium text-foreground/70">{{ copy.reason }}：</span><span class="break-words">{{ event.reason }}</span></div>
                      </div>
                      <details v-if="hasRawProperties(event)" class="mt-3 text-[11px]">
                        <summary class="cursor-pointer text-muted-foreground hover:text-foreground">{{ copy.rawData }}</summary>
                        <pre class="mt-2 max-h-64 overflow-auto whitespace-pre-wrap break-all rounded-md bg-muted p-3 font-mono text-[10px] leading-5">{{ JSON.stringify(event.properties, null, 2) }}</pre>
                      </details>
                    </div>
                  </div>
                </div>
              </template>
            </div>
          </div>
        </div>
        <div v-else class="flex min-h-[220px] flex-col items-center justify-center px-6 text-center">
          <Clock3 class="h-8 w-8 text-muted-foreground/60" />
          <p class="mt-3 max-w-lg text-sm text-muted-foreground">{{ copy.noSessions }}</p>
        </div>

        <div v-if="sessionTotalPage > 1" class="flex items-center justify-between border-t border-border px-5 py-4">
          <Button variant="outline" size="sm" :disabled="sessionPage <= 1" @click="goToPage(sessionPage - 1)">{{ copy.previous }}</Button>
          <span class="text-xs text-muted-foreground">{{ sessionPage }} / {{ sessionTotalPage }}</span>
          <Button variant="outline" size="sm" :disabled="sessionPage >= sessionTotalPage" @click="goToPage(sessionPage + 1)">{{ copy.next }}</Button>
        </div>
      </section>
    </template>
  </div>
</template>

<style scoped>
.behavior-grid {
  background-image:
    linear-gradient(hsl(var(--border) / 0.32) 1px, transparent 1px),
    linear-gradient(90deg, hsl(var(--border) / 0.32) 1px, transparent 1px);
  background-size: 28px 28px;
  mask-image: linear-gradient(to right, black, transparent 88%);
}
</style>
