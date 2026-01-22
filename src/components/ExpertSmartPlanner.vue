<template>
  <div v-if="visible" class="modal-overlay" @click.self="close">
    <div class="modal-content">
      <div class="modal-header">
        <h4>专家咨询费智能规划</h4>
        <button class="modal-close" @click="close">×</button>
      </div>
      <div class="modal-body">
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
          <div class="step-title-row">
            <span class="step-title">各会议参数设置（人数为奇数）</span>
            <button class="btn btn-small btn-add" @click="addTempRow">+ 添加临时行</button>
          </div>
          <div class="expert-config-table">
            <div class="config-header">
              <span class="col-enable">参与</span>
              <span class="col-name">会议内容</span>
              <span class="col-cost">标准</span>
              <span class="col-days">天数</span>
              <span class="col-param">人数</span>
              <span class="col-param">次数</span>
              <span class="col-action">操作</span>
            </div>
            <div v-for="(config, index) in expertConfigs" :key="config.id || index" class="config-row" :class="{ 'row-disabled': !config.enabled, 'row-temp': config.isTemp }">
              <div class="col-enable">
                <input type="checkbox" v-model="config.enabled" title="是否参与规划">
              </div>
              <div class="col-name">
                <span v-if="!config.isTemp" class="name-value">{{ config.name }}</span>
                <input v-else type="text" v-model="config.name" placeholder="会议名称" class="input-name">
              </div>
              <div class="col-cost">
                <span v-if="!config.isTemp" class="cost-value">{{ config.standard }}元</span>
                <input v-else type="number" v-model.number="config.standard" min="0" step="100" class="input-cost" placeholder="元">
              </div>
              <div class="col-days">
                <span v-if="!config.isTemp" class="days-value">{{ config.days }}天</span>
                <input v-else type="number" v-model.number="config.days" min="1" step="1" class="input-days">
              </div>
              <div class="col-param">
                <div class="param-control">
                  <label class="lock-label" :class="{ locked: config.peopleLocked }">
                    <input type="checkbox" v-model="config.peopleLocked" title="锁定为固定值">
                    <span class="lock-icon">{{ config.peopleLocked ? '🔒' : '🔓' }}</span>
                  </label>
                  <template v-if="config.peopleLocked">
                    <input type="number" v-model.number="config.peopleFixed" min="0" step="2" class="input-tiny" :disabled="!config.enabled">
                  </template>
                  <template v-else>
                    <input type="number" v-model.number="config.minPeople" min="1" step="2" class="input-tiny" :disabled="!config.enabled">
                    <span class="range-sep">~</span>
                    <input type="number" v-model.number="config.maxPeople" min="1" step="2" class="input-tiny" :disabled="!config.enabled">
                  </template>
                </div>
              </div>
              <div class="col-param">
                <div class="param-control">
                  <label class="lock-label" :class="{ locked: config.timesLocked }">
                    <input type="checkbox" v-model="config.timesLocked" title="锁定为固定值">
                    <span class="lock-icon">{{ config.timesLocked ? '🔒' : '🔓' }}</span>
                  </label>
                  <template v-if="config.timesLocked">
                    <input type="number" v-model.number="config.timesFixed" min="0" step="1" class="input-tiny" :disabled="!config.enabled">
                  </template>
                  <template v-else>
                    <input type="number" v-model.number="config.minTimes" min="1" step="1" class="input-tiny" :disabled="!config.enabled">
                    <span class="range-sep">~</span>
                    <input type="number" v-model.number="config.maxTimes" min="1" step="1" class="input-tiny" :disabled="!config.enabled">
                  </template>
                </div>
              </div>
              <div class="col-action">
                <button v-if="config.isTemp" class="btn-delete" @click="removeTempRow(index)" title="删除临时行">×</button>
              </div>
            </div>
          </div>
          <div class="config-hint">
            💡 提示：勾选"参与"列控制是否参与规划；点击🔓可锁定变量为固定值（设为0表示不参与）
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
                <div v-for="(e, ei) in combo.experts" :key="ei" class="combo-expert" :class="{ 'item-zero': e.people === 0 || e.times === 0 }">
                  {{ e.name }}：{{ e.people }}人×{{ e.days }}天×{{ e.times }}次 = {{ e.amount.toFixed(2) }}万
                  <span v-if="e.isTemp" class="temp-badge">临时</span>
                </div>
              </div>
              <button class="btn btn-success btn-small">选择</button>
            </div>
          </div>
        </div>
        <div v-if="noSolution" class="no-solution">
          未找到合适方案，请调整参数范围
        </div>
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
  experts: {
    type: Array,
    default: () => []
  },
  plannerConfig: {
    type: Object,
    default: () => ({
      targetAmount: 2,
      tolerancePercent: 15,
      expertConfigs: []
    })
  }
})

