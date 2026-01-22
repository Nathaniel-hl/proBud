<template>
  <div v-if="visible" class="modal-overlay" @click.self="close">
    <div class="modal-content">
      <div class="modal-header">
        <h4>差旅费智能规划</h4>
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
            <span class="step-title">城市参数设置</span>
            <button class="btn btn-small btn-add" @click="addTempRow">+ 添加临时行</button>
          </div>
          <div class="city-config-table">
            <div class="city-config-header">
              <span class="col-enable">参与</span>
              <span class="col-city">城市</span>
              <span class="col-cost">单次费用</span>
              <span class="col-param">人数</span>
              <span class="col-param">天数</span>
              <span class="col-param">次数</span>
              <span class="col-action">操作</span>
            </div>
            <div v-for="(item, index) in cityConfigs" :key="item.id || index" class="city-config-row" :class="{ 'row-disabled': !item.enabled, 'row-temp': item.isTemp }">
              <div class="col-enable">
                <input type="checkbox" v-model="item.enabled" title="是否参与规划">
              </div>
              <div class="col-city">
                <span v-if="!item.isTemp" class="city-name">{{ item.city }}</span>
                <input v-else type="text" v-model="item.city" placeholder="城市名" class="input-city">
              </div>
              <div class="col-cost">
                <span v-if="!item.isTemp" class="city-cost">{{ (item.singleCost / 10000).toFixed(2) }}万</span>
                <input v-else type="number" v-model.number="item.singleCost" min="0" step="100" class="input-cost" placeholder="元">
              </div>
              <div class="col-param">
                <div class="param-control">
                  <label class="lock-label" :class="{ locked: item.peopleLocked }">
                    <input type="checkbox" v-model="item.peopleLocked" title="锁定为固定值">
                    <span class="lock-icon">{{ item.peopleLocked ? '🔒' : '🔓' }}</span>
                  </label>
                  <template v-if="item.peopleLocked">
                    <input type="number" v-model.number="item.peopleFixed" min="0" step="1" class="input-tiny" :disabled="!item.enabled">
                  </template>
                  <template v-else>
                    <input type="number" v-model.number="item.peopleMin" min="1" step="1" class="input-tiny" :disabled="!item.enabled">
                    <span class="range-sep">~</span>
                    <input type="number" v-model.number="item.peopleMax" min="1" step="1" class="input-tiny" :disabled="!item.enabled">
                  </template>
                </div>
              </div>
              <div class="col-param">
                <div class="param-control">
                  <label class="lock-label" :class="{ locked: item.daysLocked }">
                    <input type="checkbox" v-model="item.daysLocked" title="锁定为固定值">
                    <span class="lock-icon">{{ item.daysLocked ? '🔒' : '🔓' }}</span>
                  </label>
                  <template v-if="item.daysLocked">
                    <input type="number" v-model.number="item.daysFixed" min="1" step="1" class="input-tiny" :disabled="!item.enabled">
                  </template>
                  <template v-else>
                    <input type="number" v-model.number="item.daysMin" min="1" step="1" class="input-tiny" :disabled="!item.enabled">
                    <span class="range-sep">~</span>
                    <input type="number" v-model.number="item.daysMax" min="1" step="1" class="input-tiny" :disabled="!item.enabled">
                  </template>
                </div>
              </div>
              <div class="col-param">
                <div class="param-control">
                  <label class="lock-label" :class="{ locked: item.timesLocked }">
                    <input type="checkbox" v-model="item.timesLocked" title="锁定为固定值">
                    <span class="lock-icon">{{ item.timesLocked ? '🔒' : '🔓' }}</span>
                  </label>
                  <template v-if="item.timesLocked">
                    <input type="number" v-model.number="item.timesFixed" min="0" step="1" class="input-tiny" :disabled="!item.enabled">
                  </template>
                  <template v-else>
                    <input type="number" v-model.number="item.timesMin" min="1" step="1" class="input-tiny" :disabled="!item.enabled">
                    <span class="range-sep">~</span>
                    <input type="number" v-model.number="item.timesMax" min="1" step="1" class="input-tiny" :disabled="!item.enabled">
                  </template>
                </div>
              </div>
              <div class="col-action">
                <button v-if="item.isTemp" class="btn-delete" @click="removeTempRow(index)" title="删除临时行">×</button>
              </div>
            </div>
          </div>
          <div class="config-hint">
            💡 提示：勾选"参与"列控制是否参与规划；点击🔓可锁定变量为固定值（设为0表示不出差）
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
                <div v-for="(city, ci) in combo.cities" :key="ci" class="combo-city" :class="{ 'city-zero': city.people === 0 || city.times === 0 }">
                  {{ city.city }}：{{ city.people }}人 × {{ city.days }}天 × {{ city.times }}次 = {{ city.amount.toFixed(2) }}万
                  <span v-if="city.isTemp" class="temp-badge">临时</span>
                </div>
              </div>
              <button class="btn btn-success btn-small">选择此方案</button>
            </div>
          </div>
        </div>
        <div v-if="noSolution" class="no-solution">
          未找到合适的方案，请调整目标金额或参数范围
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
let tempIdCounter = 0

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
  if (item.isTemp) {
    // 临时行直接使用 singleCost
    return item.singleCost || 0
  }
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

