<template>
  <div class="section">
    <h3 class="section-title">劳务费</h3>
    
    <!-- 人员劳务费 -->
    <div class="subsection">
      <h4 class="subsection-title">3.1 人员劳务费</h4>
      
      <!-- 智能规划按钮 -->
      <div class="planner-trigger">
        <button class="btn btn-primary btn-small" @click="showPlanner = true">
          🧮 智能规划
        </button>
      </div>

      <!-- 智能规划组件 -->
      <LaborSmartPlanner
        v-model:visible="showPlanner"
        v-model:plannerConfig="plannerConfig"
        :personnels="personnels"
        @select="handlePlannerSelect"
      />

      <div class="standard-info">
        <span>参考标准：</span>
        <span class="standard-item">硕士研究生 <strong>0.25</strong>万/人月</span>
        <span class="standard-item">博士研究生 <strong>0.30</strong>万/人月</span>
        <span class="standard-item">科研辅助人员 <strong>1.20</strong>万/人月</span>
        <span class="standard-item">技术研究人员 <strong>2.80</strong>万/人月</span>
      </div>
      
      <div class="unit-selector">
        <label>单位：</label>
        <select v-model="personnelUnit">
          <option value="yuan">元</option>
          <option value="wan">万元</option>
        </select>
      </div>
      <div class="table-container">
        <table>
          <thead>
            <tr>
              <th style="width: 40px">序号</th>
              <th style="width: 120px">人员类型</th>
              <th style="width: 50px">人数</th>
              <th style="width: 50px">月数</th>
              <th style="width: 80px">月均费用</th>
              <th style="width: 80px">金额</th>
              <th style="width: 120px">说明</th>
              <th style="width: 50px">操作</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(item, index) in personnels" :key="index">
              <td>{{ index + 1 }}</td>
              <td>
                <select v-model="item.type" @change="onPersonnelTypeChange(item)">
                  <option value="">请选择</option>
                  <option v-for="pt in personnelTypes" :key="pt.type" :value="pt.type">{{ pt.type }}</option>
                  <option value="自定义">自定义</option>
                </select>
                <input v-if="item.type === '自定义'" type="text" v-model="item.customType" placeholder="输入类型" class="custom-type-input">
              </td>
              <td><input type="number" v-model.number="item.count" min="1" step="1"></td>
              <td><input type="number" v-model.number="item.months" min="1" step="1"></td>
              <td><input type="number" v-model.number="item.monthlyCost" min="0" step="0.01"></td>
              <td class="amount-display">{{ formatNumber(item.count * item.months * item.monthlyCost) }}</td>
              <td><input type="text" v-model="item.description" placeholder="说明"></td>
              <td><button class="delete-btn" @click="removePersonnelRow(index)">删</button></td>
            </tr>
            <tr class="total-row">
              <td colspan="5" style="text-align: right">合计：</td>
              <td class="amount-display">{{ formatNumber(personnelTotal) }}</td>
              <td colspan="2"></td>
            </tr>
          </tbody>
        </table>
      </div>
      <button class="btn btn-primary btn-small add-row-btn" @click="addPersonnelRow">+ 新增行</button>
    </div>

    <!-- 专家咨询费 -->
    <div class="subsection" style="margin-top: 30px">
      <h4 class="subsection-title">3.2 专家咨询费</h4>
      
      <!-- 智能规划按钮 -->
      <div class="planner-trigger">
        <button class="btn btn-primary btn-small" @click="showExpertPlanner = true">
          🧮 智能规划
        </button>
      </div>

      <!-- 智能规划组件 -->
      <ExpertSmartPlanner
        v-model:visible="showExpertPlanner"
        v-model:plannerConfig="expertPlannerConfig"
        :experts="experts"
        @select="handleExpertPlannerSelect"
      />

      <div class="standard-info">
        <span>参考标准：</span>
        <span class="standard-item">人均标准 <strong>2800</strong>元/人/天</span>
        <span class="standard-item">会期一般 <strong>1</strong>天</span>
      </div>
      
      <div class="table-container">
        <table>
          <thead>
            <tr>
              <th style="width: 40px">序号</th>
              <th style="width: 120px">会议内容</th>
              <th style="width: 50px">人数</th>
              <th style="width: 60px">会期(天)</th>
              <th style="width: 50px">次数</th>
              <th style="width: 90px">人均标准(元)</th>
              <th style="width: 80px">金额(万)</th>
              <th style="width: 50px">操作</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(item, index) in experts" :key="index">
              <td>{{ index + 1 }}</td>
              <td><input type="text" v-model="item.name" placeholder="会议内容"></td>
              <td><input type="number" v-model.number="item.people" min="1" step="2"></td>
              <td><input type="number" v-model.number="item.days" min="1" step="1"></td>
              <td><input type="number" v-model.number="item.times" min="1" step="1"></td>
              <td><input type="number" v-model.number="item.standard" min="0" step="100"></td>
              <td class="amount-display">{{ formatNumber(calcExpertAmount(item)) }}</td>
              <td><button class="delete-btn" @click="removeExpertRow(index)">删</button></td>
            </tr>
            <tr class="total-row">
              <td colspan="6" style="text-align: right">合计：</td>
              <td class="amount-display">{{ formatNumber(expertTotal) }}</td>
              <td></td>
            </tr>
          </tbody>
        </table>
      </div>
      <button class="btn btn-primary btn-small add-row-btn" @click="addExpertRow">+ 新增行</button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import LaborSmartPlanner from './LaborSmartPlanner.vue'
import ExpertSmartPlanner from './ExpertSmartPlanner.vue'

