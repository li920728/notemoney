<template>
  <div class="app">
    <header class="header">
      <h1>记账 App</h1>
      <div class="data-actions">
        <button class="btn-export" @click="exportData">导出数据</button>
        <button class="btn-import" @click="triggerImport">导入数据</button>
        <input 
          type="file" 
          ref="fileInput"
          style="display: none"
          accept=".json"
          @change="importData"
        />
      </div>
    </header>

    <div class="total">
      <div class="total-label">总计</div>
      <div class="total-amount">¥{{ totalAmount.toFixed(2) }}</div>
      <div class="subsidy-estimate">预计补助花费 ¥{{ subsidyEstimate.toFixed(2) }} 元 (5000)</div>
      <div class="subsidy-formula">计算公式：唯一天数 × 170 元/天 = {{ uniqueDaysCount }}天 × 170 = ¥{{ subsidyEstimate.toFixed(2) }}</div>
    </div>

    <div class="form">
      <h2 class="form-title">添加记录</h2>
      
      <div class="form-group">
        <label>时间</label>
        <input 
          type="text" 
          v-model="newRecord.timeDisplay"
          placeholder="2024/01/15 09:30"
          required
        />
      </div>

      <div class="form-group">
        <label>付款人</label>
        <select v-model="newRecord.payer">
          <option value="李龙龙">李龙龙</option>
          <option value="李志豪">李志豪</option>
          <option value="魏兴兴">魏兴兴</option>
        </select>
      </div>

      <div class="form-group">
        <label>类别</label>
        <select v-model="newRecord.category">
          <option value="1 住宿">1 住宿</option>
          <option value="2 早餐">2 早餐</option>
          <option value="3 午餐">3 午餐</option>
          <option value="4 晚餐">4 晚餐</option>
          <option value="5 发票">5 发票</option>
          <option value="6 其他">6 其他</option>
        </select>
      </div>

      <div class="form-group">
        <label>金额 (元)</label>
        <input 
          type="number" 
          v-model.number="newRecord.amount"
          step="0.01"
          min="0"
          placeholder="0.00"
          required
        />
      </div>

      <button class="btn-add" @click="addRecord">添加记录</button>
    </div>

    <div class="records" style="margin-top: 20px;">
      <div class="records-header">
        <h2 class="records-title">按付款人统计</h2>
      </div>
      <div class="payer-stats-vertical">
        <div 
          v-for="(stat, payer) in payerStats" 
          :key="payer"
          class="payer-stat-row"
        >
          <span class="payer-name-row">{{ payer }}:</span>
          <div class="payer-amount-row">
            <span class="payer-amount-text">¥{{ stat.toFixed(2) }}</span>
            <span class="payer-third-text">¥{{ (stat / 3).toFixed(2) }}</span>
          </div>
        </div>
      </div>
    </div>

    <div class="accommodation-stats" style="margin-top: 20px; background: #e3f2fd; padding: 12px 15px; border-radius: 6px;">
      <div style="font-size: 14px; font-weight: 500; color: #1565c0; margin-bottom: 10px;">住宿统计：</div>
      <div class="payer-stats-vertical">
        <div 
          v-for="(stat, payer) in accommodationStats" 
          :key="payer"
          class="payer-stat-row"
        >
          <span class="payer-name-row">{{ payer }}:</span>
          <div class="payer-amount-row">
            <span class="payer-amount-text">¥{{ stat.toFixed(2) }}</span>
            <span class="payer-third-text">¥{{ (stat / 3).toFixed(2) }}</span>
          </div>
        </div>
      </div>
    </div>

    <div class="recent-filter" style="margin-top: 20px;">
      <div class="recent-filter-header">
        <h3 class="filter-title">{{ recentMode === 'settlement' ? '结算后消费统计' : `最近 ${recentDays} 天消费` }}</h3>
        <div class="recent-actions">
          <button 
            v-if="recentMode === 'settlement'"
            class="btn-toggle-mode"
            @click="switchToDays"
          >
            切换到按天数统计
          </button>
          <button 
            v-if="recentMode === 'days'"
            class="btn-toggle-mode"
            @click="switchToSettlement"
          >
            切换到结算后统计
          </button>
          <button 
            v-if="recentMode === 'days'"
            class="btn-toggle-days"
            @click="toggleRecentDays"
          >
            切换到最近 {{ recentDays === 2 ? 3 : 2 }} 天
          </button>
        </div>
      </div>
      <div class="payer-stats-vertical">
        <div 
          v-for="(stat, payer) in recentStats" 
          :key="payer"
          class="payer-stat-row"
        >
          <span class="payer-name-row">{{ payer }}:</span>
          <div class="payer-amount-row">
            <span class="payer-amount-text">¥{{ stat.toFixed(2) }}</span>
            <span class="payer-third-text">¥{{ (stat / 3).toFixed(2) }}</span>
          </div>
        </div>
      </div>
      <div class="recent-period-info" v-if="recentMode === 'settlement' && lastSettlementTime">
        统计时间：{{ lastSettlementTime }} 至今
      </div>
    </div>

    <div class="last-settlement" style="margin-top: 20px;">
      <div class="settlement-header">
        <h3 class="filter-title">上次结算时间</h3>
        <button class="btn-reset-settlement" @click="resetSettlement">重置结算时间</button>
      </div>
      <div 
        class="settlement-time-wrapper"
        @click="handleSettlementClick"
        style="cursor: pointer;"
        title="点击 3 次记录当前时间"
      >
        <div class="settlement-time">{{ lastSettlementTime || '暂无结算记录' }}</div>
        <div class="settlement-hint" v-if="showSettlementHint">
          再点击 {{ remainingClicks }} 次完成记录
        </div>
      </div>
    </div>

    <div class="date-filter" style="margin-top: 20px;">
      <h3 class="filter-title">选择日期范围</h3>
      <div class="filter-group">
        <div class="filter-input">
          <label>开始日期</label>
          <input 
            type="text" 
            v-model="filterStartDate"
            placeholder="YYYY-MM-DD"
            @focus="showStartDatePicker = true"
            @blur="setTimeout(() => showStartDatePicker = false, 200)"
            readonly
            style="cursor: pointer;"
          />
          <div v-if="showStartDatePicker" class="date-picker-popup" @click.stop>
            <div class="date-picker-header">
              <button @click="prevMonth" class="date-nav-btn">&lt;</button>
              <span>{{ pickerYear }}年{{ pickerMonth }}月</span>
              <button @click="nextMonth" class="date-nav-btn">&gt;</button>
            </div>
            <div class="date-picker-grid">
              <div v-for="day in getDaysInMonth()" :key="day" 
                   @click="selectStartDate(day)"
                   :class="['date-picker-day', { 
                     'selected': filterStartDate === `${pickerYear}-${String(pickerMonth).padStart(2,'0')}-${String(day).padStart(2,'0')}`,
                     'today': isToday(pickerYear, pickerMonth, day)
                   }]">
                {{ day }}
              </div>
            </div>
          </div>
        </div>
        <div class="filter-input">
          <label>结束日期</label>
          <input 
            type="text" 
            v-model="filterEndDate"
            placeholder="YYYY-MM-DD"
            @focus="showEndDatePicker = true"
            @blur="setTimeout(() => showEndDatePicker = false, 200)"
            readonly
            style="cursor: pointer;"
          />
          <div v-if="showEndDatePicker" class="date-picker-popup" @click.stop>
            <div class="date-picker-header">
              <button @click="prevMonth" class="date-nav-btn">&lt;</button>
              <span>{{ pickerYear }}年{{ pickerMonth }}月</span>
              <button @click="nextMonth" class="date-nav-btn">&gt;</button>
            </div>
            <div class="date-picker-grid">
              <div v-for="day in getDaysInMonth()" :key="day" 
                   @click="selectEndDate(day)"
                   :class="['date-picker-day', { 
                     'selected': filterEndDate === `${pickerYear}-${String(pickerMonth).padStart(2,'0')}-${String(day).padStart(2,'0')}`,
                     'today': isToday(pickerYear, pickerMonth, day)
                   }]">
                {{ day }}
              </div>
            </div>
          </div>
        </div>
        <button class="btn-reset" @click="resetDateFilter">重置</button>
      </div>
      <div v-if="isFiltered" class="filtered-stats">
        <div class="filtered-stats-title">选中日期统计</div>
        <div 
          v-for="(stat, payer) in filteredPayerStats" 
          :key="payer"
          class="filtered-stat-item"
        >
          <span class="filtered-payer">{{ payer }}:</span>
          <div class="filtered-amount-wrapper">
            <span class="filtered-amount">¥{{ stat.toFixed(2) }}</span>
            <span class="filtered-third">¥{{ (stat / 3).toFixed(2) }}</span>
          </div>
        </div>
      </div>
    </div>

    <div class="records" style="margin-top: 20px;">
      <div class="records-header">
        <h2 class="records-title">记录列表</h2>
      </div>

      <div v-if="records.length === 0" class="empty-state">
        <div class="empty-state-icon">📝</div>
        <div class="empty-state-text">还没有记录，点击上方添加</div>
      </div>

      <div v-else>
          <div 
            v-for="(record, index) in paginatedRecords" 
            :key="record.id"
            class="record-item"
          >
            <div class="record-info">
              <div class="record-payer">{{ record.payer }}</div>
              <select 
                v-if="editingRecord && editingRecord.id === record.id"
                v-model="editingRecord.category"
                class="edit-category-select"
              >
                <option value="1 住宿">1 住宿</option>
                <option value="2 早餐">2 早餐</option>
                <option value="3 午餐">3 午餐</option>
                <option value="4 晚餐">4 晚餐</option>
                <option value="5 发票">5 发票</option>
                <option value="6 其他">6 其他</option>
              </select>
              <div 
                v-else
                class="record-category"
                @dblclick="startEdit(record)"
                style="cursor: pointer;"
                title="双击编辑类别"
              >
                {{ record.category }}
              </div>
              <div class="record-time">
                <input 
                  v-if="editingRecord && editingRecord.id === record.id"
                  type="text"
                  v-model="editingRecord.timeDisplay"
                  class="edit-time-input"
                />
                <span 
                  v-else
                  @dblclick="startEdit(record)"
                  class="record-time-text"
                  style="cursor: pointer;"
                  title="双击编辑时间"
                >
                  {{ formatTime(record.time) }}
                </span>
                <span 
                  v-if="record.note" 
                  class="record-note-display"
                  @click="editNote(record)"
                >
                  {{ record.note }}
                </span>
                <span 
                  v-else 
                  class="record-note-add"
                  @click="editNote(record)"
                >
                  + 备注
                </span>
              </div>
            </div>
            <div class="record-amount">
              <input 
                v-if="editingRecord && editingRecord.id === record.id"
                type="number"
                v-model.number="editingRecord.amount"
                step="0.01"
                min="0"
                class="edit-amount-input"
              />
              <div 
                v-else
                class="record-price"
                @dblclick="startEdit(record)"
                style="cursor: pointer;"
                title="双击编辑金额"
              >
                ¥{{ record.amount.toFixed(2) }}
              </div>
            </div>
            <div class="record-actions">
              <button 
                v-if="editingRecord && editingRecord.id === record.id"
                class="btn-save-edit"
                @click="saveEdit(record)"
              >
                保存
              </button>
              <button 
                v-if="editingRecord && editingRecord.id === record.id"
                class="btn-cancel-edit"
                @click="cancelEdit"
              >
                取消
              </button>
              <button 
                v-if="!editingRecord"
                class="btn-edit"
                @click="startEdit(record)"
              >
                编辑
              </button>
              <button 
                class="btn-delete"
                :class="{ 'btn-delete-warn': deleteClickCount > 0 && recordToDelete === record.id }"
                @click="deleteRecordFromList(record.id)"
              >
                {{ deleteClickCount === 0 ? '删除' : (deleteClickCount === 1 ? '再次点击删除' : '第三次点击删除') }}
              </button>
            </div>
          </div>

        <div v-if="totalPages > 1" class="pagination">
          <button 
            class="pagination-btn" 
            @click="goToPrevious" 
            :disabled="currentPage === 1"
          >
            上一页
          </button>
          <span class="pagination-info">
            第 {{ currentPage }} 页，共 {{ totalPages }} 页
          </span>
          <button 
            class="pagination-btn" 
            @click="goToNext" 
            :disabled="currentPage === totalPages"
          >
            下一页
          </button>
        </div>
      </div>
    </div>

    <div v-if="showNoteInput" class="note-modal-overlay" @click="cancelNote">
      <div class="note-modal" @click.stop>
        <h3 class="note-modal-title">编辑备注</h3>
        <textarea 
          v-model="noteInput" 
          class="note-input"
          placeholder="请输入备注内容"
          rows="3"
          autofocus
        ></textarea>
        <div class="note-modal-actions">
          <button class="btn-cancel" @click="cancelNote">取消</button>
          <button class="btn-save" @click="saveNote">保存</button>
        </div>
      </div>
    </div>

    <div v-if="showMissingRecordModal" class="note-modal-overlay" @click="closeMissingRecordModal">
      <div class="note-modal" @click.stop style="max-width: 500px;">
        <h3 class="note-modal-title">记录缺失提醒</h3>
        <div class="missing-record-content">
          <div v-if="missingLunchDates.length > 0" class="missing-item">
            <div class="missing-text">你没有吃午饭吗？缺失日期：{{ missingLunchDates.join(', ') }}</div>
            <button class="btn-confirm-missing" @click="confirmMissing('lunch')">我确实没有吃饭</button>
          </div>
          <div v-if="missingDinnerDates.length > 0" class="missing-item">
            <div class="missing-text">你没有吃晚饭吗？缺失日期：{{ missingDinnerDates.join(', ') }}</div>
            <button class="btn-confirm-missing" @click="confirmMissing('dinner')">我确实没有吃饭</button>
          </div>
          <div v-if="missingAccommodationDates.length > 0" class="missing-item">
            <div class="missing-text">你露宿街头了？缺失日期：{{ missingAccommodationDates.join(', ') }}</div>
            <button class="btn-confirm-missing" @click="confirmMissing('accommodation')">我确实没有住宿</button>
          </div>
        </div>
        <div class="note-modal-actions">
          <button class="btn-cancel" @click="closeMissingRecordModal">关闭</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch } from 'vue'

