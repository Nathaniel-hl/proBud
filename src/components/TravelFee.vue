<template>
  <div class="section">
    <h3 class="section-title">差旅费</h3>
    
    <!-- 智能规划按钮 -->
    <div class="planner-trigger">
      <button class="btn btn-primary btn-small" @click="openPlanner">
        🧮 智能规划
      </button>
    </div>

    <!-- 智能规划组件 -->
    <SmartPlanner
      v-model:visible="showPlanner"
      v-model:plannerConfig="plannerConfig"
      :travels="travels"
      :formula="formula"
      @select="handlePlannerSelect"
    />

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
    <div class="row-actions">
      <button class="btn btn-primary btn-small" @click="addRow">+ 新增行</button>
      <div class="city-preset">
        <label>快速添加：</label>
        <select v-model="selectedCity" @change="applyCity">
          <option value="">-- 选择城市 --</option>
          <option v-for="city in citiesData" :key="city.city" :value="city.city">
            {{ city.city }} (车票{{ city.transport }}元, 住宿{{ city.accommodation }}元/天)
          </option>
        </select>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, watch } from 'vue'
import citiesData from '../data/travelCities.json'
import SmartPlanner from './SmartPlanner.vue'

const props = defineProps({
  modelValue: {
    type: Object,
    default: () => ({
      travels: [],
      formula: null,
      plannerConfig: null
    })
  }
})

const emit = defineEmits(['update:modelValue'])

const selectedCity = ref('')
const showFormulaEditor = ref(false)
const showPlanner = ref(false)

const plannerConfig = ref(props.modelValue.plannerConfig || {
  targetAmount: 5,
  tolerancePercent: 15,
  cityConfigs: []
})

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

const openPlanner = () => {
  showPlanner.value = true
}

const handlePlannerSelect = (combo) => {
  for (const cityPlan of combo.cities) {
    const travelItem = travels.value.find(t => t.city === cityPlan.city)
    if (travelItem) {
      travelItem.people = cityPlan.people
      travelItem.days = cityPlan.days
      travelItem.times = cityPlan.times
    }
  }
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

watch([travels, formula, plannerConfig], () => {
  emit('update:modelValue', {
    travels: travels.value,
    formula: { ...formula },
    totalWan: total.value,
    plannerConfig: plannerConfig.value
  })
}, { deep: true, immediate: true })

watch(() => props.modelValue, (newVal) => {
  if (newVal.travels) travels.value = newVal.travels
  if (newVal.formula) Object.assign(formula, newVal.formula)
  if (newVal.plannerConfig) plannerConfig.value = newVal.plannerConfig
}, { deep: true })
</script>

<style scoped>
.planner-trigger {
  margin-bottom: 12px;
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

.row-actions {
  display: flex;
  align-items: center;
  gap: 15px;
  margin-top: 10px;
}

.city-preset {
  display: flex;
  align-items: center;
  gap: 10px;
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
