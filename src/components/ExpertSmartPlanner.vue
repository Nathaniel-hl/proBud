<template>
  <div v-if="visible" class="modal-overlay" @click.self="close">
    <div class="modal-content">
      <div class="modal-header">
        <h4>专家咨询费智能规划</h4>
        <button class="modal-close" @click="close">×</button>
      </div>
      <div class="modal-body">
        <div v-if="expertConfigs.length === 0" class="no-data-hint">
          请先在表格中添加会议内容和人均标准
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
            <div class="step-title">各会议参数范围（人数为奇数）</div>
            <div class="expert-config-list">
              <div v-for="(config, index) in expertConfigs" :key="index" class="expert-config-item">
                <span class="expert-name">{{ config.name }}</span>
                <span class="expert-cost">{{ config.standard }}元/人/天</span>
                <div class="config-ranges">
                  <div class="range-group">
                    <label>人数</label>
                    <input type="number" v-model.number="config.minPeople" min="1" step="2" class="input-tiny">
                    <span>~</span>
                    <input type="number" v-model.number="config.maxPeople" min="1" step="2" class="input-tiny">
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
                  <div v-for="(e, ei) in combo.experts" :key="ei" class="combo-expert">
                    {{ e.name }}：{{ e.people }}人×{{ e.days }}天×{{ e.times }}次 = {{ e.amount.toFixed(2) }}万
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

// 算法参数
const SAMPLE_COUNT = 3000
const TOP_CANDIDATES = 50
const LOCAL_OPT_ITERATIONS = 15
const MAX_RESULTS = 10

const availableExperts = computed(() => {
  return props.experts.filter(e => e.name && e.standard > 0 && e.days > 0)
})

const canGenerate = computed(() => {
  if (targetAmount.value <= 0 || expertConfigs.value.length === 0) {
    return false
  }
  return expertConfigs.value.every(c => 
    c.minPeople >= 1 && c.maxPeople >= c.minPeople &&
    c.minTimes >= 1 && c.maxTimes >= c.minTimes
  )
})

const updateExpertConfigs = () => {
  const available = availableExperts.value
  const savedConfigs = props.plannerConfig?.expertConfigs || []
  
  expertConfigs.value = available.map(e => {
    const savedConfig = savedConfigs.find(c => c.name === e.name)
    const existing = expertConfigs.value.find(c => c.name === e.name)
    
    return {
      name: e.name,
      days: e.days,
      standard: e.standard,
      minPeople: savedConfig?.minPeople ?? existing?.minPeople ?? 3,
      maxPeople: savedConfig?.maxPeople ?? existing?.maxPeople ?? 9,
      minTimes: savedConfig?.minTimes ?? existing?.minTimes ?? 1,
      maxTimes: savedConfig?.maxTimes ?? existing?.maxTimes ?? 3,
      expertItem: e
    }
  })
}

const emitConfigUpdate = () => {
  const configToSave = {
    targetAmount: targetAmount.value,
    tolerancePercent: tolerancePercent.value,
    expertConfigs: expertConfigs.value.map(c => ({
      name: c.name,
      minPeople: c.minPeople,
      maxPeople: c.maxPeople,
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

const calculateSolutionCost = (solution) => {
  let total = 0
  for (let i = 0; i < solution.length; i++) {
    const { people, times } = solution[i]
    const config = expertConfigs.value[i]
    total += (config.standard / 10000) * people * config.days * times
  }
  return total
}

const generateRandomSolution = () => {
  const solution = []
  for (const config of expertConfigs.value) {
    // 生成奇数人数
    let people = randomInt(config.minPeople, config.maxPeople)
    if (people % 2 === 0) people = Math.max(config.minPeople, people - 1)
    if (people % 2 === 0) people = Math.min(config.maxPeople, people + 1)
    const times = randomInt(config.minTimes, config.maxTimes)
    solution.push({ people, times })
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
      const config = expertConfigs.value[i]
      const original = { ...currentSolution[i] }
      
      // 人数调整步长为2（保持奇数）
      const adjustments = [
        { field: 'people', delta: 2 },
        { field: 'people', delta: -2 },
        { field: 'times', delta: 1 },
        { field: 'times', delta: -1 }
      ]
      
      for (const adj of adjustments) {
        const newValue = original[adj.field] + adj.delta
        
        if (adj.field === 'people' && (newValue < config.minPeople || newValue > config.maxPeople)) continue
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
  return results.map(r => {
    const experts = r.solution.map((s, i) => {
      const config = expertConfigs.value[i]
      const amount = (config.standard / 10000) * s.people * config.days * s.times
      return {
        name: config.name,
        people: s.people,
        days: config.days,
        times: s.times,
        amount: amount,
        expertItem: config.expertItem
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

.expert-config-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.expert-config-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 6px 8px;
  background: #fff3e0;
  border-radius: 4px;
  flex-wrap: wrap;
}

.expert-name {
  font-weight: 500;
  color: #333;
  min-width: 100px;
  font-size: 12px;
}

.expert-cost {
  font-size: 11px;
  color: #666;
  min-width: 90px;
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

.no-data-hint {
  padding: 15px;
  background: #fff3e0;
  border: 1px solid #ffcc80;
  border-radius: 4px;
  color: #e65100;
  font-size: 13px;
  text-align: center;
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