export default {
  name: 'App',
  setup() {
    const newRecord = ref({
      timeDisplay: new Date().toLocaleString('zh-CN', {
        year: 'numeric',
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit'
      }).replace(/,/g, ''),
      time: new Date().toISOString(),
      payer: '李龙龙',
      category: '1 住宿',
      amount: null
    })

    const records = ref([])

    const filterStartDate = ref('')
    const filterEndDate = ref('')
    const isFiltered = ref(false)

    const currentPage = ref(1)
    const pageSize = 10

    const editingRecord = ref(null)
    const showNoteInput = ref(false)
    const noteInput = ref('')
    const showMissingRecordModal = ref(false)
    const missingLunchDates = ref([])
    const missingDinnerDates = ref([])
    const missingAccommodationDates = ref([])
    const recentDays = ref(2)
    const recentMode = ref('settlement')
    const lastSettlementTime = ref('')
    const settlementClickCount = ref(0)
    const settlementTimeout = ref(null)
    const deleteClickCount = ref(0)
    const deleteTimeout = ref(null)
    const recordToDelete = ref(null)
    const editRecordTemp = ref(null)
    
    // 日期选择器相关
    const showStartDatePicker = ref(false)
    const showEndDatePicker = ref(false)
    const pickerYear = ref(new Date().getFullYear())
    const pickerMonth = ref(new Date().getMonth() + 1)

    const loadRecords = () => {
      const saved = localStorage.getItem('expenseRecords')
      if (saved) {
        records.value = JSON.parse(saved)
      }
      const settlement = localStorage.getItem('lastSettlementTime')
      if (settlement) {
        lastSettlementTime.value = settlement
      }
      const savedMode = localStorage.getItem('recentStatsMode')
      if (savedMode) {
        recentMode.value = savedMode
      }
      
      // 检查补助预估是否超过 5000，超过则弹窗提示
      if (records.value.length > 0) {
        const uniqueDays = new Set(records.value.map(record => {
          const date = new Date(record.time)
          return `${date.getFullYear()}/${date.getMonth() + 1}/${date.getDate()}`
        }))
        const subsidyAmount = uniqueDays.size * 170
        
        if (subsidyAmount > 5000) {
          setTimeout(() => {
            alert(`⚠️ 预计补助花费已达 ¥${subsidyAmount.toFixed(2)} 元，超过 5000 元！`)
          }, 500)
        }
        
        // 检查从 5 月 25 号开始的记录是否完整
        checkMissingRecords()
      }
    }
    
    const checkMissingRecords = () => {
      const startDate = new Date(2026, 4, 22) // 5 月 22 号（月份从 0 开始）
      const today = new Date()
      
      // 获取所有日期的记录
      const recordsByDate = {}
      records.value.forEach(record => {
        const recordDate = new Date(record.time)
        if (recordDate >= startDate) {
          const dateKey = `${recordDate.getFullYear()}/${recordDate.getMonth() + 1}/${recordDate.getDate()}`
          if (!recordsByDate[dateKey]) {
            recordsByDate[dateKey] = { lunch: false, dinner: false, accommodation: false }
          }
          
          // 根据类别判断午餐和晚餐
          if (record.category === '3 午餐') {
            recordsByDate[dateKey].lunch = true
          }
          if (record.category === '4 晚餐') {
            recordsByDate[dateKey].dinner = true
          }
          
          // 根据备注判断住宿：格式为"月。日"（如 5.29, 5.30, 6.1 等）
          if (record.category === '1 住宿' && record.note) {
            const note = record.note
            const recordYear = recordDate.getFullYear()
            
            // 匹配"月。日"格式（如 5.29, 6.1 等），支持一条记录多天
            const monthDayMatches = note.match(/(\d+\.\d+)/g)
            if (monthDayMatches) {
              monthDayMatches.forEach(match => {
                const parts = match.split('.')
                const month = parseInt(parts[0])
                const day = parseInt(parts[1])
                const accommodationDateKey = `${recordYear}/${month}/${day}`
                if (!recordsByDate[accommodationDateKey]) {
                  recordsByDate[accommodationDateKey] = { lunch: false, dinner: false, accommodation: false }
                }
                recordsByDate[accommodationDateKey].accommodation = true
              })
            }
          }
        }
      })
      
      // 遍历从 5 月 25 号到今天的每一天
      const missingLunch = []
      const missingDinner = []
      const missingAccommodation = []
      
      const currentDate = new Date(startDate)
      while (currentDate <= today) {
        const dateKey = `${currentDate.getFullYear()}/${currentDate.getMonth() + 1}/${currentDate.getDate()}`
        
        // 过滤已结算的日期（如果有结算时间记录）
        const shouldCheck = true
        
        if (shouldCheck) {
          if (!recordsByDate[dateKey] || !recordsByDate[dateKey].lunch) {
            missingLunch.push(dateKey)
          }
          if (!recordsByDate[dateKey] || !recordsByDate[dateKey].dinner) {
            missingDinner.push(dateKey)
          }
          if (!recordsByDate[dateKey] || !recordsByDate[dateKey].accommodation) {
            missingAccommodation.push(dateKey)
          }
        }
        
        currentDate.setDate(currentDate.getDate() + 1)
      }
      
      // 保存缺失记录信息供弹窗使用
      window.missingRecordsData = { missingLunch, missingDinner, missingAccommodation }
      missingLunchDates.value = missingLunch
      missingDinnerDates.value = missingDinner
      missingAccommodationDates.value = missingAccommodation
      
      if (missingLunch.length > 0 || missingDinner.length > 0 || missingAccommodation.length > 0) {
        setTimeout(() => {
          showMissingRecordModal.value = true
        }, 1000)
      }
    }

    const closeMissingRecordModal = () => {
      showMissingRecordModal.value = false
      missingLunchDates.value = []
      missingDinnerDates.value = []
      missingAccommodationDates.value = []
    }

    const confirmMissing = (type) => {
      const dates = type === 'lunch' ? missingLunchDates.value :
                    type === 'dinner' ? missingDinnerDates.value :
                    missingAccommodationDates.value
      
      dates.forEach(dateStr => {
        const parts = dateStr.split('/')
        const timeStr = `${dateStr} 12:00`
        const time = new Date(
          parseInt(parts[0]),
          parseInt(parts[1]) - 1,
          parseInt(parts[2]),
          12, 0, 0
        ).toISOString()
        
        records.value.push({
          id: Date.now() + Math.random(),
          time: time,
          payer: '未知',
          category: type === 'lunch' ? '3 午餐' :
                    type === 'dinner' ? '4 晚餐' : '1 住宿',
          amount: 0.00,
          note: type === 'accommodation' ? dateStr : ''
        })
      })
      
      // 从列表中移除已确认的日期
      if (type === 'lunch') {
        missingLunchDates.value = []
      } else if (type === 'dinner') {
        missingDinnerDates.value = []
      } else {
        missingAccommodationDates.value = []
      }
      
      alert(`已为 ${dates.length} 天添加 0.00 元记录`)
      
      if (missingLunchDates.value.length === 0 && 
          missingDinnerDates.value.length === 0 && 
          missingAccommodationDates.value.length === 0) {
        showMissingRecordModal.value = false
      }
    }

    const saveRecords = () => {
      localStorage.setItem('expenseRecords', JSON.stringify(records.value))
    }

    watch(records, saveRecords, { deep: true })

    loadRecords()

    const addRecord = () => {
      if (!newRecord.value.amount || newRecord.value.amount <= 0) {
        alert('请输入有效金额')
        return
      }

      const timeStr = newRecord.value.timeDisplay.trim()
      const parsedTime = parseTimeInput(timeStr)
      if (!parsedTime) {
        alert('时间格式不正确，请使用 YYYY/MM/DD HH:mm 格式')
        return
      }

      records.value.push({
        id: Date.now(),
        time: parsedTime,
        payer: newRecord.value.payer,
        category: newRecord.value.category,
        amount: newRecord.value.amount,
        note: ''
      })

      newRecord.value = {
        timeDisplay: new Date().toLocaleString('zh-CN', {
          year: 'numeric',
          month: '2-digit',
          day: '2-digit',
          hour: '2-digit',
          minute: '2-digit'
        }).replace(/,/g, ''),
        time: new Date().toISOString(),
        payer: '李龙龙',
        category: '1 住宿',
        amount: null
      }
    }

    const parseTimeInput = (timeStr) => {
      const formats = [
        /(\d{4})[-/](\d{1,2})[-/](\d{1,2})\s+(\d{1,2}):(\d{1,2})/,
        /(\d{4})[-/](\d{1,2})[-/](\d{1,2})T(\d{1,2}):(\d{1,2})/
      ]

      for (const regex of formats) {
        const match = timeStr.match(regex)
        if (match) {
          const [, year, month, day, hour, minute] = match
          const date = new Date(+year, +month - 1, +day, +hour, +minute)
          if (!isNaN(date.getTime())) {
            return date.toISOString()
          }
        }
      }
      return null
    }

    const deleteRecord = (index) => {
      records.value.splice(index, 1)
    }

    const deleteRecordFromList = (id) => {
      if (deleteClickCount.value === 0) {
        deleteClickCount.value = 1
        recordToDelete.value = id
        deleteTimeout.value = setTimeout(() => {
          deleteClickCount.value = 0
          recordToDelete.value = null
          deleteTimeout.value = null
        }, 2000)
      } else if (deleteClickCount.value === 1 && recordToDelete.value === id) {
        clearTimeout(deleteTimeout.value)
        deleteClickCount.value = 2
        deleteTimeout.value = setTimeout(() => {
          deleteClickCount.value = 0
          recordToDelete.value = null
          deleteTimeout.value = null
        }, 3000)
      } else if (deleteClickCount.value === 2 && recordToDelete.value === id) {
        clearTimeout(deleteTimeout.value)
        if (confirm('确定要删除这条记录吗？')) {
          const index = records.value.findIndex(r => r.id === id)
          if (index !== -1) {
            records.value.splice(index, 1)
          }
          alert('已删除')
        }
        deleteClickCount.value = 0
        recordToDelete.value = null
        deleteTimeout.value = null
      }
    }

    const startEdit = (record) => {
      editRecordTemp.value = {
        id: record.id,
        timeDisplay: formatTime(record.time),
        amount: record.amount,
        category: record.category
      }
      editingRecord.value = editRecordTemp.value
    }

    const saveEdit = (record) => {
      if (!editRecordTemp.value) return
      
      const timeStr = editRecordTemp.value.timeDisplay.trim()
      const parsedTime = parseTimeInput(timeStr)
      
      if (!parsedTime) {
        alert('时间格式不正确，请使用 YYYY/MM/DD HH:mm 格式')
        return
      }
      
      if (!editRecordTemp.value.amount || editRecordTemp.value.amount <= 0) {
        alert('请输入有效金额')
        return
      }
      
      record.time = parsedTime
      record.amount = editRecordTemp.value.amount
      record.category = editRecordTemp.value.category
      
      editRecordTemp.value = null
      editingRecord.value = null
    }

    const cancelEdit = () => {
      editRecordTemp.value = null
      editingRecord.value = null
    }

    const recentStats = computed(() => {
      const stats = {}
      
      if (recentMode.value === 'settlement' && lastSettlementTime.value) {
        const parts = lastSettlementTime.value.split(/[/\s:]/)
        const startTime = new Date(
          parseInt(parts[0]),
          parseInt(parts[1]) - 1,
          parseInt(parts[2]),
          parseInt(parts[3]) || 0,
          parseInt(parts[4]) || 0
        )
        
        records.value.forEach(record => {
          const recordDate = new Date(record.time)
          if (recordDate.getTime() >= startTime.getTime()) {
            const payer = record.payer || '未知'
            if (!stats[payer]) {
              stats[payer] = 0
            }
            stats[payer] += record.amount
          }
        })
      } else if (recentMode.value === 'days') {
        const now = new Date()
        const today = new Date(now.getFullYear(), now.getMonth(), now.getDate())
        const startTime = new Date(today)
        startTime.setDate(startTime.getDate() - recentDays.value + 1)
        startTime.setHours(0, 0, 0, 0)
        
        records.value.forEach(record => {
          const recordDate = new Date(record.time)
          if (recordDate.getTime() >= startTime.getTime()) {
            const payer = record.payer || '未知'
            if (!stats[payer]) {
              stats[payer] = 0
            }
            stats[payer] += record.amount
          }
        })
      }
      
      return stats
    })

    const switchToSettlement = () => {
      recentMode.value = 'settlement'
      localStorage.setItem('recentStatsMode', 'settlement')
    }

    const switchToDays = () => {
      recentMode.value = 'days'
      localStorage.setItem('recentStatsMode', 'days')
    }

    const toggleRecentDays = () => {
      recentDays.value = recentDays.value === 2 ? 3 : 2
    }

    const showSettlementHint = computed(() => {
      return settlementClickCount.value > 0 && settlementClickCount.value < 3
    })

    const remainingClicks = computed(() => {
      return 3 - settlementClickCount.value
    })

    const handleSettlementClick = () => {
      if (settlementClickCount.value === 0) {
        settlementClickCount.value = 1
        settlementTimeout.value = setTimeout(() => {
          settlementClickCount.value = 0
          settlementTimeout.value = null
        }, 3000)
      } else if (settlementClickCount.value === 1) {
        settlementClickCount.value = 2
        clearTimeout(settlementTimeout.value)
        settlementTimeout.value = setTimeout(() => {
          settlementClickCount.value = 0
          settlementTimeout.value = null
        }, 3000)
      } else if (settlementClickCount.value === 2) {
        clearTimeout(settlementTimeout.value)
        const now = new Date()
        const timeStr = `${now.getFullYear()}/${(now.getMonth() + 1).toString().padStart(2, '0')}/${now.getDate().toString().padStart(2, '0')} ${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}`
        lastSettlementTime.value = timeStr
        localStorage.setItem('lastSettlementTime', timeStr)
        settlementClickCount.value = 0
        settlementTimeout.value = null
        alert('结算时间已记录')
      }
    }

    const resetSettlement = () => {
      if (confirm('确定要重置结算时间吗？此操作不可恢复。')) {
        if (confirm('请再次确认：是否确定要清除结算时间记录？')) {
          lastSettlementTime.value = ''
          localStorage.removeItem('lastSettlementTime')
          alert('结算时间已重置')
        }
      }
    }

    const accommodationStats = computed(() => {
      const stats = {}
      let filtered = records.value
      
      if (filterStartDate.value && filterEndDate.value) {
        const startParts = filterStartDate.value.split('-')
        const start = new Date(
          parseInt(startParts[0]),
          parseInt(startParts[1]) - 1,
          parseInt(startParts[2]),
          0, 0, 0, 0
        )
        const endParts = filterEndDate.value.split('-')
        const end = new Date(
          parseInt(endParts[0]),
          parseInt(endParts[1]) - 1,
          parseInt(endParts[2]),
          23, 59, 59, 999
        )
        filtered = records.value.filter(record => {
          const recordDate = new Date(record.time)
          return recordDate.getTime() >= start.getTime() && recordDate.getTime() <= end.getTime()
        })
      }
      
      filtered.forEach(record => {
        if (record.category === '1 住宿') {
          const payer = record.payer || '未知'
          if (!stats[payer]) {
            stats[payer] = 0
          }
          stats[payer] += record.amount
        }
      })
      return stats
    })

    const totalAmount = computed(() => {
      return records.value.reduce((sum, record) => {
        return sum + record.amount
      }, 0)
    })

    const subsidyEstimate = computed(() => {
      const uniqueDays = new Set(records.value.map(record => {
        const date = new Date(record.time)
        return `${date.getFullYear()}/${date.getMonth() + 1}/${date.getDate()}`
      }))
      return uniqueDays.size * 170
    })

    const uniqueDaysCount = computed(() => {
      return new Set(records.value.map(record => {
        const date = new Date(record.time)
        return `${date.getFullYear()}/${date.getMonth() + 1}/${date.getDate()}`
      })).size
    })

    const sortedRecords = computed(() => {
      return [...records.value].sort((a, b) => {
        return new Date(b.time) - new Date(a.time)
      })
    })

    const paginatedRecords = computed(() => {
      let filtered = records.value
      if (isFiltered.value && filterStartDate.value && filterEndDate.value) {
        const startParts = filterStartDate.value.split('-')
        const start = new Date(
          parseInt(startParts[0]),
          parseInt(startParts[1]) - 1,
          parseInt(startParts[2]),
          0, 0, 0, 0
        )
        const endParts = filterEndDate.value.split('-')
        const end = new Date(
          parseInt(endParts[0]),
          parseInt(endParts[1]) - 1,
          parseInt(endParts[2]),
          23, 59, 59, 999
        )
        filtered = records.value.filter(record => {
          const recordDate = new Date(record.time)
          return recordDate.getTime() >= start.getTime() && recordDate.getTime() <= end.getTime()
        })
      }
      const sorted = [...filtered].sort((a, b) => {
        return new Date(b.time) - new Date(a.time)
      })
      const start = (currentPage.value - 1) * pageSize
      const end = start + pageSize
      return sorted.slice(start, end)
    })

    const totalPages = computed(() => {
      let filtered = records.value
      if (isFiltered.value && filterStartDate.value && filterEndDate.value) {
        const startParts = filterStartDate.value.split('-')
        const start = new Date(
          parseInt(startParts[0]),
          parseInt(startParts[1]) - 1,
          parseInt(startParts[2]),
          0, 0, 0, 0
        )
        const endParts = filterEndDate.value.split('-')
        const end = new Date(
          parseInt(endParts[0]),
          parseInt(endParts[1]) - 1,
          parseInt(endParts[2]),
          23, 59, 59, 999
        )
        filtered = records.value.filter(record => {
          const recordDate = new Date(record.time)
          return recordDate.getTime() >= start.getTime() && recordDate.getTime() <= end.getTime()
        })
      }
      return Math.ceil(filtered.length / pageSize)
    })

    const goToPage = (page) => {
      if (page >= 1 && page <= totalPages.value) {
        currentPage.value = page
      }
    }

    const goToPrevious = () => {
      if (currentPage.value > 1) {
        currentPage.value--
      }
    }

    const goToNext = () => {
      if (currentPage.value < totalPages.value) {
        currentPage.value++
      }
    }

    const payerStats = computed(() => {
      const stats = {}
      records.value.forEach(record => {
        const payer = record.payer || '未知'
        if (!stats[payer]) {
          stats[payer] = 0
        }
        stats[payer] += record.amount
      })
      return stats
    })

    const filteredPayerStats = computed(() => {
      let filtered = records.value
      if (isFiltered.value && filterStartDate.value && filterEndDate.value) {
        const startParts = filterStartDate.value.split('-')
        const start = new Date(
          parseInt(startParts[0]),
          parseInt(startParts[1]) - 1,
          parseInt(startParts[2]),
          0, 0, 0, 0
        )
        const endParts = filterEndDate.value.split('-')
        const end = new Date(
          parseInt(endParts[0]),
          parseInt(endParts[1]) - 1,
          parseInt(endParts[2]),
          23, 59, 59, 999
        )
        filtered = records.value.filter(record => {
          const recordDate = new Date(record.time)
          return recordDate.getTime() >= start.getTime() && recordDate.getTime() <= end.getTime()
        })
      }
      const stats = {}
      filtered.forEach(record => {
        const payer = record.payer || '未知'
        if (!stats[payer]) {
          stats[payer] = 0
        }
        stats[payer] += record.amount
      })
      return stats
    })

    const formatTime = (timeStr) => {
      const date = new Date(timeStr)
      const year = date.getFullYear()
      const month = (date.getMonth() + 1).toString().padStart(2, '0')
      const day = date.getDate().toString().padStart(2, '0')
      const hour = date.getHours().toString().padStart(2, '0')
      const minute = date.getMinutes().toString().padStart(2, '0')
      return `${year}/${month}/${day} ${hour}:${minute}`
    }

    const applyDateFilter = () => {
      if (filterStartDate.value && filterEndDate.value) {
        isFiltered.value = true
      }
    }

    const resetDateFilter = () => {
      filterStartDate.value = ''
      filterEndDate.value = ''
      isFiltered.value = false
    }

    const editNote = (record) => {
      editingRecord.value = record
      noteInput.value = record.note || ''
      showNoteInput.value = true
    }

    const saveNote = () => {
      if (editingRecord.value) {
        editingRecord.value.note = noteInput.value.trim()
      }
      showNoteInput.value = false
      editingRecord.value = null
      noteInput.value = ''
    }

    const cancelNote = () => {
      showNoteInput.value = false
      editingRecord.value = null
      noteInput.value = ''
    }

    const fileInput = ref(null)

    const exportData = () => {
      const data = localStorage.getItem('expenseRecords')
      if (!data || data === '[]') {
        alert('没有可导出的数据')
        return
      }
      const blob = new Blob([data], { type: 'application/json' })
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url
      const now = new Date()
      const timeStr = `${now.getFullYear()}${(now.getMonth() + 1).toString().padStart(2, '0')}${now.getDate().toString().padStart(2, '0')}-${now.getHours().toString().padStart(2, '0')}${now.getMinutes().toString().padStart(2, '0')}`
      a.download = `记账-${timeStr}.json`
      a.click()
      URL.revokeObjectURL(url)
    }

    const triggerImport = () => {
      fileInput.value.click()
    }

    const importData = (event) => {
      const file = event.target.files[0]
      if (!file) return
      
      const reader = new FileReader()
      reader.onload = (e) => {
        try {
          const imported = JSON.parse(e.target.result)
          let recordsToImport = []
          
          if (Array.isArray(imported)) {
            recordsToImport = imported
          } else if (imported && Array.isArray(imported.records)) {
            recordsToImport = imported.records
          } else {
            alert('文件格式不正确')
            return
          }
          
          if (confirm('导入将覆盖当前数据，确定继续吗？')) {
            records.value = recordsToImport
            alert('导入成功！')
          }
        } catch (err) {
          alert('文件解析失败：' + err.message)
        }
      }
      reader.readAsText(file)
      event.target.value = ''
    }

    // 日期选择器函数
    const selectStartDate = (day) => {
      filterStartDate.value = `${pickerYear.value}-${String(pickerMonth.value).padStart(2, '0')}-${String(day).padStart(2, '0')}`
      showStartDatePicker.value = false
      applyDateFilter()
    }

    const selectEndDate = (day) => {
      filterEndDate.value = `${pickerYear.value}-${String(pickerMonth.value).padStart(2, '0')}-${String(day).padStart(2, '0')}`
      showEndDatePicker.value = false
      applyDateFilter()
    }

    const prevMonth = () => {
      if (pickerMonth.value === 1) {
        pickerMonth.value = 12
        pickerYear.value--
      } else {
        pickerMonth.value--
      }
    }

    const nextMonth = () => {
      if (pickerMonth.value === 12) {
        pickerMonth.value = 1
        pickerYear.value++
      } else {
        pickerMonth.value++
      }
    }

    const getDaysInMonth = () => {
      return new Date(pickerYear.value, pickerMonth.value, 0).getDate()
    }

    const isToday = (year, month, day) => {
      const today = new Date()
      return year === today.getFullYear() && 
             month === today.getMonth() + 1 && 
             day === today.getDate()
    }

    return {
      newRecord,
      records,
      addRecord,
      deleteRecord,
      deleteRecordFromList,
      totalAmount,
      subsidyEstimate,
      uniqueDaysCount,
      showMissingRecordModal,
      missingLunchDates,
      missingDinnerDates,
      missingAccommodationDates,
      closeMissingRecordModal,
      confirmMissing,
      showStartDatePicker,
      showEndDatePicker,
      pickerYear,
      pickerMonth,
      selectStartDate,
      selectEndDate,
      prevMonth,
      nextMonth,
      getDaysInMonth,
      isToday,
      paginatedRecords,
      currentPage,
      totalPages,
      goToPage,
      goToPrevious,
      goToNext,
      formatTime,
      payerStats,
      accommodationStats,
      filterStartDate,
      filterEndDate,
      isFiltered,
      filteredPayerStats,
      applyDateFilter,
      resetDateFilter,
      recentDays,
      recentMode,
      recentStats,
      toggleRecentDays,
      switchToSettlement,
      switchToDays,
      lastSettlementTime,
      settlementClickCount,
      showSettlementHint,
      remainingClicks,
      handleSettlementClick,
      resetSettlement,
      showNoteInput,
      noteInput,
      editNote,
      saveNote,
      cancelNote,
      fileInput,
      exportData,
      triggerImport,
      importData,
      editingRecord,
      startEdit,
      saveEdit,
      cancelEdit,
      deleteClickCount,
      recordToDelete
    }
  }
}
</script>

