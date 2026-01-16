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

      <!-- 智能规划弹窗 -->
      <div v-if="showPlanner" class="modal-overlay" @click.self="showPlanner = false">
        <div class="modal-content">
          <div class="modal-header">
            <h4>人员劳务费智能规划</h4>
            <button class="modal-close" @click="showPlanner = false">×</button>
          </div>
          <div class="modal-body">
            <div v-if="availablePersonnelsForPlanner.length === 0" class="no-data-hint">
              请先在表格中添加人员类型和月均费用
            </div>
            <template v-else>
              <div class="planner-row">
                <div class="planner-field">
                  <label>目标金额(万)</label>
                  <input type="number" v-model.number="plannerTarget" min="0" step="0.1" class="input-small">
                </div>
              </div>
              <div class="planner-step">
                <div class="step-title">各类人员参数范围</div>
                <div class="personnel-config-list">
                  <div v-for="(config, index) in plannerPersonnelConfigs" :key="index" class="personnel-config-item">
                    <span class="personnel-name">{{ config.type }}</span>
                    <span class="personnel-cost">{{ config.monthlyCost }}万/人月</span>
                    <div class="config-ranges">
                      <div class="range-group">
                        <label>人数</label>
                        <input type="number" v-model.number="config.minCount" min="0" class="input-tiny">
                        <span>~</span>
                        <input type="number" v-model.number="config.maxCount" min="0" class="input-tiny">
                      </div>
                      <div class="range-group">
                        <label>月数</label>
                        <input type="number" v-model.number="config.minMonths" min="1" class="input-tiny">
                        <span>~</span>
                        <input type="number" v-model.number="config.maxMonths" min="1" class="input-tiny">
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
                    <div v-for="(p, pi) in combo.personnels" :key="pi" class="combo-personnel">
                      {{ p.type }}：{{ p.count }}人×{{ p.months }}月 = {{ p.amount.toFixed(2) }}万
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

      <!-- 智能规划弹窗 -->
      <div v-if="showExpertPlanner" class="modal-overlay" @click.self="showExpertPlanner = false">
        <div class="modal-content">
          <div class="modal-header">
            <h4>专家咨询费智能规划</h4>
            <button class="modal-close" @click="showExpertPlanner = false">×</button>
          </div>
          <div class="modal-body">
            <div v-if="availableExpertsForPlanner.length === 0" class="no-data-hint">
              请先在表格中添加会议内容和人均标准
            </div>
            <template v-else>
              <div class="planner-row">
                <div class="planner-field">
                  <label>目标金额(万)</label>
                  <input type="number" v-model.number="expertPlannerTarget" min="0" step="0.1" class="input-small">
                </div>
              </div>
              <div class="planner-step">
                <div class="step-title">各会议参数范围（人数为奇数）</div>
                <div class="personnel-config-list">
                  <div v-for="(config, index) in expertPlannerConfigs" :key="index" class="personnel-config-item">
                    <span class="personnel-name">{{ config.name }}</span>
                    <span class="personnel-cost">{{ config.standard }}元/人/天</span>
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
                <button class="btn btn-primary btn-small" @click="generateExpertCombinations" :disabled="!canGenerateExpert">
                  生成方案
                </button>
              </div>
              
              <!-- 方案结果 -->
              <div v-if="expertCombinations.length > 0" class="combinations-result">
                <div class="step-title">选择方案</div>
                <div class="combination-list">
                  <div 
                    v-for="(combo, index) in expertCombinations" 
                    :key="index" 
                    class="combination-card"
                    @click="selectExpertCombination(combo)"
                  >
                    <div class="combo-header">
                      <span class="combo-title">方案 {{ index + 1 }}</span>
                      <span class="combo-total">{{ combo.total.toFixed(2) }} 万元</span>
                    </div>
                    <div class="combo-details">
                      <div v-for="(e, ei) in combo.experts" :key="ei" class="combo-personnel">
                        {{ e.name }}：{{ e.people }}人×{{ e.days }}天×{{ e.times }}次 = {{ e.amount.toFixed(2) }}万
                      </div>
                    </div>
                    <button class="btn btn-success btn-small">选择</button>
                  </div>
                </div>
              </div>
              <div v-if="expertNoSolution" class="no-solution">
                未找到合适方案，请调整参数范围
              </div>
            </template>
          </div>
          <div class="modal-footer">
            <button class="btn btn-small" @click="showExpertPlanner = false">关闭</button>
          </div>
        </div>
      </div>

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
      expertUnit: 'wan'
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
const expertPlannerTarget = ref(2)
const expertCombinations = ref([])
const expertNoSolution = ref(false)
const expertPlannerConfigs = ref([])