// 获取参与规划的配置
const enabledConfigs = computed(() => {
  return cityConfigs.value.filter(c => c.enabled)
})

const canGenerate = computed(() => {
  if (targetAmount.value <= 0) return false
  const enabled = enabledConfigs.value
  if (enabled.length === 0) return false
  
  return enabled.every(c => {
    // 检查锁定的值是否有效
    if (c.peopleLocked && c.peopleFixed < 0) return false
    if (c.daysLocked && c.daysFixed < 1) return false
    if (c.timesLocked && c.timesFixed < 0) return false
    // 检查范围是否有效
    if (!c.peopleLocked && (c.peopleMin < 1 || c.peopleMax < c.peopleMin)) return false
    if (!c.daysLocked && (c.daysMin < 1 || c.daysMax < c.daysMin)) return false
    if (!c.timesLocked && (c.timesMin < 1 || c.timesMax < c.timesMin)) return false
    return true
  })
})

const createConfigItem = (t, savedConfig, existing) => {
  const singleCost = calculateItemSingleCost(t)
  return {
    id: t.city + '_' + Date.now(),
    city: t.city,
    singleCost: singleCost,
    travelItem: t,
    isTemp: false,
    enabled: savedConfig?.enabled ?? existing?.enabled ?? true,
    // 人数
    peopleLocked: savedConfig?.peopleLocked ?? existing?.peopleLocked ?? false,
    peopleFixed: savedConfig?.peopleFixed ?? existing?.peopleFixed ?? 1,
    peopleMin: savedConfig?.peopleMin ?? existing?.peopleMin ?? 1,
    peopleMax: savedConfig?.peopleMax ?? existing?.peopleMax ?? 5,
    // 天数
    daysLocked: savedConfig?.daysLocked ?? existing?.daysLocked ?? false,
    daysFixed: savedConfig?.daysFixed ?? existing?.daysFixed ?? (t.days || 1),
    daysMin: savedConfig?.daysMin ?? existing?.daysMin ?? (t.days || 1),
    daysMax: savedConfig?.daysMax ?? existing?.daysMax ?? (t.days || 1),
    // 次数
    timesLocked: savedConfig?.timesLocked ?? existing?.timesLocked ?? false,
    timesFixed: savedConfig?.timesFixed ?? existing?.timesFixed ?? 1,
    timesMin: savedConfig?.timesMin ?? existing?.timesMin ?? 1,
    timesMax: savedConfig?.timesMax ?? existing?.timesMax ?? 5
  }
}