<style scoped>
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.header h1 {
  font-size: 24px;
  color: #333;
}

.subsidy-estimate {
  font-size: 12px;
  color: #f44336;
  margin-top: 8px;
  font-weight: 500;
}

.subsidy-formula {
  font-size: 11px;
  color: #999;
  margin-top: 5px;
  font-style: italic;
}

.data-actions {
  display: flex;
  gap: 10px;
}

.btn-export,
.btn-import {
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  transition: opacity 0.2s;
}

.btn-export {
  background: #4caf50;
  color: white;
}

.btn-import {
  background: #2196f3;
  color: white;
}

.btn-export:hover,
.btn-import:hover {
  opacity: 0.9;
}

.date-filter {
  background: #f9f9f9;
  padding: 15px;
  border-radius: 8px;
}

.date-filter .filter-title {
  font-size: 16px;
  color: #333;
  margin: 0 0 12px 0;
}

.filter-group {
  display: flex;
  gap: 15px;
  align-items: flex-end;
}

.filter-input {
  display: flex;
  flex-direction: column;
  gap: 5px;
  position: relative;
}

.filter-input label {
  font-size: 13px;
  color: #666;
  font-weight: 500;
}

.filter-input input[type="text"] {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 14px;
  background: white;
  cursor: pointer;
  width: 140px;
}

