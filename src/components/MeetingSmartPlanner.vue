<template>
  <div v-if="visible" class="modal-overlay" @click.self="close">
    <div class="modal-content">
      <div class="modal-header">
        <h4>会议费智能规划</h4>
        <button class="modal-close" @click="close">×</button>
      </div>
      <div class="modal-body">
        <div v-if="meetingConfigs.length === 0" class="no-meetings-hint">
          请先在表格中添加会议名称和标准
        </div>
        <template v-else>
          <div class="planner-row">
            <div class="planner-field">
              <label>目标金额(万)</label>
              <input type="number" v-model.number="targetAmount" min="0" step="0.1" class="input-small">
            </div>
            <div class="planner-field">
              <label>容差范围</label>
              <select v-model.number="tolerancePercent" class="input-small">
                <option :value="5">±5%</option>
                <option :value="10">±10%</option>
                <option :value="15">±15%</option>
                <option :value="20">±20%</option>
              </select>
            </div>
          </div>
          <div class="planner-step">
            <div class="step-title">各会议参数范围</div>
            <div class="meeting-config-list">
              <div v-for="(config, index) in meetingConfigs" :key="index" class="meeting-config-item">
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
            <button class="btn btn-primary btn-small" @click="generateCombinations" :disabled="!canGenerate || isGenerating">
              {{ isGenerating ? '计算中...' : '生成方案' }}
            </button>
            <span v-if="generationTime" class="generation-time">耗时: {{ generationTime }}ms</span>
          </div>
          
          <!-- 方案结果 -->
          <div v-if="combinations.length > 0" class="combinations-result">
            <div class="step-title">选择方案 (共{{ combinations.length }}个)</div>
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
                  <span class="combo-deviation" :class="getDeviationClass(combo.deviation)">
                    {{ combo.deviation > 0 ? '+' : '' }}{{ (combo.deviation * 100).toFixed(1) }}%
                  </span>
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
        <button class="btn btn-small" @click="close">关闭</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  visible: {
    type: Boolean,
    default: false
  },
  meetings: {
    type: Array,
    default: () => []
  },
  plannerConfig: {
    type: Object,
    default: () => ({
      targetAmount: 2,
      tolerancePercent: 15,
      meetingConfigs: []
    })
  }
})

const emit = defineEmits(['update:visible', 'select', 'update:plannerConfig'])

const targetAmount = ref(props.plannerConfig?.targetAmount ?? 2)
const tolerancePercent = ref(props.plannerConfig?.tolerancePercent ?? 15)
const combinations = ref([])
const noSolution = ref(false)
const meetingConfigs = ref([])
const isGenerating = ref(false)
const generationTime = ref(0)

// 算法参数
const SAMPLE_COUNT = 3000
const TOP_CANDIDATES = 50
const LOCAL_OPT_ITERATIONS = 15
const MAX_RESULTS = 10

const availableMeetings = computed(() => {
  return props.meetings.filter(m => m.name && m.standard > 0 && m.days > 0)
})

const canGenerate = computed(() => {
  if (targetAmount.value <= 0 || meetingConfigs.value.length === 0) {
    return false
  }
  return meetingConfigs.value.every(c => 
    c.minAttendees >= 1 && c.maxAttendees >= c.minAttendees &&
    c.minTimes >= 1 && c.maxTimes >= c.minTimes
  )
})

const updateMeetingConfigs = () => {
  const available = availableMeetings.value
  const savedConfigs = props.plannerConfig?.meetingConfigs || []
  
  meetingConfigs.value = available.map(m => {
    const savedConfig = savedConfigs.find(c => c.name === m.name)
    const existing = meetingConfigs.value.find(c => c.name === m.name)
    
    return {
      name: m.name,
      days: m.days,
      standard: m.standard,
      minAttendees: savedConfig?.minAttendees ?? existing?.minAttendees ?? 5,
      maxAttendees: savedConfig?.maxAttendees ?? existing?.maxAttendees ?? 30,
      minTimes: savedConfig?.minTimes ?? existing?.minTimes ?? 1,
      maxTimes: savedConfig?.maxTimes ?? existing?.maxTimes ?? 3,
      meetingItem: m
    }
  })
}