const availableExpertsForPlanner = computed(() => {
  return experts.value.filter(e => e.name && e.standard > 0 && e.days > 0)
})

const canGenerateExpert = computed(() => {
  return expertPlannerTarget.value > 0 && availableExpertsForPlanner.value.length > 0
})

const updateExpertPlannerConfigs = () => {
  const available = availableExpertsForPlanner.value
  expertPlannerConfigs.value = available.map(e => {
    const existing = expertPlannerConfigs.value.find(c => c.name === e.name)
    return {
      name: e.name,
      days: e.days,
      standard: e.standard,
      minPeople: existing?.minPeople ?? 3,
      maxPeople: existing?.maxPeople ?? 9,
      minTimes: existing?.minTimes ?? 1,
      maxTimes: existing?.maxTimes ?? 3,
      expertItem: e
    }
  })
}

watch(availableExpertsForPlanner, updateExpertPlannerConfigs, { deep: true })

const calcExpertAmount = (item) => {
  return (item.people || 0) * (item.days || 0) * (item.times || 0) * (item.standard || 0) / 10000
}

const generateExpertCombinations = () => {
  expertCombinations.value = []
  expertNoSolution.value = false
  
  const targetWan = expertPlannerTarget.value
  const configs = expertPlannerConfigs.value
  const results = []
  const seenKeys = new Set()
  
  const tryAddResult = (expertResults, totalCost) => {
    const deviation = Math.abs(totalCost - targetWan) / targetWan
    if (deviation <= 0.15) {
      const key = expertResults.map(e => `${e.name}-${e.people}-${e.times}`).join('|')
      if (!seenKeys.has(key)) {
        seenKeys.add(key)
        results.push({
          experts: [...expertResults],
          total: totalCost,
          deviation: deviation
        })
      }
    }
  }
  
  const generateForExpert = (expertIndex, currentResults, currentTotal) => {
    if (expertIndex >= configs.length) {
      if (currentResults.length > 0) {
        tryAddResult(currentResults, currentTotal)
      }
      return
    }
    
    const config = configs[expertIndex]
    const costPerPersonDay = config.standard / 10000
    const days = config.days
    
    for (let people = config.minPeople; people <= config.maxPeople; people += 2) {
      if (people % 2 === 0) continue
      for (let times = config.minTimes; times <= config.maxTimes; times++) {
        const cost = costPerPersonDay * people * days * times
        
        if (currentTotal + cost > targetWan * 1.5) continue
        
        const newResults = [...currentResults, {
          name: config.name,
          people: people,
          days: days,
          times: times,
          amount: cost,
          expertItem: config.expertItem
        }]
        
        if (results.length < 100) {
          generateForExpert(expertIndex + 1, newResults, currentTotal + cost)
        }
      }
    }
  }
  
  generateForExpert(0, [], 0)
  
  results.sort((a, b) => a.deviation - b.deviation)
  expertCombinations.value = results.slice(0, 8)
  
  if (expertCombinations.value.length === 0) {
    expertNoSolution.value = true
  }
}

const selectExpertCombination = (combo) => {
  if (!confirm('将更新表格中对应会议的人数和次数，是否继续？')) {
    return
  }
  
  for (const plan of combo.experts) {
    const expert = experts.value.find(e => e.name === plan.name)
    if (expert) {
      expert.people = plan.people
      expert.times = plan.times
    }
  }
  
  showExpertPlanner.value = false
  expertCombinations.value = []
}

const showPlanner = ref(false)
const plannerTarget = ref(10)
const combinations = ref([])
const noSolution = ref(false)

const plannerPersonnelConfigs = ref([])

const availablePersonnelsForPlanner = computed(() => {
  return personnels.value.filter(p => {
    const typeName = p.type === '自定义' ? p.customType : p.type
    return typeName && p.monthlyCost > 0
  })
})

const canGenerate = computed(() => {
  return plannerTarget.value > 0 && availablePersonnelsForPlanner.value.length > 0
})