const emit = defineEmits(['update:visible', 'select', 'update:plannerConfig'])

const targetAmount = ref(props.plannerConfig?.targetAmount ?? 2)
const tolerancePercent = ref(props.plannerConfig?.tolerancePercent ?? 15)
const combinations = ref([])
const noSolution = ref(false)
const expertConfigs = ref([])
const isGenerating = ref(false)
const generationTime = ref(0)
let tempIdCounter = 0

// 算法参数
const SAMPLE_COUNT = 3000
const TOP_CANDIDATES = 50
const LOCAL_OPT_ITERATIONS = 15
const MAX_RESULTS = 10

const availableExperts = computed(() => {
  return props.experts.filter(e => e.name && e.standard > 0 && e.days > 0)
})

const enabledConfigs = computed(() => {
  return expertConfigs.value.filter(c => c.enabled)
})

const canGenerate = computed(() => {
  if (targetAmount.value <= 0) return false
  const enabled = enabledConfigs.value
  if (enabled.length === 0) return false
  
  return enabled.every(c => {
    if (c.peopleLocked && c.peopleFixed < 0) return false
    if (c.timesLocked && c.timesFixed < 0) return false
    if (!c.peopleLocked && (c.minPeople < 1 || c.maxPeople < c.minPeople)) return false
    if (!c.timesLocked && (c.minTimes < 1 || c.maxTimes < c.minTimes)) return false
    return true
  })
})

const getEffectiveRange = (config, field) => {
  const lockedKey = field + 'Locked'
  const fixedKey = field + 'Fixed'
  const minKey = 'min' + field.charAt(0).toUpperCase() + field.slice(1)
  const maxKey = 'max' + field.charAt(0).toUpperCase() + field.slice(1)
  
  if (config[lockedKey]) {
    const fixed = config[fixedKey]
    return { min: fixed, max: fixed }
  }
  return { min: config[minKey], max: config[maxKey] }
}

const updateExpertConfigs = () => {
  const available = availableExperts.value
  const savedConfigs = props.plannerConfig?.expertConfigs || []
  
  const tempRows = expertConfigs.value.filter(c => c.isTemp)
  
  const newConfigs = available.map(e => {
    const savedConfig = savedConfigs.find(c => c.name === e.name && !c.isTemp)
    const existing = expertConfigs.value.find(c => c.name === e.name && !c.isTemp)
    
    return {
      id: e.name + '_' + Date.now(),
      name: e.name,
      days: e.days,
      standard: e.standard,
      isTemp: false,
      enabled: savedConfig?.enabled ?? existing?.enabled ?? true,
      peopleLocked: savedConfig?.peopleLocked ?? existing?.peopleLocked ?? false,
      peopleFixed: savedConfig?.peopleFixed ?? existing?.peopleFixed ?? 5,
      minPeople: savedConfig?.minPeople ?? existing?.minPeople ?? 3,
      maxPeople: savedConfig?.maxPeople ?? existing?.maxPeople ?? 9,
      timesLocked: savedConfig?.timesLocked ?? existing?.timesLocked ?? false,
      timesFixed: savedConfig?.timesFixed ?? existing?.timesFixed ?? 1,
      minTimes: savedConfig?.minTimes ?? existing?.minTimes ?? 1,
      maxTimes: savedConfig?.maxTimes ?? existing?.maxTimes ?? 3,
      expertItem: e
    }
  })
  
  const savedTempRows = savedConfigs.filter(c => c.isTemp)
  for (const saved of savedTempRows) {
    if (!tempRows.find(t => t.id === saved.id)) {
      tempRows.push({ ...saved, expertItem: null })
    }
  }
  
  expertConfigs.value = [...newConfigs, ...tempRows]
}

const addTempRow = () => {
  tempIdCounter++
  expertConfigs.value.push({
    id: 'temp_' + tempIdCounter + '_' + Date.now(),
    name: '',
    days: 1,
    standard: 2800,
    isTemp: true,
    enabled: true,
    peopleLocked: false,
    peopleFixed: 5,
    minPeople: 3,
    maxPeople: 9,
    timesLocked: false,
    timesFixed: 1,
    minTimes: 1,
    maxTimes: 3,
    expertItem: null
  })
}

const removeTempRow = (index) => {
  expertConfigs.value.splice(index, 1)
}