const updateCityConfigs = () => {
  const cities = availableCities.value
  const savedConfigs = props.plannerConfig?.cityConfigs || []
  
  // 保留临时行
  const tempRows = cityConfigs.value.filter(c => c.isTemp)
  
  const newConfigs = cities.map(t => {
    const savedConfig = savedConfigs.find(c => c.city === t.city && !c.isTemp)
    const existing = cityConfigs.value.find(c => c.city === t.city && !c.isTemp)
    return createConfigItem(t, savedConfig, existing)
  })
  
  // 恢复保存的临时行
  const savedTempRows = savedConfigs.filter(c => c.isTemp)
  for (const saved of savedTempRows) {
    if (!tempRows.find(t => t.id === saved.id)) {
      tempRows.push({
        ...saved,
        travelItem: { transport: 0, accommodation: 0, food: 0, localTransport: 0, days: 1 }
      })
    }
  }
  
  cityConfigs.value = [...newConfigs, ...tempRows]
}

const addTempRow = () => {
  tempIdCounter++
  cityConfigs.value.push({
    id: 'temp_' + tempIdCounter + '_' + Date.now(),
    city: '',
    singleCost: 3000,
    travelItem: { transport: 0, accommodation: 0, food: 0, localTransport: 0, days: 1 },
    isTemp: true,
    enabled: true,
    peopleLocked: false,
    peopleFixed: 1,
    peopleMin: 1,
    peopleMax: 5,
    daysLocked: true,
    daysFixed: 3,
    daysMin: 1,
    daysMax: 5,
    timesLocked: false,
    timesFixed: 1,
    timesMin: 1,
    timesMax: 5
  })
}

const removeTempRow = (index) => {
  cityConfigs.value.splice(index, 1)
}

const emitConfigUpdate = () => {
  const configToSave = {
    targetAmount: targetAmount.value,
    tolerancePercent: tolerancePercent.value,
    cityConfigs: cityConfigs.value.map(c => ({
      id: c.id,
      city: c.city,
      isTemp: c.isTemp,
      singleCost: c.singleCost,
      enabled: c.enabled,
      peopleLocked: c.peopleLocked,
      peopleFixed: c.peopleFixed,
      peopleMin: c.peopleMin,
      peopleMax: c.peopleMax,
      daysLocked: c.daysLocked,
      daysFixed: c.daysFixed,
      daysMin: c.daysMin,
      daysMax: c.daysMax,
      timesLocked: c.timesLocked,
      timesFixed: c.timesFixed,
      timesMin: c.timesMin,
      timesMax: c.timesMax
    }))
  }
  emit('update:plannerConfig', configToSave)
}