.filter-input input[type="text"]:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 2px rgba(102, 126, 234, 0.2);
}

.date-picker-popup {
  position: absolute;
  top: 100%;
  left: 0;
  background: white;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  padding: 12px;
  z-index: 1000;
  margin-top: 5px;
  width: 220px;
}

.date-picker-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
  font-size: 14px;
  font-weight: 500;
  color: #333;
}

.date-nav-btn {
  background: none;
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 2px 8px;
  cursor: pointer;
  font-size: 14px;
  color: #666;
}

.date-nav-btn:hover {
  background: #f5f5f5;
}

.date-picker-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 4px;
}

.date-picker-day {
  padding: 8px 0;
  text-align: center;
  cursor: pointer;
  border-radius: 4px;
  font-size: 13px;
  color: #333;
  transition: all 0.2s;
}

.date-picker-day:hover {
  background: #667eea;
  color: white;
}

.date-picker-day.selected {
  background: #667eea;
  color: white;
  font-weight: 500;
}

.date-picker-day.today {
  border: 1px solid #667eea;
  color: #667eea;
  font-weight: 500;
}

.btn-reset {
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  background: #999;
  color: white;
  cursor: pointer;
  font-size: 14px;
  transition: opacity 0.2s;
  height: 38px;
}

.btn-reset:hover {
  opacity: 0.9;
}