const emitConfigUpdate = () => {
  const configToSave = {
    targetAmount: targetAmount.value,
    tolerancePercent: tolerancePercent.value,
    meetingConfigs: meetingConfigs.value.map(c => ({
      name: c.name,
      minAttendees: c.minAttendees,
      maxAttendees: c.maxAttendees,
      minTimes: c.minTimes,
      maxTimes: c.maxTimes
    }))
  }
  emit('update:plannerConfig', configToSave)
}

watch(() => props.visible, (val) => {
  if (val) {
    if (props.plannerConfig) {
      targetAmount.value = props.plannerConfig.targetAmount ?? 2
      tolerancePercent.value = props.plannerConfig.tolerancePercent ?? 15
    }
    updateMeetingConfigs()
    combinations.value = []
    noSolution.value = false
  }
})

watch(() => props.meetings, updateMeetingConfigs, { deep: true })

watch([targetAmount, tolerancePercent], emitConfigUpdate)
watch(meetingConfigs, emitConfigUpdate, { deep: true })

const close = () => {
  emit('update:visible', false)
}

const getDeviationClass = (deviation) => {
  const absDeviation = Math.abs(deviation)
  if (absDeviation <= 0.05) return 'deviation-good'
  if (absDeviation <= 0.10) return 'deviation-ok'
  return 'deviation-warn'
}

const calculateSolutionCost = (solution) => {
  let total = 0
  for (let i = 0; i < solution.length; i++) {
    const { attendees, times } = solution[i]
    const config = meetingConfigs.value[i]
    total += config.standard * attendees * config.days * times
  }
  return total
}

const generateRandomSolution = () => {
  const solution = []
  for (const config of meetingConfigs.value) {
    const attendees = randomInt(config.minAttendees, config.maxAttendees)
    const times = randomInt(config.minTimes, config.maxTimes)
    solution.push({ attendees, times })
  }
  return solution
}

const randomInt = (min, max) => {
  return Math.floor(Math.random() * (max - min + 1)) + min
}

const localOptimize = (solution, targetWan) => {
  let currentSolution = solution.map(s => ({ ...s }))
  let currentCost = calculateSolutionCost(currentSolution)
  let currentGap = Math.abs(currentCost - targetWan)
  
  for (let iter = 0; iter < LOCAL_OPT_ITERATIONS; iter++) {
    let improved = false
    
    for (let i = 0; i < currentSolution.length; i++) {
      const config = meetingConfigs.value[i]
      const original = { ...currentSolution[i] }
      
      const adjustments = [
        { field: 'attendees', delta: 1 },
        { field: 'attendees', delta: -1 },
        { field: 'attendees', delta: 5 },
        { field: 'attendees', delta: -5 },
        { field: 'times', delta: 1 },
        { field: 'times', delta: -1 }
      ]
      
      for (const adj of adjustments) {
        const newValue = original[adj.field] + adj.delta
        
        if (adj.field === 'attendees' && (newValue < config.minAttendees || newValue > config.maxAttendees)) continue
        if (adj.field === 'times' && (newValue < config.minTimes || newValue > config.maxTimes)) continue
        
        currentSolution[i][adj.field] = newValue
        const newCost = calculateSolutionCost(currentSolution)
        const newGap = Math.abs(newCost - targetWan)
        
        if (newGap < currentGap) {
          currentCost = newCost
          currentGap = newGap
          improved = true
          break
        } else {
          currentSolution[i][adj.field] = original[adj.field]
        }
      }
      
      if (improved) break
    }
    
    if (!improved) break
  }
  
  return { solution: currentSolution, cost: currentCost, gap: currentGap }
}