watch(() => props.visible, (val) => {
  if (val) {
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

// 获取配置的有效范围
const getEffectiveRange = (config, field) => {
  const lockedKey = field + 'Locked'
  const fixedKey = field + 'Fixed'
  const minKey = field + 'Min'
  const maxKey = field + 'Max'
  
  if (config[lockedKey]) {
    const fixed = config[fixedKey]
    return { min: fixed, max: fixed }
  }
  return { min: config[minKey], max: config[maxKey] }
}

/**
 * 计算解的总费用（只计算启用的配置）
 */
const calculateSolutionCost = (solution, configs) => {
  let total = 0
  for (let i = 0; i < solution.length; i++) {
    const { people, days, times } = solution[i]
    const config = configs[i]
    const costPerTrip = config.isTemp ? config.singleCost : calculateSingleCostForDays(config.travelItem, days)
    total += costPerTrip * people * times
  }
  return total
}

/**
 * 生成随机解
 */
const generateRandomSolution = (configs) => {
  const solution = []
  for (const config of configs) {
    const peopleRange = getEffectiveRange(config, 'people')
    const daysRange = getEffectiveRange(config, 'days')
    const timesRange = getEffectiveRange(config, 'times')
    
    const people = randomInt(peopleRange.min, peopleRange.max)
    const days = randomInt(daysRange.min, daysRange.max)
    const times = randomInt(timesRange.min, timesRange.max)
    solution.push({ people, days, times })
  }
  return solution
}

const randomInt = (min, max) => {
  if (min > max) return min
  return Math.floor(Math.random() * (max - min + 1)) + min
}

/**
 * 局部优化 - 爬山法
 */
const localOptimize = (solution, targetYuan, configs) => {
  let currentSolution = solution.map(s => ({ ...s }))
  let currentCost = calculateSolutionCost(currentSolution, configs)
  let currentGap = Math.abs(currentCost - targetYuan)
  
  for (let iter = 0; iter < LOCAL_OPT_ITERATIONS; iter++) {
    let improved = false
    
    for (let i = 0; i < currentSolution.length; i++) {
      const config = configs[i]
      const original = { ...currentSolution[i] }
      
      const adjustments = []
      // 只有未锁定的字段才能调整
      if (!config.peopleLocked) {
        adjustments.push({ field: 'people', delta: 1 })
        adjustments.push({ field: 'people', delta: -1 })
      }
      if (!config.daysLocked) {
        adjustments.push({ field: 'days', delta: 1 })
        adjustments.push({ field: 'days', delta: -1 })
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
  const configs = enabledConfigs.value
  if (configs.length === 0) return []
  
  const candidates = []
  
  for (let i = 0; i < SAMPLE_COUNT; i++) {
    const solution = generateRandomSolution(configs)
    const cost = calculateSolutionCost(solution, configs)
    const gap = Math.abs(cost - targetYuan)
    candidates.push({ solution, cost, gap })
  }
  
  candidates.sort((a, b) => a.gap - b.gap)
  const bestCandidates = candidates.slice(0, TOP_CANDIDATES)
  
  const optimized = []
  const seenKeys = new Set()
  
  for (const candidate of bestCandidates) {
    const result = localOptimize(candidate.solution, targetYuan, configs)
    
    const key = result.solution.map(s => `${s.people}-${s.days}-${s.times}`).join('|')
    if (seenKeys.has(key)) continue
    seenKeys.add(key)
    
    optimized.push(result)
  }
  
  optimized.sort((a, b) => a.gap - b.gap)
  
  const tolerance = targetYuan * (tolerancePercent.value / 100)
  const filtered = optimized.filter(r => r.gap <= tolerance)
  
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
  const configs = enabledConfigs.value
  
  return results.map(r => {
    const cities = r.solution.map((s, i) => {
      const config = configs[i]
      const costPerTrip = config.isTemp ? config.singleCost : calculateSingleCostForDays(config.travelItem, s.days)
      const amount = costPerTrip * s.people * s.times
      return {
        city: config.city || '(未命名)',
        people: s.people,
        days: s.days,
        times: s.times,
        amount: amount / 10000,
        travelItem: config.travelItem,
        isTemp: config.isTemp,
        configId: config.id
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
  const hasTemp = combo.cities.some(c => c.isTemp)
  const msg = hasTemp 
    ? '将更新表格中对应城市的人数、天数和次数。临时行数据需要手动添加到表格中。是否继续？'
    : '将更新表格中对应城市的人数、天数和次数，是否继续？'
  
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
  width: 900px;
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

.step-title-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.step-title {
  font-size: 13px;
  font-weight: 500;
  color: #444;
}

.btn-add {
  background: #2196f3;
  color: white;
}

.btn-add:hover {
  background: #1976d2;
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

.col-city {
  width: 80px;
  flex-shrink: 0;
}

.col-cost {
  width: 70px;
  flex-shrink: 0;
}

.col-param {
  flex: 1;
  min-width: 130px;
}

.col-action {
  width: 40px;
  flex-shrink: 0;
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

.input-city {
  width: 70px !important;
}

.input-cost {
  width: 60px !important;
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

.config-hint {
  margin-top: 8px;
  font-size: 11px;
  color: #888;
}

.city-name {
  font-weight: 500;
  color: #333;
  font-size: 12px;
}

.city-cost {
  font-size: 11px;
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

.combo-city.city-zero {
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
