<template>
  <div class="section">
    <h3 class="section-title">差旅费</h3>
    
    <!-- 智能规划按钮 -->
    <div class="planner-trigger">
      <button class="btn btn-primary btn-small" @click="openPlanner">
        🧮 智能规划
      </button>
    </div>

    <!-- 智能规划弹窗 -->
    <div v-if="showPlanner" class="modal-overlay" @click.self="showPlanner = false">
      <div class="modal-content">
        <div class="modal-header">
          <h4>差旅费智能规划</h4>
          <button class="modal-close" @click="showPlanner = false">×</button>
        </div>
        <div class="modal-body">
          <div v-if="availableCitiesForPlanner.length === 0" class="no-cities-hint">
            请先在表格中添加城市及费用标准
          </div>
          <template v-else>
            <div class="planner-row">
              <div class="planner-field">
                <label>目标金额(万)</label>
                <input type="number" v-model.number="plannerTarget" min="0" step="0.1" class="input-small">
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
                <div v-for="(item, index) in plannerCityConfigs" :key="index" class="city-config-row">
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
                    <span class="combo-total">总计：{{ combo.total.toFixed(2) }} 万元</span>
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
          <button class="btn btn-small" @click="showPlanner = false">关闭</button>
        </div>
      </div>
    </div>

    <!-- 计算公式配置 -->
    <div class="formula-config">
      <div class="formula-header">
        <span class="formula-label">计算公式：</span>
        <button class="btn btn-small" @click="showFormulaEditor = !showFormulaEditor">
          {{ showFormulaEditor ? '收起' : '编辑公式' }}
        </button>
      </div>
      <div class="formula-display">
        <code>{{ formulaDescription }}</code>
      </div>
      <div v-if="showFormulaEditor" class="formula-editor">
        <div class="formula-item">
          <label>往返交通费系数：</label>
          <input type="number" v-model.number="formula.transportMultiplier" min="1" step="1">
          <span class="formula-hint">（默认2，表示往返）</span>
        </div>
        <div class="formula-item">
          <label>住宿天数计算：</label>
          <select v-model="formula.accommodationDays">
            <option value="days-1">天数 - 1</option>
            <option value="days">天数</option>
          </select>
          <span class="formula-hint">（默认天数-1，最后一天不住宿）</span>
        </div>
        <div class="formula-item">
          <label>伙食天数计算：</label>
          <select v-model="formula.foodDays">
            <option value="days">天数</option>
            <option value="days-1">天数 - 1</option>
          </select>
        </div>
        <div class="formula-item">
          <label>市内交通天数：</label>
          <select v-model="formula.localTransportDays">
            <option value="days">天数</option>
            <option value="days-1">天数 - 1</option>
          </select>
        </div>
      </div>
    </div>

    <div class="city-preset">
      <label>快速添加：</label>
      <select v-model="selectedCity" @change="applyCity">
        <option value="">-- 选择城市 --</option>
        <option v-for="city in citiesData" :key="city.city" :value="city.city">
          {{ city.city }} (车票{{ city.transport }}元, 住宿{{ city.accommodation }}元/天)
        </option>
      </select>
    </div>
    <div class="table-container">
      <table>
        <thead>
          <tr>
            <th style="width: 40px">序号</th>
            <th style="width: 70px">城市</th>
            <th style="width: 90px">出差目的</th>
            <th style="width: 70px">车票</th>
            <th style="width: 60px">住宿</th>
            <th style="width: 60px">伙食</th>
            <th style="width: 60px">交通</th>
            <th style="width: 45px">天</th>
            <th style="width: 45px">人</th>
            <th style="width: 45px">次</th>
            <th style="width: 80px">金额(万)</th>
            <th style="width: 50px">操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in travels" :key="index">
            <td>{{ index + 1 }}</td>
            <td><input type="text" v-model="item.city" placeholder="城市"></td>
            <td>
              <select v-model="item.purpose">
                <option value="">请选择</option>
                <option v-for="p in purposeOptions" :key="p" :value="p">{{ p }}</option>
              </select>
            </td>
            <td><input type="number" v-model.number="item.transport" min="0" step="1"></td>
            <td><input type="number" v-model.number="item.accommodation" min="0" step="1"></td>
            <td><input type="number" v-model.number="item.food" min="0" step="1"></td>
            <td><input type="number" v-model.number="item.localTransport" min="0" step="1"></td>
            <td><input type="number" v-model.number="item.days" min="1" step="1"></td>
            <td><input type="number" v-model.number="item.people" min="1" step="1"></td>
            <td><input type="number" v-model.number="item.times" min="1" step="1"></td>
            <td class="amount-display">{{ formatNumber(calculateRowAmount(item)) }}</td>
            <td><button class="delete-btn" @click="removeRow(index)">删</button></td>
          </tr>
          <tr class="total-row">
            <td colspan="10" style="text-align: right">合计：</td>
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
import { ref, reactive, computed, watch } from 'vue'
import citiesData from '../data/travelCities.json'

