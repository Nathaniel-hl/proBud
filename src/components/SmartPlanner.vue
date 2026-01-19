<template>
  <div v-if="visible" class="modal-overlay" @click.self="close">
    <div class="modal-content">
      <div class="modal-header">
        <h4>差旅费智能规划</h4>
        <button class="modal-close" @click="close">×</button>
      </div>
      <div class="modal-body">
        <div v-if="cityConfigs.length === 0" class="no-cities-hint">
          请先在表格中添加城市及费用标准
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
            <div class="step-title">城市参数范围设置</div>
            <div class="city-config-table">
              <div class="city-config-header">
                <span class="col-city">城市</span>
                <span class="col-cost">单次费用</span>
                <span class="col-range">人数范围</span>
                <span class="col-range">天数范围</span>
                <span class="col-range">次数范围</span>
              </div>
              <div v-for="(item, index) in cityConfigs" :key="index" class="city-config-row">
                <span class="col-city city-name">{{ item.city }}</span>
                <span class="col-cost city-cost">{{ (item.singleCost / 10000).toFixed(2) }}万</span>
                <div class="col-range range-inline">
                  <input type="number" v-model.number="item.peopleMin" min="1" step="1" class="input-tiny">
                  <span>~</span>
                  <input type="number" v-model.number="item.peopleMax" min="1" step="1" class="input-tiny">
                </div>
                <div class="col-range range-inline">
                  <input type="number" v-model.number="item.daysMin" min="1" step="1" class="input-tiny">
                  <span>~</span>
                  <input type="number" v-model.number="item.daysMax" min="1" step="1" class="input-tiny">
                </div>
                <div class="col-range range-inline">
                  <input type="number" v-model.number="item.timesMin" min="1" step="1" class="input-tiny">
                  <span>~</span>
                  <input type="number" v-model.number="item.timesMax" min="1" step="1" class="input-tiny">
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
                  <span class="combo-total">总计：{{ combo.total.toFixed(2) }} 万元</span>
                  <span class="combo-deviation" :class="getDeviationClass(combo.deviation)">
                    {{ combo.deviation > 0 ? '+' : '' }}{{ (combo.deviation * 100).toFixed(1) }}%
                  </span>
                </div>
                <div class="combo-details">
                  <div v-for="(city, ci) in combo.cities" :key="ci" class="combo-city">
                    {{ city.city }}：{{ city.people }}人 × {{ city.days }}天 × {{ city.times }}次 = {{ city.amount.toFixed(2) }}万
                  </div>
                </div>
                <button class="btn btn-success btn-small">选择此方案</button>
              </div>
            </div>
          </div>
          <div v-if="noSolution" class="no-solution">
            未找到合适的方案，请调整目标金额或参数范围
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
  travels: {
    type: Array,
    default: () => []
  },
  formula: {
    type: Object,
    default: () => ({
      transportMultiplier: 1,
      accommodationDays: 'days-1',
      foodDays: 'days',
      localTransportDays: 'days'
    })
  },
  plannerConfig: {
    type: Object,
    default: () => ({
      targetAmount: 5,
      tolerancePercent: 15,
      cityConfigs: []
    })
  }
})

const emit = defineEmits(['update:visible', 'select', 'update:plannerConfig'])

const targetAmount = ref(props.plannerConfig?.targetAmount ?? 5)
const tolerancePercent = ref(props.plannerConfig?.tolerancePercent ?? 15)
const combinations = ref([])
const noSolution = ref(false)
const cityConfigs = ref([])
const isGenerating = ref(false)
const generationTime = ref(0)

// 算法参数
const SAMPLE_COUNT = 5000      // 随机采样次数
const TOP_CANDIDATES = 50      // 保留候选解数量
const LOCAL_OPT_ITERATIONS = 20 // 局部优化迭代次数
const MAX_RESULTS = 10         // 最大返回结果数

const getDaysValue = (days, mode) => {
  if (mode === 'days-1') {
    return Math.max(0, days - 1)
  }
  return days
}