const findBudgetCombinations = (targetWan) => {
  const candidates = []
  
  for (let i = 0; i < SAMPLE_COUNT; i++) {
    const solution = generateRandomSolution()
    const cost = calculateSolutionCost(solution)
    const gap = Math.abs(cost - targetWan)
    candidates.push({ solution, cost, gap })
  }
  
  candidates.sort((a, b) => a.gap - b.gap)
  const bestCandidates = candidates.slice(0, TOP_CANDIDATES)
  
  const optimized = []
  const seenKeys = new Set()
  
  for (const candidate of bestCandidates) {
    const result = localOptimize(candidate.solution, targetWan)
    
    const key = result.solution.map(s => `${s.attendees}-${s.times}`).join('|')
    if (seenKeys.has(key)) continue
    seenKeys.add(key)
    
    optimized.push(result)
  }
  
  optimized.sort((a, b) => a.gap - b.gap)
  
  const tolerance = targetWan * (tolerancePercent.value / 100)
  const filtered = optimized.filter(r => r.gap <= tolerance)
  
  let results = filtered
  if (results.length < 3 && optimized.length > 0) {
    const bestGap = optimized[0].gap
    results = optimized.filter(r => r.gap <= bestGap * 1.5)
  }
  
  return results.slice(0, MAX_RESULTS)
}

const formatResults = (results, targetWan) => {
  return results.map(r => {
    const meetings = r.solution.map((s, i) => {
      const config = meetingConfigs.value[i]
      const amount = config.standard * s.attendees * config.days * s.times
      return {
        name: config.name,
        attendees: s.attendees,
        times: s.times,
        days: config.days,
        amount: amount,
        meetingItem: config.meetingItem
      }
    })
    
    return {
      meetings,
      total: r.cost,
      deviation: (r.cost - targetWan) / targetWan
    }
  })
}

const generateCombinations = () => {
  combinations.value = []
  noSolution.value = false
  isGenerating.value = true
  generationTime.value = 0
  
  const startTime = performance.now()
  
  setTimeout(() => {
    try {
      const targetWan = targetAmount.value
      const results = findBudgetCombinations(targetWan)
      
      if (results.length === 0) {
        noSolution.value = true
      } else {
        combinations.value = formatResults(results, targetWan)
      }
      
      generationTime.value = Math.round(performance.now() - startTime)
    } finally {
      isGenerating.value = false
    }
  }, 10)
}

const selectCombination = (combo) => {
  if (!confirm('将更新表格中对应会议的人数和次数，是否继续？')) {
    return
  }
  
  emit('select', combo)
  close()
}
</script>

<style scoped>
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
  width: 650px;
  max-width: 90%;
  max-height: 85vh;
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
  width: 70px !important;
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
  display: flex;
  align-items: center;
  gap: 10px;
  margin-top: 8px;
}

.generation-time {
  font-size: 12px;
  color: #888;
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
  min-width: 200px;
  max-width: 280px;
}

.combination-card:hover {
  border-color: #4caf50;
  box-shadow: 0 2px 6px rgba(76, 175, 80, 0.2);
}

.combo-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 6px;
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

.combo-deviation {
  font-size: 11px;
  padding: 2px 6px;
  border-radius: 10px;
}

.deviation-good {
  background: #c8e6c9;
  color: #2e7d32;
}

.deviation-ok {
  background: #fff3e0;
  color: #e65100;
}

.deviation-warn {
  background: #ffcdd2;
  color: #c62828;
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

.btn {
  padding: 6px 12px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 13px;
}

.btn-primary {
  background: #4caf50;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #43a047;
}

.btn-primary:disabled {
  background: #a5d6a7;
  cursor: not-allowed;
}

.btn-success {
  background: #2e7d32;
  color: white;
}

.btn-success:hover {
  background: #1b5e20;
}

.btn-small {
  padding: 4px 10px;
  font-size: 12px;
}
</style>
