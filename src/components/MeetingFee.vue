<template>
  <div class="section">
    <h3 class="section-title">会议费</h3>
    
    <!-- 智能规划按钮 -->
    <div class="planner-trigger">
      <button class="btn btn-primary btn-small" @click="showPlanner = true">
        🧮 智能规划
      </button>
    </div>

    <!-- 智能规划弹窗 -->
    <div v-if="showPlanner" class="modal-overlay" @click.self="showPlanner = false">
      <div class="modal-content">
        <div class="modal-header">
          <h4>会议费智能规划</h4>
          <button class="modal-close" @click="showPlanner = false">×</button>
        </div>
        <div class="modal-body">
          <div v-if="availableMeetingsForPlanner.length === 0" class="no-meetings-hint">
            请先在表格中添加会议名称和标准
          </div>
          <template v-else>
            <div class="planner-row">
              <div class="planner-field">
                <label>目标金额(万)</label>
                <input type="number" v-model.number="plannerTarget" min="0" step="0.1" class="input-small">
              </div>
            </div>
            <div class="planner-step">
              <div class="step-title">各会议参数范围</div>
              <div class="meeting-config-list">
                <div v-for="(config, index) in plannerMeetingConfigs" :key="index" class="meeting-config-item">
                  <span class="meeting-name">{{ config.name || `会议${index+1}` }}</span>
                  <span class="meeting-cost">{{ config.standard }}万/人/天</span>
                  <div class="config-ranges">
                    <div class="range-group">
                      <label>人数</label>
                      <input type="number" v-model.number="config.minAttendees" min="1" class="input-tiny">
                      <span>~</span>
                      <input type="number" v-model.number="config.maxAttendees" min="1" class="input-tiny">
                    </div>
                    <div class="range-group">
                      <label>次数</label>
                      <input type="number" v-model.number="config.minTimes" min="1" class="input-tiny">
                      <span>~</span>
                      <input type="number" v-model.number="config.maxTimes" min="1" class="input-tiny">
                    </div>
                  </div>
                </div>
              </div>
            </div>
            <div class="planner-actions">
              <button class="btn btn-primary btn-small" @click="generateCombinations" :disabled="!canGenerate">
                生成方案
              </button>
            </div>
            
            <!-- 方案结果 -->
            <div v-if="combinations.length > 0" class="combinations-result">
              <div class="step-title">选择方案</div>
              <div class="combination-list">
                <div 
                  v-for="(combo, index) in combinations" 
                  :key="index" 
                  class="combination-card"
                  @click="selectCombination(combo)"
                >
                  <div class="combo-header">
                    <span class="combo-title">方案 {{ index + 1 }}</span>
                    <span class="combo-total">{{ combo.total.toFixed(2) }} 万元</span>
                  </div>
                  <div class="combo-details">
                    <div v-for="(m, mi) in combo.meetings" :key="mi" class="combo-meeting">
                      {{ m.name }}：{{ m.attendees }}人×{{ m.times }}次×{{ m.days }}天 = {{ m.amount.toFixed(2) }}万
                    </div>
                  </div>
                  <button class="btn btn-success btn-small">选择</button>
                </div>
              </div>
            </div>
            <div v-if="noSolution" class="no-solution">
              未找到合适方案，请调整参数范围
            </div>
          </template>
        </div>
        <div class="modal-footer">
          <button class="btn btn-small" @click="showPlanner = false">关闭</button>
        </div>
      </div>
    </div>

    <div class="standard-info">
      <span>默认标准：</span>
      <strong>0.055 万元/人/天</strong>
      <span class="standard-hint">（可在表格中自定义修改）</span>
    </div>
    <div class="table-container">
      <table>
        <thead>
          <tr>
            <th style="width: 40px">序号</th>
            <th style="width: 150px">会议名称</th>
            <th style="width: 50px">次数</th>
            <th style="width: 50px">天数</th>
            <th style="width: 50px">人数</th>
            <th style="width: 80px">标准(万)</th>
            <th style="width: 80px">总计(万)</th>
            <th style="width: 50px">操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in meetings" :key="index">
            <td>{{ index + 1 }}</td>
            <td><input type="text" v-model="item.name" placeholder="名称"></td>
            <td><input type="number" v-model.number="item.times" min="1" step="1"></td>
            <td><input type="number" v-model.number="item.days" min="1" step="1"></td>
            <td><input type="number" v-model.number="item.attendees" min="1" step="1"></td>
            <td><input type="number" v-model.number="item.standard" min="0" step="0.001"></td>
            <td class="amount-display">{{ formatNumber(item.times * item.days * item.attendees * item.standard) }}</td>
            <td><button class="delete-btn" @click="removeRow(index)">删</button></td>
          </tr>
          <tr class="total-row">
            <td colspan="6" style="text-align: right">合计：</td>
            <td class="amount-display">{{ formatNumber(total) }}</td>
            <td></td>
          </tr>
        </tbody>
      </table>
    </div>
    <button class="btn btn-primary btn-small add-row-btn" @click="addRow">+ 新增行</button>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const DEFAULT_STANDARD = 0.055