.missing-record-content {
  max-height: 400px;
  overflow-y: auto;
  padding: 10px 0;
}

.missing-item {
  margin-bottom: 20px;
  padding-bottom: 15px;
  border-bottom: 1px solid #eee;
}

.missing-item:last-child {
  border-bottom: none;
}

.missing-text {
  font-size: 14px;
  color: #333;
  margin-bottom: 10px;
  line-height: 1.5;
}

.btn-confirm-missing {
  padding: 6px 16px;
  border: none;
  border-radius: 6px;
  background: #667eea;
  color: white;
  cursor: pointer;
  font-size: 13px;
  transition: all 0.2s;
}

.btn-confirm-missing:hover {
  background: #5a6fd6;
}

.recent-filter {
  background: #f9f9f9;
  padding: 15px;
  border-radius: 8px;
}

.recent-filter-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.recent-filter .filter-title {
  font-size: 16px;
  color: #333;
  margin: 0;
}

.recent-actions {
  display: flex;
  gap: 8px;
}

.btn-toggle-mode {
  padding: 6px 12px;
  border: 1px solid #4caf50;
  border-radius: 6px;
  background: white;
  color: #4caf50;
  cursor: pointer;
  font-size: 13px;
  transition: all 0.2s;
}

.btn-toggle-mode:hover {
  background: #4caf50;
  color: white;
}