const props = defineProps({
  modelValue: {
    type: Object,
    default: () => ({
      travels: [],
      formula: null
    })
  }
})

const emit = defineEmits(['update:modelValue'])

const selectedCity = ref('')
const showFormulaEditor = ref(false)
const showPlanner = ref(false)

const plannerTarget = ref(5)
const combinations = ref([])
const noSolution = ref(false)
const plannerCityConfigs = ref([])

const purposeOptions = [
  '学术会议',
  '项目调研',
  '合作交流',
  '实验测试',
  '样品采集',
  '设备调试',
  '技术培训',
  '成果鉴定',
  '项目验收',
  '其他'
]

const formula = reactive(props.modelValue.formula || {
  transportMultiplier: 1,
  accommodationDays: 'days-1',
  foodDays: 'days',
  localTransportDays: 'days'
})

const formulaDescription = computed(() => {
  const accDays = formula.accommodationDays === 'days-1' ? '(天数-1)' : '天数'
  const foodDays = formula.foodDays === 'days-1' ? '(天数-1)' : '天数'
  const transDays = formula.localTransportDays === 'days-1' ? '(天数-1)' : '天数'
  return `[往返交通费×${formula.transportMultiplier} + 住宿×${accDays} + 伙食×${foodDays} + 市内交通×${transDays}] × 人数 × 次数`
})

const travels = ref(props.modelValue.travels?.length > 0 
  ? props.modelValue.travels 
  : [{ city: '', purpose: '', transport: 0, accommodation: 0, food: 0, localTransport: 80, days: 1, people: 1, times: 1 }])

const formatNumber = (num) => {
  if (!num || isNaN(num)) return '0.00'
  return Number(num).toFixed(2)
}

const getDaysValue = (days, mode) => {
  if (mode === 'days-1') {
    return Math.max(0, days - 1)
  }
  return days
}

const calculateItemSingleCost = (item) => {
  const days = item.days || 1
  const transportCost = (item.transport || 0) * formula.transportMultiplier
  const accCost = (item.accommodation || 0) * getDaysValue(days, formula.accommodationDays)
  const foodCost = (item.food || 0) * getDaysValue(days, formula.foodDays)
  const localCost = (item.localTransport || 0) * getDaysValue(days, formula.localTransportDays)
  return transportCost + accCost + foodCost + localCost
}

const availableCitiesForPlanner = computed(() => {
  return travels.value.filter(t => t.city && t.transport > 0)
})

const canGenerate = computed(() => {
  if (plannerTarget.value <= 0 || availableCitiesForPlanner.value.length === 0) {
    return false
  }
  return plannerCityConfigs.value.every(c => 
    c.peopleMin >= 1 && c.peopleMax >= c.peopleMin &&
    c.daysMin >= 1 && c.daysMax >= c.daysMin &&
    c.timesMin >= 1 && c.timesMax >= c.timesMin
  )
})

const calculateSingleCostForDays = (item, days) => {
  const transportCost = (item.transport || 0) * formula.transportMultiplier
  const accCost = (item.accommodation || 0) * getDaysValue(days, formula.accommodationDays)
  const foodCost = (item.food || 0) * getDaysValue(days, formula.foodDays)
  const localCost = (item.localTransport || 0) * getDaysValue(days, formula.localTransportDays)
  return transportCost + accCost + foodCost + localCost
}