const props = defineProps({
  modelValue: {
    type: Object,
    default: () => ({
      meetings: []
    })
  }
})

const emit = defineEmits(['update:modelValue'])

const showPlanner = ref(false)
const plannerTarget = ref(2)
const combinations = ref([])
const noSolution = ref(false)
const plannerMeetingConfigs = ref([])

const meetings = ref(props.modelValue.meetings?.length > 0 
  ? props.modelValue.meetings 
  : [{ name: '', times: 1, days: 1, attendees: 1, standard: DEFAULT_STANDARD }])

const formatNumber = (num) => {
  if (!num || isNaN(num)) return '0.00'
  return Number(num).toFixed(2)
}

const total = computed(() => {
  return meetings.value.reduce((sum, item) => {
    return sum + (item.times || 1) * (item.days || 1) * (item.attendees || 1) * (item.standard || 0)
  }, 0)
})

const availableMeetingsForPlanner = computed(() => {
  return meetings.value.filter(m => m.name && m.standard > 0 && m.days > 0)
})

const canGenerate = computed(() => {
  return plannerTarget.value > 0 && availableMeetingsForPlanner.value.length > 0
})

const updatePlannerMeetingConfigs = () => {
  const available = availableMeetingsForPlanner.value
  plannerMeetingConfigs.value = available.map(m => {
    const existing = plannerMeetingConfigs.value.find(c => c.name === m.name)
    return {
      name: m.name,
      days: m.days,
      standard: m.standard,
      minAttendees: existing?.minAttendees || 5,
      maxAttendees: existing?.maxAttendees || 30,
      minTimes: existing?.minTimes || 1,
      maxTimes: existing?.maxTimes || 3,
      meetingItem: m
    }
  })
}

watch(availableMeetingsForPlanner, updatePlannerMeetingConfigs, { deep: true })

const generateCombinations = () => {
  combinations.value = []
  noSolution.value = false
  
  const targetWan = plannerTarget.value
  const configs = plannerMeetingConfigs.value
  const results = []
  const seenKeys = new Set()
  
  const tryAddResult = (meetingResults, totalCost) => {
    const deviation = Math.abs(totalCost - targetWan) / targetWan
    if (deviation <= 0.15) {
      const key = meetingResults.map(m => `${m.name}-${m.attendees}-${m.times}`).join('|')
      if (!seenKeys.has(key)) {
        seenKeys.add(key)
        results.push({
          meetings: [...meetingResults],
          total: totalCost,
          deviation: deviation
        })
      }
    }
  }
  
  const generateForMeeting = (meetingIndex, currentResults, currentTotal) => {
    if (meetingIndex >= configs.length) {
      tryAddResult(currentResults, currentTotal)
      return
    }
    
    const config = configs[meetingIndex]
    const costPerPersonDay = config.standard
    const days = config.days
    
    for (let attendees = config.minAttendees; attendees <= config.maxAttendees; attendees += 5) {
      for (let times = config.minTimes; times <= config.maxTimes; times++) {
        const cost = costPerPersonDay * attendees * days * times
        
        if (currentTotal + cost > targetWan * 1.5) continue
        
        const newResults = [...currentResults, {
          name: config.name,
          attendees: attendees,
          times: times,
          days: days,
          amount: cost,
          meetingItem: config.meetingItem
        }]
        
        if (meetingIndex === configs.length - 1) {
          tryAddResult(newResults, currentTotal + cost)
        } else if (results.length < 50) {
          generateForMeeting(meetingIndex + 1, newResults, currentTotal + cost)
        }
      }
    }
  }
  
  generateForMeeting(0, [], 0)
  
  results.sort((a, b) => a.deviation - b.deviation)
  combinations.value = results.slice(0, 8)
  
  if (combinations.value.length === 0) {
    noSolution.value = true
  }
}