const calculateSingleCostForDays = (item, days) => {
  const transportCost = (item.transport || 0) * props.formula.transportMultiplier
  const accCost = (item.accommodation || 0) * getDaysValue(days, props.formula.accommodationDays)
  const foodCost = (item.food || 0) * getDaysValue(days, props.formula.foodDays)
  const localCost = (item.localTransport || 0) * getDaysValue(days, props.formula.localTransportDays)
  return transportCost + accCost + foodCost + localCost
}

const calculateItemSingleCost = (item) => {
  return calculateSingleCostForDays(item, item.days || 1)
}

const availableCities = computed(() => {
  return props.travels.filter(t => t.city && t.transport > 0)
})

const canGenerate = computed(() => {
  if (targetAmount.value <= 0 || cityConfigs.value.length === 0) {
    return false
  }
  return cityConfigs.value.every(c => 
    c.peopleMin >= 1 && c.peopleMax >= c.peopleMin &&
    c.daysMin >= 1 && c.daysMax >= c.daysMin &&
    c.timesMin >= 1 && c.timesMax >= c.timesMin
  )
})

const updateCityConfigs = () => {
  const cities = availableCities.value
  const savedConfigs = props.plannerConfig?.cityConfigs || []
  
  cityConfigs.value = cities.map(t => {
    // 优先从缓存的配置中恢复，其次从当前组件状态中查找
    const savedConfig = savedConfigs.find(c => c.city === t.city)
    const existing = cityConfigs.value.find(c => c.city === t.city)
    const singleCost = calculateItemSingleCost(t)
    
    return {
      city: t.city,
      singleCost: singleCost,
      travelItem: t,
      peopleMin: savedConfig?.peopleMin ?? existing?.peopleMin ?? 1,
      peopleMax: savedConfig?.peopleMax ?? existing?.peopleMax ?? 5,
      daysMin: savedConfig?.daysMin ?? existing?.daysMin ?? t.days,
      daysMax: savedConfig?.daysMax ?? existing?.daysMax ?? t.days,
      timesMin: savedConfig?.timesMin ?? existing?.timesMin ?? 1,
      timesMax: savedConfig?.timesMax ?? existing?.timesMax ?? 5
    }
  })
}

const emitConfigUpdate = () => {
  // 将配置同步到父组件进行缓存，排除travelItem避免循环引用
  const configToSave = {
    targetAmount: targetAmount.value,
    tolerancePercent: tolerancePercent.value,
    cityConfigs: cityConfigs.value.map(c => ({
      city: c.city,
      peopleMin: c.peopleMin,
      peopleMax: c.peopleMax,
      daysMin: c.daysMin,
      daysMax: c.daysMax,
      timesMin: c.timesMin,
      timesMax: c.timesMax
    }))
  }
  emit('update:plannerConfig', configToSave)
}

watch(() => props.visible, (val) => {
  if (val) {
    // 打开时从缓存恢复配置
    if (props.plannerConfig) {
      targetAmount.value = props.plannerConfig.targetAmount ?? 5
      tolerancePercent.value = props.plannerConfig.tolerancePercent ?? 15
    }
    updateCityConfigs()
    combinations.value = []
    noSolution.value = false
  }
})

watch(() => props.travels, updateCityConfigs, { deep: true })
watch(() => props.formula, updateCityConfigs, { deep: true })

// 监听配置变化，同步到父组件进行缓存
watch([targetAmount, tolerancePercent], emitConfigUpdate)
watch(cityConfigs, emitConfigUpdate, { deep: true })

const close = () => {
  emit('update:visible', false)
}

const getDeviationClass = (deviation) => {
  const absDeviation = Math.abs(deviation)
  if (absDeviation <= 0.05) return 'deviation-good'
  if (absDeviation <= 0.10) return 'deviation-ok'
  return 'deviation-warn'
}

/**
 * 计算解的总费用
 */