const updatePlannerPersonnelConfigs = () => {
  const available = availablePersonnelsForPlanner.value
  plannerPersonnelConfigs.value = available.map(p => {
    const typeName = p.type === '自定义' ? p.customType : p.type
    const existing = plannerPersonnelConfigs.value.find(c => c.type === typeName)
    return {
      type: typeName,
      monthlyCost: p.monthlyCost,
      minCount: existing?.minCount ?? 1,
      maxCount: existing?.maxCount ?? 5,
      minMonths: existing?.minMonths ?? 6,
      maxMonths: existing?.maxMonths ?? 24,
      personnelItem: p
    }
  })
}

watch(availablePersonnelsForPlanner, updatePlannerPersonnelConfigs, { deep: true })

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

const generateCombinations = () => {
  combinations.value = []
  noSolution.value = false
  
  const targetWan = plannerTarget.value
  const configs = plannerPersonnelConfigs.value
  const results = []
  const seenKeys = new Set()
  
  const tryAddResult = (personnelResults, totalCost) => {
    const deviation = Math.abs(totalCost - targetWan) / targetWan
    if (deviation <= 0.15) {
      const key = personnelResults.map(p => `${p.type}-${p.count}-${p.months}`).join('|')
      if (!seenKeys.has(key)) {
        seenKeys.add(key)
        results.push({
          personnels: [...personnelResults],
          total: totalCost,
          deviation: deviation
        })
      }
    }
  }
  
  const generateForPersonnel = (personnelIndex, currentResults, currentTotal) => {
    if (personnelIndex >= configs.length) {
      if (currentResults.length > 0) {
        tryAddResult(currentResults, currentTotal)
      }
      return
    }
    
    const config = configs[personnelIndex]
    const costPerMonth = config.monthlyCost
    
    for (let count = config.minCount; count <= config.maxCount; count++) {
      if (count === 0) {
        if (results.length < 100) {
          generateForPersonnel(personnelIndex + 1, currentResults, currentTotal)
        }
        continue
      }
      
      for (let months = config.minMonths; months <= config.maxMonths; months += 3) {
        const cost = costPerMonth * count * months
        
        if (currentTotal + cost > targetWan * 1.5) continue
        
        const newResults = [...currentResults, {
          type: config.type,
          count: count,
          months: months,
          monthlyCost: costPerMonth,
          amount: cost
        }]
        
        if (results.length < 100) {
          generateForPersonnel(personnelIndex + 1, newResults, currentTotal + cost)
        }
      }
    }
  }
  
  generateForPersonnel(0, [], 0)
  
  results.sort((a, b) => a.deviation - b.deviation)
  combinations.value = results.slice(0, 8)
  
  if (combinations.value.length === 0) {
    noSolution.value = true
  }
}

const selectCombination = (combo) => {
  if (!confirm('将更新表格中对应人员的人数和月数，是否继续？')) {
    return
  }
  
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
  
  showPlanner.value = false
  combinations.value = []
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

watch([personnels, experts, personnelUnit, expertUnit], () => {
  emit('update:modelValue', {
    personnels: personnels.value,
    experts: experts.value,
    personnelUnit: personnelUnit.value,
    expertUnit: expertUnit.value,
    personnelTotalWan: personnelTotalWan.value,
    expertTotalWan: expertTotalWan.value
  })
}, { deep: true, immediate: true })

watch(() => props.modelValue, (newVal) => {
  if (newVal.personnels) personnels.value = newVal.personnels
  if (newVal.experts) experts.value = newVal.experts
  if (newVal.personnelUnit) personnelUnit.value = newVal.personnelUnit
  if (newVal.expertUnit) expertUnit.value = newVal.expertUnit
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

.personnel-config-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.personnel-config-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 6px 8px;
  background: #f1f8e9;
  border-radius: 4px;
  flex-wrap: wrap;
}

.personnel-name {
  font-weight: 500;
  color: #333;
  min-width: 100px;
  font-size: 12px;
}

.personnel-cost {
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

.combo-personnel {
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

.no-data-hint {
  padding: 15px;
  background: #fff3e0;
  border: 1px solid #ffcc80;
  border-radius: 4px;
  color: #e65100;
  font-size: 13px;
  text-align: center;
}
</style>