const personnelTypes = [
  { type: '硕士研究生', monthlyCost: 0.25 },
  { type: '博士研究生', monthlyCost: 0.30 },
  { type: '科研辅助人员', monthlyCost: 1.20 },
  { type: '技术研究人员', monthlyCost: 2.80 }
]

const props = defineProps({
  modelValue: {
    type: Object,
    default: () => ({
      personnels: [],
      experts: [],
      personnelUnit: 'wan',
      expertUnit: 'wan',
      plannerConfig: null,
      expertPlannerConfig: null
    })
  }
})

const emit = defineEmits(['update:modelValue'])

const personnelUnit = ref(props.modelValue.personnelUnit || 'wan')
const expertUnit = ref(props.modelValue.expertUnit || 'wan')

const personnels = ref(props.modelValue.personnels?.length > 0 
  ? props.modelValue.personnels 
  : [{ type: '', count: 1, months: 1, monthlyCost: 0, description: '', customType: '' }])

const experts = ref(props.modelValue.experts?.length > 0 
  ? props.modelValue.experts 
  : [{ name: '', people: 5, days: 1, times: 1, standard: 2800 }])

const showExpertPlanner = ref(false)
const showPlanner = ref(false)

const plannerConfig = ref(props.modelValue.plannerConfig || {
  targetAmount: 10,
  tolerancePercent: 15,
  personnelConfigs: []
})

const expertPlannerConfig = ref(props.modelValue.expertPlannerConfig || {
  targetAmount: 2,
  tolerancePercent: 15,
  expertConfigs: []
})

const calcExpertAmount = (item) => {
  return (item.people || 0) * (item.days || 0) * (item.times || 0) * (item.standard || 0) / 10000
}

const handlePlannerSelect = (combo) => {
  for (const plan of combo.personnels) {
    const personnel = personnels.value.find(p => {
      const typeName = p.type === '自定义' ? p.customType : p.type
      return typeName === plan.type
    })
    if (personnel) {
      personnel.count = plan.count
      personnel.months = plan.months
    }
  }
}

const handleExpertPlannerSelect = (combo) => {
  for (const plan of combo.experts) {
    const expert = experts.value.find(e => e.name === plan.name)
    if (expert) {
      expert.people = plan.people
      expert.times = plan.times
    }
  }
}

const formatNumber = (num) => {
  if (!num || isNaN(num)) return '0'
  return Number(num).toFixed(2)
}

const onPersonnelTypeChange = (item) => {
  const found = personnelTypes.find(pt => pt.type === item.type)
  if (found) {
    item.monthlyCost = found.monthlyCost
  } else if (item.type === '自定义') {
    item.monthlyCost = 0
  }
}

const personnelTotal = computed(() => {
  return personnels.value.reduce((sum, item) => {
    return sum + (item.count || 0) * (item.months || 0) * (item.monthlyCost || 0)
  }, 0)
})

const expertTotal = computed(() => {
  return experts.value.reduce((sum, item) => {
    return sum + calcExpertAmount(item)
  }, 0)
})

const personnelTotalWan = computed(() => {
  return personnelUnit.value === 'yuan' ? personnelTotal.value / 10000 : personnelTotal.value
})

const expertTotalWan = computed(() => {
  return expertTotal.value
})

const addPersonnelRow = () => {
  personnels.value.push({ type: '', count: 1, months: 1, monthlyCost: 0, description: '', customType: '' })
}

const removePersonnelRow = (index) => {
  if (personnels.value.length > 1) {
    personnels.value.splice(index, 1)
  }
}

const addExpertRow = () => {
  experts.value.push({ name: '', people: 5, days: 1, times: 1, standard: 2800 })
}

const removeExpertRow = (index) => {
  if (experts.value.length > 1) {
    experts.value.splice(index, 1)
  }
}

watch([personnels, experts, personnelUnit, expertUnit, plannerConfig, expertPlannerConfig], () => {
  emit('update:modelValue', {
    personnels: personnels.value,
    experts: experts.value,
    personnelUnit: personnelUnit.value,
    expertUnit: expertUnit.value,
    personnelTotalWan: personnelTotalWan.value,
    expertTotalWan: expertTotalWan.value,
    plannerConfig: plannerConfig.value,
    expertPlannerConfig: expertPlannerConfig.value
  })
}, { deep: true, immediate: true })

watch(() => props.modelValue, (newVal) => {
  if (newVal.personnels) personnels.value = newVal.personnels
  if (newVal.experts) experts.value = newVal.experts
  if (newVal.personnelUnit) personnelUnit.value = newVal.personnelUnit
  if (newVal.expertUnit) expertUnit.value = newVal.expertUnit
  if (newVal.plannerConfig) plannerConfig.value = newVal.plannerConfig
  if (newVal.expertPlannerConfig) expertPlannerConfig.value = newVal.expertPlannerConfig
}, { deep: true })
</script>

<style scoped>
.subsection {
  margin-bottom: 20px;
}

.subsection-title {
  font-size: 16px;
  color: #555;
  margin-bottom: 15px;
  padding-left: 10px;
  border-left: 3px solid #667eea;
}

.standard-info {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 8px;
  margin-bottom: 10px;
  font-size: 12px;
  color: #555;
  padding: 8px 12px;
  background: #f5f5f5;
  border-radius: 4px;
}

.standard-info strong {
  color: #1976d2;
}

.standard-item {
  padding: 2px 8px;
  background: white;
  border-radius: 3px;
  border: 1px solid #e0e0e0;
}

.custom-type-input {
  margin-top: 4px;
  width: 100% !important;
}

.planner-trigger {
  margin-bottom: 12px;
}
</style>