const calculateSolutionCost = (solution) => {
  let total = 0
  for (let i = 0; i < solution.length; i++) {
    const { people, days, times } = solution[i]
    const config = cityConfigs.value[i]
    const costPerTrip = calculateSingleCostForDays(config.travelItem, days)
    total += costPerTrip * people * times
  }
  return total
}

/**
 * 生成随机解
 */
const generateRandomSolution = () => {
  const solution = []
  for (const config of cityConfigs.value) {
    const people = randomInt(config.peopleMin, config.peopleMax)
    const days = randomInt(config.daysMin, config.daysMax)
    const times = randomInt(config.timesMin, config.timesMax)
    solution.push({ people, days, times })
  }
  return solution
}

const randomInt = (min, max) => {
  return Math.floor(Math.random() * (max - min + 1)) + min
}

/**
 * 局部优化 - 爬山法
 */
const localOptimize = (solution, targetYuan) => {
  let currentSolution = solution.map(s => ({ ...s }))
  let currentCost = calculateSolutionCost(currentSolution)
  let currentGap = Math.abs(currentCost - targetYuan)
  
  for (let iter = 0; iter < LOCAL_OPT_ITERATIONS; iter++) {
    let improved = false
    
    // 遍历每个城市，尝试微调
    for (let i = 0; i < currentSolution.length; i++) {
      const config = cityConfigs.value[i]
      const original = { ...currentSolution[i] }
      
      // 尝试调整人数、天数、次数
      const adjustments = [
        { field: 'people', delta: 1 },
        { field: 'people', delta: -1 },
        { field: 'days', delta: 1 },
        { field: 'days', delta: -1 },
        { field: 'times', delta: 1 },
        { field: 'times', delta: -1 }
      ]
      
      for (const adj of adjustments) {
        const newValue = original[adj.field] + adj.delta
        
        // 检查约束
        if (adj.field === 'people' && (newValue < config.peopleMin || newValue > config.peopleMax)) continue
        if (adj.field === 'days' && (newValue < config.daysMin || newValue > config.daysMax)) continue
        if (adj.field === 'times' && (newValue < config.timesMin || newValue > config.timesMax)) continue
        
        currentSolution[i][adj.field] = newValue
        const newCost = calculateSolutionCost(currentSolution)
        const newGap = Math.abs(newCost - targetYuan)
        
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

/**
 * 主算法：随机采样 + 局部优化 + 结果筛选
 */
const findBudgetCombinations = (targetYuan) => {
  const candidates = []
  
  // 1. 随机采样阶段
  for (let i = 0; i < SAMPLE_COUNT; i++) {
    const solution = generateRandomSolution()
    const cost = calculateSolutionCost(solution)
    const gap = Math.abs(cost - targetYuan)
    candidates.push({ solution, cost, gap })
  }
  
  // 2. 选择最佳候选解
  candidates.sort((a, b) => a.gap - b.gap)
  const bestCandidates = candidates.slice(0, TOP_CANDIDATES)
  
  // 3. 局部优化阶段
  const optimized = []
  const seenKeys = new Set()
  
  for (const candidate of bestCandidates) {
    const result = localOptimize(candidate.solution, targetYuan)
    
    // 生成唯一键，避免重复解
    const key = result.solution.map(s => `${s.people}-${s.days}-${s.times}`).join('|')
    if (seenKeys.has(key)) continue
    seenKeys.add(key)
    
    optimized.push(result)
  }
  
  // 4. 结果筛选与排序
  optimized.sort((a, b) => a.gap - b.gap)
  
  // 设置容差阈值
  const tolerance = targetYuan * (tolerancePercent.value / 100)
  const filtered = optimized.filter(r => r.gap <= tolerance)
  
  // 多样化处理：如果结果太少，放宽条件
  let results = filtered
  if (results.length < 3 && optimized.length > 0) {
    const bestGap = optimized[0].gap
    results = optimized.filter(r => r.gap <= bestGap * 1.5)
  }
  
  return results.slice(0, MAX_RESULTS)
}

/**
 * 将解转换为展示格式
 */
const formatResults = (results, targetYuan) => {
  return results.map(r => {
    const cities = r.solution.map((s, i) => {
      const config = cityConfigs.value[i]
      const costPerTrip = calculateSingleCostForDays(config.travelItem, s.days)
      const amount = costPerTrip * s.people * s.times
      return {
        city: config.city,
        people: s.people,
        days: s.days,
        times: s.times,
        amount: amount / 10000,
        travelItem: config.travelItem
      }
    })
    
    return {
      cities,
      total: r.cost / 10000,
      deviation: (r.cost - targetYuan) / targetYuan
    }
  })
}

const generateCombinations = () => {
  combinations.value = []
  noSolution.value = false
  isGenerating.value = true
  generationTime.value = 0
  
  const startTime = performance.now()
  
  // 使用 setTimeout 让 UI 有机会更新
  setTimeout(() => {
    try {
      const targetYuan = targetAmount.value * 10000
      const results = findBudgetCombinations(targetYuan)
      
      if (results.length === 0) {
        noSolution.value = true
      } else {
        combinations.value = formatResults(results, targetYuan)
      }
      
      generationTime.value = Math.round(performance.now() - startTime)
    } finally {
      isGenerating.value = false
    }
  }, 10)
}

const selectCombination = (combo) => {
  if (!confirm('将更新表格中对应城市的人数、天数和次数，是否继续？')) {
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
  width: 850px;
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

.planner-step {
  margin-bottom: 12px;
}

.step-title {
  font-size: 13px;
  font-weight: 500;
  color: #444;
  margin-bottom: 6px;
}

.city-config-table {
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  overflow: hidden;
}

.city-config-header {
  display: flex;
  background: #f5f5f5;
  padding: 8px 10px;
  font-size: 12px;
  font-weight: 500;
  color: #555;
  border-bottom: 1px solid #e0e0e0;
}

.city-config-row {
  display: flex;
  padding: 8px 10px;
  align-items: center;
  border-bottom: 1px solid #f0f0f0;
}

.city-config-row:last-child {
  border-bottom: none;
}

.city-config-row:hover {
  background: #fafafa;
}

.col-city {
  width: 80px;
  flex-shrink: 0;
}

.col-cost {
  width: 80px;
  flex-shrink: 0;
}

.col-range {
  flex: 1;
  min-width: 100px;
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

.range-inline {
  display: flex;
  align-items: center;
  gap: 4px;
}

.range-inline span {
  font-size: 11px;
  color: #666;
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

.no-cities-hint {
  padding: 15px;
  background: #fff3e0;
  border: 1px solid #ffcc80;
  border-radius: 4px;
  color: #e65100;
  font-size: 13px;
  text-align: center;
}

.city-name {
  min-width: 60px;
  font-weight: 500;
  color: #333;
}

.city-cost {
  font-size: 11px;
  color: #888;
  min-width: 90px;
}

.combinations-result {
  margin-top: 15px;
  padding-top: 12px;
  border-top: 1px solid #a5d6a7;
}

.combination-list {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.combination-card {
  background: white;
  border: 1px solid #ddd;
  border-radius: 6px;
  padding: 10px;
  min-width: 220px;
  max-width: 280px;
  cursor: pointer;
  transition: all 0.2s;
}

.combination-card:hover {
  border-color: #4caf50;
  box-shadow: 0 2px 8px rgba(76, 175, 80, 0.2);
}

.combo-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 6px;
  margin-bottom: 8px;
  padding-bottom: 6px;
  border-bottom: 1px solid #eee;
}

.combo-title {
  font-weight: 600;
  color: #333;
  font-size: 13px;
}

.combo-total {
  font-size: 12px;
  color: #2e7d32;
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
  margin-bottom: 8px;
}

.combo-city {
  font-size: 12px;
  color: #555;
  margin-bottom: 3px;
}

.no-solution {
  padding: 10px;
  background: #fff3e0;
  border: 1px solid #ffcc80;
  border-radius: 4px;
  color: #e65100;
  font-size: 13px;
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