const emitConfigUpdate = () => {
  const configToSave = {
    targetAmount: targetAmount.value,
    tolerancePercent: tolerancePercent.value,
    expertConfigs: expertConfigs.value.map(c => ({
      id: c.id,
      name: c.name,
      isTemp: c.isTemp,
      days: c.days,
      standard: c.standard,
      enabled: c.enabled,
      peopleLocked: c.peopleLocked,
      peopleFixed: c.peopleFixed,
      minPeople: c.minPeople,
      maxPeople: c.maxPeople,
      timesLocked: c.timesLocked,
      timesFixed: c.timesFixed,
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
    updateExpertConfigs()
    combinations.value = []
    noSolution.value = false
  }
})

watch(() => props.experts, updateExpertConfigs, { deep: true })

watch([targetAmount, tolerancePercent], emitConfigUpdate)
watch(expertConfigs, emitConfigUpdate, { deep: true })

const close = () => {
  emit('update:visible', false)
}

const getDeviationClass = (deviation) => {
  const absDeviation = Math.abs(deviation)
  if (absDeviation <= 0.05) return 'deviation-good'
  if (absDeviation <= 0.10) return 'deviation-ok'
  return 'deviation-warn'
}

const calculateSolutionCost = (solution, configs) => {
  let total = 0
  for (let i = 0; i < solution.length; i++) {
    const { people, times } = solution[i]
    const config = configs[i]
    total += (config.standard / 10000) * people * config.days * times
  }
  return total
}

const generateRandomSolution = (configs) => {
  const solution = []
  for (const config of configs) {
    const peopleRange = getEffectiveRange(config, 'people')
    const timesRange = getEffectiveRange(config, 'times')
    
    // 生成奇数人数
    let people = randomInt(peopleRange.min, peopleRange.max)
    if (!config.peopleLocked && people > 0 && people % 2 === 0) {
      people = Math.max(peopleRange.min, people - 1)
      if (people % 2 === 0) people = Math.min(peopleRange.max, people + 1)
    }
    const times = randomInt(timesRange.min, timesRange.max)
    solution.push({ people, times })
  }
  return solution
}

const randomInt = (min, max) => {
  if (min > max) return min
  return Math.floor(Math.random() * (max - min + 1)) + min
}

const localOptimize = (solution, targetWan, configs) => {
  let currentSolution = solution.map(s => ({ ...s }))
  let currentCost = calculateSolutionCost(currentSolution, configs)
  let currentGap = Math.abs(currentCost - targetWan)
  
  for (let iter = 0; iter < LOCAL_OPT_ITERATIONS; iter++) {
    let improved = false
    
    for (let i = 0; i < currentSolution.length; i++) {
      const config = configs[i]
      const original = { ...currentSolution[i] }
      
      const adjustments = []
      if (!config.peopleLocked) {
        adjustments.push({ field: 'people', delta: 2 })
        adjustments.push({ field: 'people', delta: -2 })
      }
      if (!config.timesLocked) {
        adjustments.push({ field: 'times', delta: 1 })
        adjustments.push({ field: 'times', delta: -1 })
      }
      
      for (const adj of adjustments) {
        const newValue = original[adj.field] + adj.delta
        const range = getEffectiveRange(config, adj.field)
        
        if (newValue < range.min || newValue > range.max) continue
        
        currentSolution[i][adj.field] = newValue
        const newCost = calculateSolutionCost(currentSolution, configs)
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
  const configs = enabledConfigs.value
  if (configs.length === 0) return []
  
  const candidates = []
  
  for (let i = 0; i < SAMPLE_COUNT; i++) {
    const solution = generateRandomSolution(configs)
    const cost = calculateSolutionCost(solution, configs)
    const gap = Math.abs(cost - targetWan)
    candidates.push({ solution, cost, gap })
  }
  
  candidates.sort((a, b) => a.gap - b.gap)
  const bestCandidates = candidates.slice(0, TOP_CANDIDATES)
  
  const optimized = []
  const seenKeys = new Set()
  
  for (const candidate of bestCandidates) {
    const result = localOptimize(candidate.solution, targetWan, configs)
    
    const key = result.solution.map(s => `${s.people}-${s.times}`).join('|')
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
  const configs = enabledConfigs.value
  
  return results.map(r => {
    const experts = r.solution.map((s, i) => {
      const config = configs[i]
      const amount = (config.standard / 10000) * s.people * config.days * s.times
      return {
        name: config.name || '(未命名)',
        people: s.people,
        days: config.days,
        times: s.times,
        amount: amount,
        expertItem: config.expertItem,
        isTemp: config.isTemp,
        configId: config.id
      }
    })
    
    return {
      experts,
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
  const hasTemp = combo.experts.some(e => e.isTemp)
  const msg = hasTemp 
    ? '将更新表格中对应会议的人数和次数。临时行数据需要手动添加到表格中。是否继续？'
    : '将更新表格中对应会议的人数和次数，是否继续？'
  
  if (!confirm(msg)) return
  
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
  width: 800px;
  max-width: 95%;
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
  background: #fff3e0;
  border-radius: 8px 8px 0 0;
}

.modal-header h4 {
  margin: 0;
  font-size: 16px;
  color: #e65100;
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
  width: 42px !important;
  text-align: center;
  padding: 2px 4px !important;
}

.planner-step {
  margin-bottom: 10px;
}

.step-title-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.step-title {
  font-size: 12px;
  font-weight: 500;
  color: #444;
}

.btn-add {
  background: #ff9800;
  color: white;
}

.btn-add:hover {
  background: #f57c00;
}

.expert-config-table {
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  overflow: hidden;
}

.config-header {
  display: flex;
  background: #f5f5f5;
  padding: 8px 10px;
  font-size: 12px;
  font-weight: 500;
  color: #555;
  border-bottom: 1px solid #e0e0e0;
}

.config-row {
  display: flex;
  padding: 8px 10px;
  align-items: center;
  border-bottom: 1px solid #f0f0f0;
}

.config-row:last-child {
  border-bottom: none;
}

.config-row:hover {
  background: #fafafa;
}

.row-disabled {
  opacity: 0.5;
  background: #f9f9f9;
}

.row-temp {
  background: #fff8e1;
}

.row-temp:hover {
  background: #fff3c4;
}

.col-enable {
  width: 40px;
  flex-shrink: 0;
  text-align: center;
}

.col-name {
  width: 100px;
  flex-shrink: 0;
}

.col-cost {
  width: 60px;
  flex-shrink: 0;
}

.col-days {
  width: 50px;
  flex-shrink: 0;
}

.col-param {
  flex: 1;
  min-width: 110px;
}

.col-action {
  width: 40px;
  flex-shrink: 0;
  text-align: center;
}

.name-value {
  font-weight: 500;
  color: #333;
  font-size: 12px;
}

.cost-value, .days-value {
  font-size: 11px;
  color: #666;
}

.input-name {
  width: 90px !important;
}

.input-cost {
  width: 50px !important;
  text-align: right;
}

.input-days {
  width: 40px !important;
  text-align: center;
}

.param-control {
  display: flex;
  align-items: center;
  gap: 3px;
}

.lock-label {
  cursor: pointer;
  display: flex;
  align-items: center;
}

.lock-label input {
  display: none;
}

.lock-icon {
  font-size: 12px;
  opacity: 0.6;
  transition: opacity 0.2s;
}

.lock-label:hover .lock-icon {
  opacity: 1;
}

.lock-label.locked .lock-icon {
  opacity: 1;
}

.range-sep {
  font-size: 10px;
  color: #999;
}

.config-hint {
  margin-top: 8px;
  font-size: 11px;
  color: #888;
}

.planner-actions {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-top: 12px;
}

.generation-time {
  font-size: 12px;
  color: #888;
}

.btn-delete {
  background: #ff5252;
  color: white;
  border: none;
  border-radius: 4px;
  width: 22px;
  height: 22px;
  cursor: pointer;
  font-size: 14px;
  line-height: 1;
}

.btn-delete:hover {
  background: #d32f2f;
}

.combinations-result {
  margin-top: 12px;
  padding-top: 10px;
  border-top: 1px solid #ffcc80;
}

.combination-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.combination-card {
  background: white;
  border: 1px solid #ffe0b2;
  border-radius: 6px;
  padding: 8px 10px;
  cursor: pointer;
  transition: all 0.2s;
  min-width: 200px;
  max-width: 280px;
}

.combination-card:hover {
  border-color: #ff9800;
  box-shadow: 0 2px 6px rgba(255, 152, 0, 0.2);
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
  color: #e65100;
  font-size: 12px;
}

.combo-total {
  font-size: 12px;
  color: #bf360c;
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

.combo-expert {
  font-size: 11px;
  color: #555;
  line-height: 1.4;
}

.combo-expert.item-zero {
  color: #999;
  text-decoration: line-through;
}

.temp-badge {
  display: inline-block;
  background: #ff9800;
  color: white;
  font-size: 10px;
  padding: 1px 4px;
  border-radius: 3px;
  margin-left: 4px;
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
  background: #ff9800;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #f57c00;
}

.btn-primary:disabled {
  background: #ffe0b2;
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
