<template>
  <div class="card">
    <div v-if="history && history.stats" class="stats-panel">
      <div class="stat-card green">
        <div class="stat-num">{{ history.stats.answeredCount }}</div>
        <div class="stat-label">✓ 已回答天数</div>
      </div>
      <div class="stat-card red">
        <div class="stat-num">{{ history.stats.missedCount }}</div>
        <div class="stat-label">✗ 断更天数</div>
      </div>
      <div class="stat-card blue">
        <div class="stat-num">
          {{ history.stats.totalDays > 0 ? Math.round(history.stats.answeredCount / history.stats.totalDays * 100) : 0 }}%
        </div>
        <div class="stat-label">📊 坚持率</div>
      </div>
    </div>

    <div class="search-container">
      <div class="search-input-wrapper">
        <span class="search-icon">🔍</span>
        <input
          v-model="searchKeyword"
          type="text"
          class="search-input"
          placeholder="输入关键词搜索问题或回答..."
          @input="handleSearch"
          @keyup.enter="handleSearch"
        />
        <button v-if="searchKeyword" class="search-clear" @click="clearSearch">×</button>
      </div>
      <div v-if="searching" class="search-status">搜索中...</div>
      <div v-else-if="searchKeyword && !searchResults.length" class="search-status empty">
        未找到包含 "{{ searchKeyword }}" 的记录
      </div>
    </div>

    <div v-if="searchResults.length > 0" class="search-results">
      <div class="search-results-header">
        <span>🔎 找到 {{ searchResults.length }} 条结果</span>
        <button class="close-search-btn" @click="clearSearch">返回日历</button>
      </div>
      <div
        v-for="(item, idx) in searchResults"
        :key="idx"
        class="search-result-item"
        @click="jumpToDate(item.date)"
      >
        <div class="search-result-date">
          <span class="date-badge" :class="{ answered: item.answered }">
            {{ item.answered ? '✓' : '○' }} {{ formatFullDate(item.date) }}
          </span>
        </div>
        <div class="search-result-q">
          <span class="result-label">问题：</span>
          <span v-html="highlightText(item.question, searchKeyword)"></span>
        </div>
        <div v-if="item.answer" class="search-result-a">
          <span class="result-label">回答：</span>
          <span v-html="highlightText(item.answer, searchKeyword)"></span>
        </div>
        <div v-else class="search-result-empty">暂无回答</div>
      </div>
    </div>

    <template v-else>
      <div class="calendar-header">
        <button class="nav-btn" @click="$emit('prev-month')">← 上月</button>
        <div class="calendar-title">
          {{ history?.year }}年 {{ history?.month }}月
        </div>
        <button class="nav-btn" @click="$emit('next-month')">下月 →</button>
      </div>

      <div class="calendar-grid">
        <div v-for="w in weekdays" :key="w" class="calendar-weekday">{{ w }}</div>
        <div
          v-for="(day, idx) in fullCalendar"
          :key="idx"
          class="calendar-day"
          :class="getDayClass(day)"
          @click="selectDay(day)"
        >
          <span v-if="day" class="day-num">{{ day.day }}</span>
        </div>
      </div>

      <div class="legend">
        <div class="legend-item">
          <div class="legend-box answered"></div>
          <span>已回答</span>
        </div>
        <div class="legend-item">
          <div class="legend-box missed"></div>
          <span>断更（应回答未答）</span>
        </div>
        <div class="legend-item">
          <div class="legend-box today"></div>
          <span>今天</span>
        </div>
        <div class="legend-item">
          <div class="legend-box empty-day"></div>
          <span>无记录</span>
        </div>
      </div>
    </template>

    <div v-if="selectedDay" class="day-detail">
      <h4>📌 {{ formatFullDate(selectedDay.date) }}</h4>
      <div v-if="selectedDay.hasQuestion">
        <p class="q-text">❓ {{ selectedDay.question }}</p>
        <div v-if="selectedDay.answered && selectedDay.answer">
          <p class="a-text">{{ selectedDay.answer }}</p>
        </div>
        <div v-else class="empty-note">这一天没有回答</div>
      </div>
      <div v-else class="empty-note">这一天还没有分配问题</div>
    </div>

    <div v-if="loading" class="empty-note">加载中...</div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  history: Object,
  loading: Boolean
})