const updatePlannerCityConfigs = () => {
  const cities = availableCitiesForPlanner.value
  plannerCityConfigs.value = cities.map(t => {
    const existing = plannerCityConfigs.value.find(c => c.city === t.city)
    const singleCost = calculateItemSingleCost(t)
    return {
      city: t.city,
      singleCost: singleCost,
      travelItem: t,
      peopleMin: existing?.peopleMin ?? 1,
      peopleMax: existing?.peopleMax ?? 5,
      daysMin: existing?.daysMin ?? t.days,
      daysMax: existing?.daysMax ?? t.days,
      timesMin: existing?.timesMin ?? 1,
      timesMax: existing?.timesMax ?? 5
    }
  })
}

watch(availableCitiesForPlanner, updatePlannerCityConfigs, { deep: true })
watch(formula, updatePlannerCityConfigs, { deep: true })

const openPlanner = () => {
  updatePlannerCityConfigs()
  showPlanner.value = true
}

const calculateRowAmount = (item) => {
  const days = item.days || 1
  const transportCost = (item.transport || 0) * formula.transportMultiplier
  const accCost = (item.accommodation || 0) * getDaysValue(days, formula.accommodationDays)
  const foodCost = (item.food || 0) * getDaysValue(days, formula.foodDays)
  const localCost = (item.localTransport || 0) * getDaysValue(days, formula.localTransportDays)
  const tripCost = transportCost + accCost + foodCost + localCost
  const totalCost = tripCost * (item.people || 1) * (item.times || 1)
  return totalCost / 10000
}

const total = computed(() => {
  return travels.value.reduce((sum, item) => {
    return sum + calculateRowAmount(item)
  }, 0)
})

const applyCity = () => {
  if (!selectedCity.value) return
  const cityInfo = citiesData.find(c => c.city === selectedCity.value)
  if (cityInfo) {
    travels.value.push({
      city: cityInfo.city,
      purpose: '',
      transport: cityInfo.transport,
      accommodation: cityInfo.accommodation,
      food: cityInfo.food,
      localTransport: 80,
      days: 1,
      people: 1,
      times: 1
    })
  }
  selectedCity.value = ''
}

const addRow = () => {
  travels.value.push({ city: '', purpose: '', transport: 0, accommodation: 0, food: 0, localTransport: 80, days: 1, people: 1, times: 1 })
}

const removeRow = (index) => {
  if (travels.value.length > 1) {
    travels.value.splice(index, 1)
  }
}

const generateCombinations = () => {
  combinations.value = []
  noSolution.value = false
  
  const targetYuan = plannerTarget.value * 10000
  const configs = plannerCityConfigs.value
  const results = []
  const seenKeys = new Set()
  
  const tryAddResult = (cityResults, totalCost) => {
    const deviation = Math.abs(totalCost - targetYuan) / targetYuan
    if (deviation <= 0.15) {
      const key = cityResults.map(c => `${c.city}-${c.people}-${c.days}-${c.times}`).join('|')
      if (!seenKeys.has(key)) {
        seenKeys.add(key)
        results.push({
          cities: [...cityResults],
          total: totalCost / 10000,
          deviation: deviation
        })
      }
    }
  }
  
  const generateForCity = (cityIndex, currentResults, currentTotal, remainingTarget) => {
    if (results.length >= 100) return
    
    if (cityIndex >= configs.length) {
      tryAddResult(currentResults, currentTotal)
      return
    }
    
    const config = configs[cityIndex]
    const travelItem = config.travelItem
    
    for (let d = config.daysMin; d <= config.daysMax; d++) {
      const costPerPersonTrip = calculateSingleCostForDays(travelItem, d)
      if (costPerPersonTrip <= 0) continue
      
      for (let p = config.peopleMin; p <= config.peopleMax; p++) {
        for (let t = config.timesMin; t <= config.timesMax; t++) {
          const cost = costPerPersonTrip * p * t
          
          if (cost > remainingTarget * 2 && cityIndex < configs.length - 1) continue
          
          const newResults = [...currentResults, {
            city: config.city,
            people: p,
            days: d,
            times: t,
            amount: cost / 10000,
            travelItem: travelItem
          }]
          
          if (cityIndex === configs.length - 1) {
            tryAddResult(newResults, currentTotal + cost)
          } else {
            generateForCity(cityIndex + 1, newResults, currentTotal + cost, remainingTarget - cost)
          }
        }
      }
    }
  }
  
  generateForCity(0, [], 0, targetYuan)
  
  results.sort((a, b) => a.deviation - b.deviation)
  combinations.value = results.slice(0, 8)
  
  if (combinations.value.length === 0) {
    noSolution.value = true
  }
}

