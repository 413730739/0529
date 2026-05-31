<script setup>
import { ref, onMounted, computed } from 'vue'

// --- 狀態定義 ---
const todayStr = ref(new Date().toISOString().split('T')[0])
const events = ref([])
const newEvent = ref({
  Summary: '',
  StartDate: '',
  EndDate: '',
  Description: '',
  Location: '',
  Category: '一般'
})
const categories = ref(['一般', '工作', '個人', '重要'])
const newCategory = ref('')
const isSyncing = ref(false)
const syncError = ref(null)
const gasUrl = ref('https://script.google.com/macros/s/AKfycbziHhMPZLiO38jP2IU8l9rLLe93hKOR2PU8aPxU9LIgKmm6hGXfLAerwTCX0Uqc6jso7A/exec')

// --- 行事曆視圖狀態 ---
const selectedDate = ref('2026-08-01')
const calendarViewDate = ref(new Date('2026-08-01'))
const weekDays = ['日', '一', '二', '三', '四', '五', '六']

// --- 內置 ICS 資料 (calendar-events-2026-05-29.ics) ---
const icsData = `BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//校園行事曆//ZH
CALSCALE:GREGORIAN
METHOD:PUBLISH
X-WR-CALNAME:校園行事曆
X-WR-TIMEZONE:Asia/Taipei
BEGIN:VEVENT
DTSTART;VALUE=DATE:20260801
DTEND;VALUE=DATE:20260802
SUMMARY:【學期課程】本學期開始
CATEGORIES:學期課程
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20260804
DTEND;VALUE=DATE:20260805
SUMMARY:【重要截止】舊生初選第1學期課程
CATEGORIES:重要截止
DESCRIPTION:含研究所新生，至8/10
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20260805
DTEND;VALUE=DATE:20260806
SUMMARY:【行政會議】校教師評審委員會會議
CATEGORIES:行政會議
DESCRIPTION:115學年度
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20260812
DTEND;VALUE=DATE:20260813
SUMMARY:【教學活動】新任系所主管研習會
CATEGORIES:教學活動
DESCRIPTION:115學年度
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20260813
DTEND;VALUE=DATE:20260814
SUMMARY:【重要截止】網路查詢註冊
CATEGORIES:重要截止
DESCRIPTION:至10月2日
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20260825
DTEND;VALUE=DATE:20260826
SUMMARY:【教學活動】新進職員教育訓練
CATEGORIES:教學活動
DESCRIPTION:全面品質管理教育訓練
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20260826
DTEND;VALUE=DATE:20260827
SUMMARY:【招生相關】招生委員會會議
CATEGORIES:招生相關
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20260826
DTEND;VALUE=DATE:20260827
SUMMARY:【重要截止】學士班/研究所新生選課
CATEGORIES:重要截止
DESCRIPTION:至9月3日
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20260905
DTEND;VALUE=DATE:20260906
SUMMARY:【學生活動】新生暨家長座談會
CATEGORIES:學生活動
DESCRIPTION:宿舍住宿生報到進住至9/6
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20260925
DTEND;VALUE=DATE:20260926
SUMMARY:【假日】中秋節
CATEGORIES:假日
DESCRIPTION:放假一天
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20260928
DTEND;VALUE=DATE:20260929
SUMMARY:【假日】教師節
CATEGORIES:假日
DESCRIPTION:放假一天
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20261010
DTEND;VALUE=DATE:20261011
SUMMARY:【假日】國慶日
CATEGORIES:假日
DESCRIPTION:放假一天
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20261108
DTEND;VALUE=DATE:20261109
SUMMARY:【學生活動】創校76週年校慶紀念日
CATEGORIES:學生活動
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20261225
DTEND;VALUE=DATE:20261226
SUMMARY:【假日】行憲紀念日
CATEGORIES:假日
DESCRIPTION:放假一天
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20270101
DTEND;VALUE=DATE:20270102
SUMMARY:【假日】開國紀念日
CATEGORIES:假日
DESCRIPTION:放假一天
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20270131
DTEND;VALUE=DATE:20270201
SUMMARY:【學期課程】學期結束
CATEGORIES:學期課程
DESCRIPTION:第1學期結束
END:VEVENT
BEGIN:VEVENT
DTSTART;VALUE=DATE:20270220
DTEND;VALUE=DATE:20270221
SUMMARY:【學期課程】開始上課
CATEGORIES:學期課程
DESCRIPTION:加退選至3/2
END:VEVENT
END:VCALENDAR`