const emit = defineEmits(['prev-month', 'next-month', 'jump-to-date'])

const weekdays = ['日', '一', '二', '三', '四', '五', '六']
const selectedDay = ref(null)
const searchKeyword = ref('')
const searchResults = ref([])
const searching = ref(false)
let searchTimer = null

const fullCalendar = computed(() => {
  if (!props.history?.calendar) return []
  const cal = [...props.history.calendar]
  const firstDay = cal[0]
  if (!firstDay) return cal
  const d = new Date(firstDay.date)
  const firstWeekday = d.getDay()
  for (let i = 0; i < firstWeekday; i++) {
    cal.unshift(null)
  }
  return cal
})

function getTodayStr() {
  const t = new Date()
  const y = t.getFullYear()
  const m = String(t.getMonth() + 1).padStart(2, '0')
  const d = String(t.getDate()).padStart(2, '0')
  return `${y}-${m}-${d}`
}

function isPast(dateStr) {
  return dateStr < getTodayStr()
}

function getDayClass(day) {
  if (!day) return 'empty'
  const classes = []
  const todayStr = getTodayStr()
  if (day.date === todayStr) {
    classes.push('today')
  }
  if (day.answered) {
    classes.push('answered')
  } else if (day.hasQuestion && isPast(day.date)) {
    classes.push('missed')
  } else if (day.hasQuestion) {
    classes.push('has-question')
  }
  return classes.join(' ')
}

function selectDay(day) {
  if (day && day.hasQuestion) {
    selectedDay.value = day
  } else {
    selectedDay.value = null
  }
}

function formatFullDate(dateStr) {
  if (!dateStr) return ''
  const d = new Date(dateStr)
  const weekdays = ['周日', '周一', '周二', '周三', '周四', '周五', '周六']
  return `${d.getFullYear()}年${d.getMonth() + 1}月${d.getDate()}日 ${weekdays[d.getDay()]}`
}

function escapeHtml(text) {
  const div = document.createElement('div')
  div.textContent = text
  return div.innerHTML
}

function highlightText(text, keyword) {
  if (!text || !keyword) return escapeHtml(text || '')
  const escaped = escapeHtml(text)
  const kw = escapeHtml(keyword)
  const regex = new RegExp(`(${kw.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')})`, 'gi')
  return escaped.replace(regex, '<mark class="highlight">$1</mark>')
}

async function doSearch() {
  const kw = searchKeyword.value.trim()
  if (!kw) {
    searchResults.value = []
    return
  }
  searching.value = true
  try {
    const res = await fetch(`/api/search?keyword=${encodeURIComponent(kw)}`)
    const json = await res.json()
    if (json.success) {
      searchResults.value = json.data
    }
  } catch (e) {
    console.error('搜索出错', e)
    searchResults.value = []
  } finally {
    searching.value = false
  }
}

function handleSearch() {
  if (searchTimer) clearTimeout(searchTimer)
  searchTimer = setTimeout(() => {
    doSearch()
  }, 300)
}

function clearSearch() {
  searchKeyword.value = ''
  searchResults.value = []
  if (searchTimer) clearTimeout(searchTimer)
}

function jumpToDate(dateStr) {
  const d = new Date(dateStr)
  emit('jump-to-date', {
    year: d.getFullYear(),
    month: d.getMonth() + 1,
    date: dateStr
  })
  clearSearch()
  const day = props.history?.calendar?.find(d => d.date === dateStr)
  if (day) {
    selectedDay.value = day
  }
}

watch(() => props.history, () => {
  if (searchResults.value.length === 0) {
    selectedDay.value = null
  }
})
</script>