const selectCombination = (combo) => {
  if (!confirm('将更新表格中对应城市的人数、天数和次数，是否继续？')) {
    return
  }
  
  for (const cityPlan of combo.cities) {
    const travelItem = travels.value.find(t => t.city === cityPlan.city)
    if (travelItem) {
      travelItem.people = cityPlan.people
      travelItem.days = cityPlan.days
      travelItem.times = cityPlan.times
    }
  }
  
  showPlanner.value = false
  combinations.value = []
}

watch([travels, formula], () => {
  emit('update:modelValue', {
    travels: travels.value,
    formula: { ...formula },
    totalWan: total.value
  })
}, { deep: true, immediate: true })

watch(() => props.modelValue, (newVal) => {
  if (newVal.travels) travels.value = newVal.travels
  if (newVal.formula) Object.assign(formula, newVal.formula)
}, { deep: true })
</script>

<style scoped>
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
  width: 800px;
  max-width: 95%;
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
  width: 60px !important;
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

.ratio-warning-text {
  font-size: 11px;
  color: #e65100;
}

.city-ratio-list {
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.city-ratio-item {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
}

.city-ratio-item select {
  width: 70px;
}

.city-ratio-item input {
  width: 40px;
  text-align: right;
}

.ratio-unit {
  font-size: 12px;
  color: #666;
}

.delete-btn-small {
  background: #ef5350;
  color: white;
  border: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  cursor: pointer;
  font-size: 14px;
  line-height: 1;
}

.ratio-total {
  font-size: 12px;
  color: #2e7d32;
  margin-top: 6px;
  padding: 4px 8px;
  background: #c8e6c9;
  border-radius: 4px;
  display: inline-block;
}

.ratio-total.ratio-error {
  background: #ffcdd2;
  color: #c62828;
}

.ratio-total.ratio-warning {
  background: #fff3e0;
  color: #e65100;
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

.range-inputs {
  display: flex;
  align-items: center;
  gap: 8px;
}

.range-inputs input {
  width: 70px;
}

.range-hint {
  font-size: 12px;
  color: #666;
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

.ratio-hint {
  font-size: 11px;
  color: #888;
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
  min-width: 200px;
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

.formula-config {
  background: #f0f4f8;
  border: 1px solid #d0d8e0;
  border-radius: 6px;
  padding: 10px 12px;
  margin-bottom: 12px;
}

.formula-header {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 6px;
}

.formula-label {
  font-weight: 500;
  color: #444;
  font-size: 13px;
}

.formula-display {
  background: #fff;
  padding: 6px 10px;
  border-radius: 4px;
  border: 1px solid #e0e0e0;
}

.formula-display code {
  font-family: 'Consolas', 'Monaco', monospace;
  font-size: 12px;
  color: #2d3748;
}

.formula-editor {
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px dashed #ccc;
}

.formula-item {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.formula-item label {
  min-width: 120px;
  font-size: 13px;
  color: #555;
}

.formula-item input,
.formula-item select {
  width: auto;
  min-width: 80px;
}

.formula-hint {
  font-size: 11px;
  color: #888;
}

.city-preset {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
}

.city-preset label {
  font-weight: 500;
  color: #555;
  font-size: 13px;
}

.city-preset select {
  min-width: 180px;
}

.city-input-wrapper {
  position: relative;
}

.city-input-wrapper input {
  width: 100%;
}

input[type="number"] {
  width: 100%;
  text-align: right;
}
</style>