// --- 格式轉換工具 ---
const toIcsDate = (dateStr) => {
  if (!dateStr) return ''
  // 將 YYYY-MM-DDTHH:mm 轉為 ICS 格式 YYYYMMDDTHHMMSSZ
  return dateStr.replace(/[-:]/g, '') + '00Z'
}

const fromIcsDate = (icsDate) => {
  if (!icsDate) return ''
  const val = icsDate.trim()
  const y = val.substring(0, 4)
  const m = val.substring(4, 6)
  const d = val.substring(6, 8)
  if (val.includes('T')) {
    const tPos = val.indexOf('T')
    const h = val.substring(tPos + 1, tPos + 3)
    const min = val.substring(tPos + 3, tPos + 5)
    return `${y}-${m}-${d}T${h}:${min}`
  }
  return `${y}-${m}-${d}T00:00` // 全天事件
}

// 強健的日期正規化工具：統一轉為 YYYY-MM-DDTHH:mm 格式進行比對
const normalizeDateStr = (d) => {
  if (!d) return ''
  // 確保轉為字串，將日期中的 / 轉為 -，處理空格為 T，並擷取 YYYY-MM-DDTHH:mm
  const s = d.toString().replace(/\//g, '-')
  return s.substring(0, 16).replace(' ', 'T')
}

const formatDate = (date) => {
  const y = date.getFullYear()
  const m = String(date.getMonth() + 1).padStart(2, '0')
  const d = String(date.getDate()).padStart(2, '0')
  return `${y}-${m}-${d}`
}

// --- 類別色彩對應 ---
const getCategoryColor = (cat) => {
  const colors = {
    '重要': '#ef4444',
    '學期課程': '#3b82f6',
    '重要截止': '#f43f5e',
    '行政會議': '#8b5cf6',
    '教學活動': '#10b981',
    '招生相關': '#f59e0b',
    '學生活動': '#06b6d4',
    '假日': '#f43f5e',
    '一般': '#6b7280',
    '工作': '#374151',
    '個人': '#42b883'
  }
  return colors[cat] || '#6b7280'
}

// --- 跨天判斷工具 ---
/**
 * 判斷活動是否在指定日期處於活躍狀態
 * @param {Object} ev 活動對象
 * @param {string} date 目標日期 'YYYY-MM-DD'
 */
const isEventActiveOnDate = (ev, date) => {
  if (!ev || !ev.StartDate) return false
  
  const s = ev.StartDate.split('T')[0]
  const e = ev.EndDate ? ev.EndDate.split('T')[0] : s
  
  // ICS 標準：全天事件的 DTEND 是結束日的隔天 00:00 (Exclusive)
  // 如果結束時間是 T00:00 且跨天，我們判斷時應使用「小於」EndDate
  const isIcsAllDay = ev.EndDate && ev.EndDate.endsWith('T00:00') && s !== e

  if (isIcsAllDay) {
    return date >= s && date < e
  }
  // 一般個人行程或計時行程使用閉區間比對
  return date >= s && date <= e
}

// --- 功能邏輯 ---
const parseIcsString = (text) => {
  const unfolded = text.replace(/\r\n\s/g, '').replace(/\n\s/g, '');
  const lines = unfolded.split(/\r?\n/);
  const importedEvents = [];
  let currentEvent = null;

  for (const line of lines) {
    if (line.startsWith('BEGIN:VEVENT')) {
      currentEvent = {};
      continue;
    }
    if (line.startsWith('END:VEVENT')) {
      if (currentEvent) {
        const category = currentEvent['CATEGORIES'] || '一般';
        if (!categories.value.includes(category)) categories.value.push(category);
        importedEvents.push({
          Summary: currentEvent['SUMMARY'] || '無標題',
          StartDate: fromIcsDate(currentEvent['DTSTART'] || ''),
          EndDate: fromIcsDate(currentEvent['DTEND'] || ''),
          Description: (currentEvent['DESCRIPTION'] || '').replace(/\\n/gi, '\n'),
          Location: (currentEvent['LOCATION'] || '').replace(/\\,/g, ','),
          Category: category
        });
      }
      currentEvent = null;
      continue;
    }
    if (currentEvent) {
      const colonIdx = line.indexOf(':');
      if (colonIdx > -1) {
        const key = line.substring(0, colonIdx).split(';')[0];
        const value = line.substring(colonIdx + 1);
        currentEvent[key] = value;
      }
    }
  }
  events.value = importedEvents;
}

const calendarDays = computed(() => {
  const year = calendarViewDate.value.getFullYear()
  const month = calendarViewDate.value.getMonth()
  const firstDay = new Date(year, month, 1).getDay()
  const daysInMonth = new Date(year, month + 1, 0).getDate()
  
  const days = []
  for (let i = 0; i < firstDay; i++) days.push(null);
  for (let i = 1; i <= daysInMonth; i++) days.push(formatDate(new Date(year, month, i)));
  return days
})

const changeMonth = (delta) => {
  calendarViewDate.value = new Date(calendarViewDate.value.getFullYear(), calendarViewDate.value.getMonth() + delta, 1)
}

const hasEventsOnDate = (date) => {
  if (!date) return false
  return events.value.some(ev => isEventActiveOnDate(ev, date))
}

const dayEvents = computed(() => {
  return events.value
    .filter(ev => isEventActiveOnDate(ev, selectedDate.value))
    .sort((a, b) => a.StartDate.localeCompare(b.StartDate))
})

const addEvent = async () => {
  if (!newEvent.value.Summary || !newEvent.value.StartDate) return alert('請填寫摘要與開始時間')

  isSyncing.value = true
  const eventData = { ...newEvent.value }

  try {
    // 將資料同步到 Google Sheets
    await fetch(gasUrl.value, {
      method: 'POST',
      mode: 'no-cors', // 重要：處理 GAS 跨網域限制
      headers: {
        'Content-Type': 'text/plain'
      },
      body: JSON.stringify(eventData)
    })

    // 成功發送後再更新本地 UI
    events.value.push(eventData)
    const dateOnly = eventData.StartDate.split('T')[0]
    selectedDate.value = dateOnly
    
    // 重置輸入表單為初始狀態
    newEvent.value = {
      Summary: '',
      StartDate: '',
      EndDate: '',
      Description: '',
      Location: '',
      Category: '一般'
    }
    alert('行程已儲存並同步至雲端')
  } catch (error) {
    console.error('儲存失敗:', error)
    alert('無法連接到雲端試算表')
  } finally {
    isSyncing.value = false
  }
}

const removeEvent = async (ev) => {
  if (!confirm('確定要刪除此行程嗎？')) return
  isSyncing.value = true

  try {
    // 發送刪除請求到 Google Sheets
    await fetch(gasUrl.value, {
      method: 'POST',
      mode: 'no-cors',
      headers: {
        'Content-Type': 'text/plain;charset=utf-8'
      },
      body: JSON.stringify({
        action: 'DELETE',
        Summary: String(ev.Summary).trim(),
        StartDate: normalizeDateStr(ev.StartDate)
      })
    })

    // 同步更新本地 UI
    const idx = events.value.findIndex(e => 
      String(e.Summary).trim() === String(ev.Summary).trim() && 
      normalizeDateStr(e.StartDate) === normalizeDateStr(ev.StartDate)
    )
    
    if (idx > -1) events.value.splice(idx, 1)
    alert('行程已從雲端同步刪除')
  } catch (error) {
    console.error('刪除失敗:', error)
    alert('無法同步刪除雲端資料')
  } finally {
    isSyncing.value = false
  }
}

const loadRemoteEvents = async (retryCount = 0) => {
  try {
    isSyncing.value = true
    syncError.value = null // 開始讀取前清除錯誤
    // 增加時間戳記防止瀏覽器快取舊資料
    // 使用 GET 請求抓取資料，增加快取排除
    const response = await fetch(`${gasUrl.value}?action=READ&t=${Date.now()}`)
    if (!response.ok) throw new Error('伺服器回應錯誤')
    const data = await response.json()
    console.log('雲端行程清單:', data)
    
    if (Array.isArray(data)) {
      const remoteEvents = data.filter(remoteEv => {
        const summary = remoteEv.Summary || remoteEv.summary
        const start = remoteEv.StartDate || remoteEv.startDate
        
        // 1. 檢查基本資料完整性 (排除標題列或無效內容)
        if (!summary || !start || start === 'null' || summary === 'Summary') return false
        
        // 2. 檢查是否已存在 (避免重複載入)
        const exists = events.value.some(localEv => 
          String(localEv.Summary).trim() === String(summary).trim() && 
          normalizeDateStr(localEv.StartDate) === normalizeDateStr(start)
        )
        return !exists
      }).map(ev => ({
        // 統一將欄位名稱轉為大寫開頭，確保與 Vue 模板相容
        Summary: ev.Summary || ev.summary || '無標題',
        StartDate: ev.StartDate || ev.startDate,
        EndDate: ev.EndDate || ev.endDate || '',
        Category: ev.Category || ev.category || '一般',
        Location: ev.Location || ev.location || '',
        Description: ev.Description || ev.description || ''
      }))
      
      // 合併雲端行程
      events.value = [...events.value, ...remoteEvents]
      // 讀取完畢後，對所有行程進行排序
      events.value.sort((a, b) => a.StartDate.localeCompare(b.StartDate))
    }
  } catch (error) {
    console.warn(`行程載入失敗 (重試第 ${retryCount} 次):`, error)
    const maxRetries = 2
    if (retryCount < maxRetries) {
      // 自動重試：間隔 2 秒後再次嘗試
      setTimeout(() => loadRemoteEvents(retryCount + 1), 2000)
    } else {
      syncError.value = '無法連線至雲端試算表，請檢查網路。'
    }
  } finally {
    isSyncing.value = false
  }
}

onMounted(async () => {
  parseIcsString(icsData)
  await loadRemoteEvents()
})
</script>

<template>
  <div class="app-wrapper">
    <header>
      <div class="header-container">
        <div class="logo">
          <span class="icon">📅</span>
          <h1>行事曆管理系統</h1>
        </div>
        <div v-if="isSyncing" class="sync-status">
          <span class="spinner"></span> 同步中...
        </div>
      </div>
    </header>

    <main>
      <aside class="sidebar">
        <div class="card">
          <div class="sync-feedback">
            <div v-if="syncError" class="sync-error-banner">
              <span class="error-msg">{{ syncError }}</span>
              <button @click="loadRemoteEvents(0)" class="mini-retry-btn">重試</button>
            </div>
          </div>
          <div class="calendar-nav">
            <button class="nav-btn" @click="changeMonth(-1)">
              <span class="icon">❮</span>
            </button>
            <h2 class="month-title">{{ calendarViewDate.getFullYear() }}年 {{ calendarViewDate.getMonth() + 1 }}月</h2>
            <button class="nav-btn" @click="changeMonth(1)">
              <span class="icon">❯</span>
            </button>
          </div>
          <div class="calendar-grid">
            <div v-for="d in weekDays" :key="d" class="weekday">{{ d }}</div>
            <div 
              v-for="(date, idx) in calendarDays" 
              :key="idx" 
              class="day"
              :class="{ 'empty': !date, 'selected': selectedDate === date, 'is-today': date === todayStr }"
              @click="onCalendarDateClick(date)"
            >
              <span v-if="date">{{ date.split('-')[2] }}</span>
              <div v-if="hasEventsOnDate(date)" class="dot"></div>
            </div>
          </div>
        </div>

        <div class="card form-card">
          <h3>新增行程事項</h3>
          <div class="event-form">
            <div class="form-group">
              <label>活動標題</label>
              <input v-model="newEvent.Summary" placeholder="例如：期中報告" class="main-input" />
            </div>
            <div class="form-group">
              <label>分類</label>
              <select v-model="newEvent.Category" class="cat-select">
                <option v-for="c in categories" :key="c" :value="c">{{ c }}</option>
              </select>
            </div>
            <div class="datetime-group">
              <div class="dt-item">
                <label>開始時間</label>
                <input type="datetime-local" v-model="newEvent.StartDate" class="custom-datetime" />
              </div>
              <div class="dt-item">
                <label>結束時間</label>
                <input type="datetime-local" v-model="newEvent.EndDate" class="custom-datetime" />
              </div>
            </div>
          <div class="form-group">
            <label>地點</label>
            <input v-model="newEvent.Location" placeholder="地點" />
          </div>
          <div class="form-group">
            <label>詳細描述</label>
            <textarea v-model="newEvent.Description" placeholder="描述內容..."></textarea>
          </div>
          <button @click="addEvent" :disabled="isSyncing" class="primary full-width">
            {{ isSyncing ? '同步處理中...' : '儲存行程' }}
          </button>
          </div>
        </div>
    </aside>

    <div class="main-content">
      <div class="card day-card">
          <div class="day-header">
          <h2>{{ selectedDate }} 行程清單</h2>
          <span class="event-count">{{ dayEvents.length }} 個事項</span>
          </div>
          <div v-if="dayEvents.length === 0" class="no-events">
          <div class="empty-state">
            <span class="empty-icon">☕</span>
            <p>今天沒有安排行程，放鬆一下吧！</p>
          </div>
          </div>
          <div v-else class="event-list">
            <div v-for="ev in dayEvents" :key="ev.Summary + ev.StartDate" class="event-item">
              <div class="category-indicator" :style="{ backgroundColor: getCategoryColor(ev.Category) }"></div>
              <div class="event-content">
                <div class="event-main">
                  <strong class="event-summary">{{ ev.Summary }}</strong>
                  <span class="tag" :style="{ color: getCategoryColor(ev.Category), backgroundColor: getCategoryColor(ev.Category) + '15' }">
                    {{ ev.Category }}
                  </span>
                </div>
                <div class="event-meta">
                  <span v-if="ev.Location">📍 {{ ev.Location }}</span>
                  <span>⏰ {{ ev.StartDate.split('T')[1] }} ~ {{ ev.EndDate ? ev.EndDate.split('T')[1] : '結束' }}</span>
                </div>
                <p v-if="ev.Description" class="event-desc">{{ ev.Description }}</p>
              </div>
              <button @click="removeEvent(ev)" class="delete-btn" title="刪除行程">✕</button>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style scoped>
/* 全域基礎 */
.app-wrapper { font-family: system-ui, -apple-system, sans-serif; color: #334155; background: #f8fafc; min-height: 100vh; padding: 0; }

/* 頁首樣式 */
header { background: #ffffff; border-bottom: 1px solid #e2e8f0; padding: 1rem 2rem; position: sticky; top: 0; z-index: 100; box-shadow: 0 1px 3px rgba(0,0,0,0.05); }
.header-container { max-width: 1400px; margin: 0 auto; display: flex; justify-content: space-between; align-items: center; }
.logo { display: flex; align-items: center; gap: 12px; }
.logo h1 { font-size: 1.25rem; font-weight: 700; margin: 0; color: #1e293b; }
.logo .icon { font-size: 1.5rem; }
.sync-status { display: flex; align-items: center; gap: 8px; font-size: 0.875rem; color: #10b981; font-weight: 600; }
.spinner { width: 12px; height: 12px; border: 2px solid #10b981; border-top-color: transparent; border-radius: 50%; animation: spin 0.8s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }

/* 主要佈局 */
main { display: grid; grid-template-columns: 400px 1fr; gap: 2rem; max-width: 1400px; margin: 2rem auto; padding: 0 2rem; }
.card { background: #ffffff; padding: 1.5rem; border-radius: 12px; border: 1px solid #e2e8f0; box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1); margin-bottom: 1.5rem; }
h3 { margin: 0 0 1rem 0; font-size: 1.1rem; color: #1e293b; font-weight: 600; }

/* 側邊欄與日曆 */
.sidebar { min-width: 0; }
.sync-feedback { min-height: 24px; margin-bottom: 8px; }
.month-title { font-size: 1.1rem; margin: 0; color: #1e293b; font-weight: 700; }
.calendar-nav { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1.5rem; }
.nav-btn { background: white; border: 1px solid #e2e8f0; width: 32px; height: 32px; border-radius: 8px; cursor: pointer; transition: all 0.2s; }
.nav-btn:hover { background: #f1f5f9; border-color: #cbd5e0; }
.sync-error-banner { background: #fff1f2; border: 1px solid #fecaca; padding: 0.5rem; border-radius: 8px; display: flex; align-items: center; justify-content: space-between; font-size: 0.8rem; }
.error-msg { color: #be123c; font-weight: 500; }

.calendar-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 4px; }
.weekday { text-align: center; color: #94a3b8; font-size: 0.7rem; font-weight: 700; padding: 8px 0; }
.day { aspect-ratio: 1; display: flex; flex-direction: column; align-items: center; justify-content: center; cursor: pointer; border-radius: 50%; font-size: 0.9rem; position: relative; transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1); border: 2px solid transparent; }
.day:not(.empty):hover { background: #f1f5f9; color: #3b82f6; }
.day.selected { background: #3b82f6 !important; color: white !important; font-weight: 700; transform: scale(1.1); box-shadow: 0 4px 10px rgba(59, 130, 246, 0.3); }
.day.is-today { border-color: #3b82f6; color: #3b82f6; font-weight: 700; }
.day.empty { cursor: default; }
.dot { width: 5px; height: 5px; background: #f59e0b; border-radius: 50%; position: absolute; bottom: 6px; }
.selected .dot { background: white; }

/* 表單樣式 */
.form-card { border-top: 4px solid #3b82f6; }
.form-group { margin-bottom: 1rem; }
.form-group label { display: block; font-size: 0.8rem; color: #64748b; margin-bottom: 4px; font-weight: 600; }
.event-form input, .event-form select, .event-form textarea { width: 100%; padding: 0.8rem 1rem; border: 1px solid #e2e8f0; border-radius: 8px; font-size: 1rem; background: #fcfcfc; transition: border-color 0.2s; box-sizing: border-box; }
.event-form input:focus, .event-form select:focus, .event-form textarea:focus { outline: none; border-color: #3b82f6; background: white; }

/* 日期時間選擇器美化 */
.custom-datetime { cursor: pointer; position: relative; }
.custom-datetime::-webkit-calendar-picker-indicator { 
  background: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="%233b82f6" class="bi bi-calendar-event" viewBox="0 0 16 16"><path d="M11 6.5a.5.5 0 0 1 .5-.5h1a.5.5 0 0 1 .5.5v1a.5.5 0 0 1-.5.5h-1a.5.5 0 0 1-.5-.5z"/><path d="M3.5 0a.5.5 0 0 1 .5.5V1h8V.5a.5.5 0 0 1 1 0V1h1a2 2 0 0 1 2 2v11a2 2 0 0 1-2 2H2a2 2 0 0 1-2-2V3a2 2 0 0 1 2-2h1V.5a.5.5 0 0 1 .5-.5M1 4v10a1 1 0 0 0 1 1h12a1 1 0 0 0 1-1V4z"/></svg>') no-repeat;
  cursor: pointer;
  padding: 5px;
}

.datetime-group { display: grid; grid-template-columns: 1fr; gap: 12px; margin-bottom: 1rem; }
button.primary { background: #3b82f6; color: white; border: none; font-weight: 600; padding: 0.8rem; border-radius: 8px; cursor: pointer; transition: background 0.2s; }
button.primary:hover { background: #2563eb; }
button.primary:disabled { background: #94a3b8; cursor: not-allowed; }

/* 行程清單樣式 */
.main-content { min-width: 0; }
.day-card { border-top: 4px solid #10b981; }
.day-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1.5rem; padding-bottom: 1rem; border-bottom: 1px solid #f1f5f9; }
.day-header h2 { margin: 0; font-size: 1.25rem; color: #1e293b; }
.event-count { font-size: 0.875rem; color: #64748b; background: #f1f5f9; padding: 4px 12px; border-radius: 20px; font-weight: 600; }

.event-item { position: relative; display: flex; background: #ffffff; border-radius: 12px; margin-bottom: 1rem; padding: 1.25rem; border: 1px solid #e2e8f0; transition: transform 0.2s, box-shadow 0.2s; }
.event-item:hover { transform: translateY(-2px); box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1); }
.category-indicator { position: absolute; left: 0; top: 0; bottom: 0; width: 4px; border-top-left-radius: 12px; border-bottom-left-radius: 12px; }
.event-main { display: flex; align-items: center; gap: 10px; margin-bottom: 0.5rem; }
.event-summary { font-size: 1.1rem; color: #1e293b; font-weight: 600; }
.tag { padding: 2px 8px; border-radius: 6px; font-size: 0.65rem; font-weight: 700; text-transform: uppercase; }
.event-meta { font-size: 0.875rem; color: #64748b; display: flex; gap: 1rem; }
.event-desc { margin-top: 1rem; font-size: 0.9rem; color: #475569; line-height: 1.6; padding: 0.75rem; background: #f8fafc; border-radius: 8px; }
.delete-btn { position: absolute; right: 1rem; top: 1.25rem; opacity: 0; color: #94a3b8; border: none; background: transparent; cursor: pointer; transition: opacity 0.2s; font-size: 1.1rem; }
.event-item:hover .delete-btn { opacity: 1; }
.delete-btn:hover { color: #ef4444; }

/* 空狀態與響應式 */
.no-events { display: flex; align-items: center; justify-content: center; min-height: 200px; text-align: center; }
.empty-state { color: #94a3b8; }
.empty-icon { font-size: 3rem; display: block; margin-bottom: 1rem; }

@media (max-width: 1024px) { main { grid-template-columns: 1fr; } }
</style>