const selectCombination = (combo) => {
  if (!confirm('将更新表格中对应会议的人数和次数，是否继续？')) {
    return
  }
  
  for (const plan of combo.meetings) {
    const meeting = meetings.value.find(m => m.name === plan.name)
    if (meeting) {
      meeting.attendees = plan.attendees
      meeting.times = plan.times
    }
  }
  
  showPlanner.value = false
  combinations.value = []
}

const addRow = () => {
  meetings.value.push({ name: '', times: 1, days: 1, attendees: 1, standard: DEFAULT_STANDARD })
}

const removeRow = (index) => {
  if (meetings.value.length > 1) {
    meetings.value.splice(index, 1)
  }
}

watch(meetings, () => {
  emit('update:modelValue', {
    meetings: meetings.value,
    totalWan: total.value
  })
}, { deep: true, immediate: true })

watch(() => props.modelValue, (newVal) => {
  if (newVal.meetings) meetings.value = newVal.meetings
}, { deep: true })
</script>

<style scoped>
.standard-info {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 10px;
  font-size: 13px;
  color: #555;
}

.standard-info strong {
  color: #2d3748;
}

.standard-hint {
  color: #888;
  font-size: 12px;
}

.planner-trigger {
  margin-bottom: 12px;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: white;
  border-radius: 8px;
  width: 600px;
  max-width: 90%;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  border-bottom: 1px solid #eee;
  background: #e8f5e9;
  border-radius: 8px 8px 0 0;
}

.modal-header h4 {
  margin: 0;
  font-size: 16px;
  color: #2e7d32;
}

.modal-close {
  background: none;
  border: none;
  font-size: 24px;
  color: #999;
  cursor: pointer;
  padding: 0;
  line-height: 1;
}

.modal-close:hover {
  color: #333;
}

.modal-body {
  padding: 20px;
}

.modal-footer {
  padding: 15px 20px;
  border-top: 1px solid #eee;
  text-align: right;
}

.planner-row {
  display: flex;
  gap: 20px;
  margin-bottom: 10px;
}

.planner-field {
  display: flex;
  align-items: center;
  gap: 6px;
}

.planner-field label {
  font-size: 12px;
  color: #555;
  white-space: nowrap;
}

.input-small {
  width: 60px !important;
  text-align: right;
}

.input-tiny {
  width: 50px !important;
  text-align: center;
}

.planner-step {
  margin-bottom: 10px;
}

.step-title {
  font-size: 12px;
  font-weight: 500;
  color: #444;
  margin-bottom: 6px;
}

.meeting-config-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.meeting-config-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 6px 8px;
  background: #f1f8e9;
  border-radius: 4px;
  flex-wrap: wrap;
}

.meeting-name {
  font-weight: 500;
  color: #333;
  min-width: 80px;
  font-size: 12px;
}

.meeting-cost {
  font-size: 11px;
  color: #666;
  min-width: 80px;
}

.config-ranges {
  display: flex;
  gap: 12px;
}

.range-group {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 11px;
}

.range-group label {
  color: #666;
}

.range-group span {
  color: #999;
}

.planner-actions {
  margin-top: 8px;
}

.no-meetings-hint {
  padding: 12px;
  background: #fff3e0;
  border: 1px solid #ffcc80;
  border-radius: 4px;
  color: #e65100;
  font-size: 12px;
  text-align: center;
}

.combinations-result {
  margin-top: 12px;
  padding-top: 10px;
  border-top: 1px solid #a5d6a7;
}

.combination-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.combination-card {
  background: white;
  border: 1px solid #c8e6c9;
  border-radius: 6px;
  padding: 8px 10px;
  cursor: pointer;
  transition: all 0.2s;
  min-width: 180px;
}

.combination-card:hover {
  border-color: #4caf50;
  box-shadow: 0 2px 6px rgba(76, 175, 80, 0.2);
}

.combo-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 6px;
}

.combo-title {
  font-weight: 600;
  color: #2e7d32;
  font-size: 12px;
}

.combo-total {
  font-size: 12px;
  color: #1b5e20;
  font-weight: 500;
}

.combo-details {
  margin-bottom: 6px;
}

.combo-meeting {
  font-size: 11px;
  color: #555;
  line-height: 1.4;
}

.no-solution {
  padding: 10px;
  background: #ffebee;
  border: 1px solid #ef9a9a;
  border-radius: 4px;
  color: #c62828;
  font-size: 12px;
  text-align: center;
  margin-top: 10px;
}
</style>