.btn-toggle-days {
  padding: 6px 12px;
  border: 1px solid #667eea;
  border-radius: 6px;
  background: white;
  color: #667eea;
  cursor: pointer;
  font-size: 13px;
  transition: all 0.2s;
}

.btn-toggle-days:hover {
  background: #667eea;
  color: white;
}

.recent-period-info {
  margin-top: 10px;
  font-size: 12px;
  color: #666;
  font-style: italic;
}

.last-settlement {
  background: #fff3e0;
  padding: 15px;
  border-radius: 8px;
  border-left: 4px solid #ff9800;
}

.settlement-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.settlement-header .filter-title {
  font-size: 15px;
  color: #e65100;
  margin: 0;
}

.btn-reset-settlement {
  padding: 4px 10px;
  border: 1px solid #ff9800;
  border-radius: 4px;
  background: white;
  color: #ff9800;
  cursor: pointer;
  font-size: 12px;
  transition: all 0.2s;
}

.btn-reset-settlement:hover {
  background: #ff9800;
  color: white;
}

.settlement-time-wrapper {
  padding: 10px;
  background: white;
  border-radius: 6px;
}

.settlement-time {
  font-size: 16px;
  color: #e65100;
  font-weight: 500;
}

.settlement-hint {
  font-size: 12px;
  color: #ff9800;
  margin-top: 5px;
}

.record-item {
  position: relative;
}

.record-actions {
  display: flex;
  gap: 8px;
}

.btn-edit {
  padding: 4px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
  cursor: pointer;
  font-size: 12px;
}

.btn-edit:hover {
  background: #f5f5f5;
}

.btn-save-edit {
  padding: 4px 12px;
  border: none;
  border-radius: 4px;
  background: #4caf50;
  color: white;
  cursor: pointer;
  font-size: 12px;
}

.btn-save-edit:hover {
  background: #45a049;
}

.btn-cancel-edit {
  padding: 4px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
  cursor: pointer;
  font-size: 12px;
}

.btn-cancel-edit:hover {
  background: #f5f5f5;
}

.edit-time-input {
  padding: 4px 8px;
  border: 1px solid #667eea;
  border-radius: 4px;
  font-size: 13px;
  width: 160px;
}

.edit-amount-input {
  padding: 4px 8px;
  border: 1px solid #667eea;
  border-radius: 4px;
  font-size: 14px;
  width: 100px;
  text-align: right;
}

.edit-category-select {
  padding: 4px 8px;
  border: 1px solid #667eea;
  border-radius: 4px;
  font-size: 13px;
  background: white;
  cursor: pointer;
  margin-bottom: 5px;
  width: 100%;
}

.btn-delete {
  padding: 4px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
  color: #f44336;
  cursor: pointer;
  font-size: 12px;
  transition: all 0.2s;
}

.btn-delete:hover {
  background: #f44336;
  color: white;
}

.btn-delete-warn {
  background: #ff9800;
  color: white;
  border-color: #ff9800;
}

.btn-delete-warn:hover {
  background: #f57c00;
}

.payer-stats-vertical {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.payer-stat-row {
  display: flex;
  align-items: center;
  gap: 15px;
  padding: 8px 12px;
  background: white;
  border-radius: 6px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.08);
}

.payer-name-row {
  font-size: 14px;
  color: #666;
  font-weight: 500;
  min-width: 70px;
}

.payer-amount-row {
  display: flex;
  align-items: baseline;
  gap: 10px;
}

.payer-amount-text {
  font-size: 16px;
  font-weight: 600;
  color: #333;
}

.payer-third-text {
  font-size: 13px;
  color: #999;
}

.accommodation-stats {
  background: #e3f2fd;
  padding: 15px;
  border-radius: 8px;
}
</style>
