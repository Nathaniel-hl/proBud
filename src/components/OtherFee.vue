<template>
  <div class="section">
    <h3 class="section-title">其他费用</h3>
    
    <!-- 审计费参考计算 -->
    <div class="calc-trigger">
      <button class="btn btn-primary btn-small" @click="showCalcModal = true">
        🧮 计算审计费参考
      </button>
    </div>

    <!-- 审计费计算弹窗 -->
    <div v-if="showCalcModal" class="modal-overlay" @click.self="showCalcModal = false">
      <div class="modal-content">
        <div class="modal-header">
          <h4>审计费参考计算</h4>
          <button class="modal-close" @click="showCalcModal = false">×</button>
        </div>
        <div class="modal-body">
          <div class="calc-rule">
            <div class="rule-title">计算规则</div>
            <div class="rule-content">
              <p><strong>审计费 = 预计总费用 × 6‰</strong></p>
              <p>审计费一般按照项目总费用的千分之六计算</p>
            </div>
          </div>
          <div class="calc-inputs">
            <div class="calc-field">
              <label>预计总费用(万元)</label>
              <input type="number" v-model.number="auditExpectedTotal" min="0" step="1" class="input-medium">
            </div>
          </div>
          <div class="calc-results">
            <div class="result-item highlight">
              <span class="result-label">参考审计费</span>
              <span class="result-value">{{ formatNumber(auditFeeResult) }} 万元</span>
              <span class="result-hint">(千分之六)</span>
            </div>
          </div>
          <div class="apply-section">
            <button class="btn btn-success btn-small" @click="applyAuditFee">添加审计费</button>
          </div>
        </div>
        <div class="modal-footer">
          <button class="btn btn-small" @click="showCalcModal = false">关闭</button>
        </div>
      </div>
    </div>

    <div class="standard-info">
      <span>提示：</span>其他费用一般包括审计费，按总费用的 <strong>6‰</strong> 计算
    </div>

    <div class="unit-selector">
      <label>单位：</label>
      <select v-model="unit">
        <option value="yuan">元</option>
        <option value="wan">万元</option>
      </select>
    </div>
    <div class="table-container">
      <table>
        <thead>
          <tr>
            <th style="width: 60px">序号</th>
            <th style="width: 200px">费用名称</th>
            <th style="width: 120px">金额</th>
            <th>说明</th>
            <th style="width: 80px">操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in others" :key="index">
            <td>{{ index + 1 }}</td>
            <td><input type="text" v-model="item.name" placeholder="费用名称"></td>
            <td><input type="number" v-model.number="item.amount" min="0" step="0.01"></td>
            <td><input type="text" v-model="item.description" placeholder="说明"></td>
            <td><button class="delete-btn" @click="removeRow(index)">删除</button></td>
          </tr>
          <tr class="total-row">
            <td colspan="2" style="text-align: right">合计：</td>
            <td class="amount-display">{{ formatNumber(total) }}</td>
            <td colspan="2"></td>
          </tr>
        </tbody>
      </table>
    </div>
    <button class="btn btn-primary btn-small add-row-btn" @click="addRow">+ 新增行</button>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  modelValue: {
    type: Object,
    default: () => ({
      others: [],
      unit: 'wan'
    })
  }
})

const emit = defineEmits(['update:modelValue'])

const unit = ref(props.modelValue.unit || 'wan')
const others = ref(props.modelValue.others?.length > 0 
  ? props.modelValue.others 
  : [{ name: '', amount: 0, description: '' }])

const showCalcModal = ref(false)
const auditExpectedTotal = ref(100)

const formatNumber = (num) => {
  if (!num || isNaN(num)) return '0'
  return Number(num).toFixed(2)
}

const auditFeeResult = computed(() => {
  return auditExpectedTotal.value * 0.006
})

const applyAuditFee = () => {
  const existingAudit = others.value.find(o => o.name === '审计费')
  const feeValue = unit.value === 'yuan' ? auditFeeResult.value * 10000 : auditFeeResult.value
  
  if (existingAudit) {
    existingAudit.amount = feeValue
    existingAudit.description = `按总费用${auditExpectedTotal.value}万元的6‰计算`
  } else {
    others.value.push({
      name: '审计费',
      amount: feeValue,
      description: `按总费用${auditExpectedTotal.value}万元的6‰计算`
    })
  }
  showCalcModal.value = false
}

const total = computed(() => {
  return others.value.reduce((sum, item) => {
    return sum + (item.amount || 0)
  }, 0)
})

const totalWan = computed(() => {
  return unit.value === 'yuan' ? total.value / 10000 : total.value
})

const addRow = () => {
  others.value.push({ name: '', amount: 0, description: '' })
}

const removeRow = (index) => {
  if (others.value.length > 1) {
    others.value.splice(index, 1)
  }
}

watch([others, unit], () => {
  emit('update:modelValue', {
    others: others.value,
    unit: unit.value,
    totalWan: totalWan.value
  })
}, { deep: true, immediate: true })

watch(() => props.modelValue, (newVal) => {
  if (newVal.others) others.value = newVal.others
  if (newVal.unit) unit.value = newVal.unit
}, { deep: true })
</script>

<style scoped>
.calc-trigger {
  margin-bottom: 12px;
}

.standard-info {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 10px;
  font-size: 13px;
  color: #555;
  padding: 8px 12px;
  background: #fff8e6;
  border: 1px solid #ffd666;
  border-radius: 4px;
}

.standard-info strong {
  color: #d48806;
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
  width: 450px;
  max-width: 90%;
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

.calc-rule {
  background: #f5f5f5;
  border-radius: 6px;
  padding: 12px 15px;
  margin-bottom: 15px;
}

.rule-title {
  font-weight: 600;
  color: #333;
  margin-bottom: 8px;
  font-size: 13px;
}

.rule-content {
  font-size: 12px;
  color: #555;
}

.rule-content p {
  margin: 0 0 6px 0;
}

.calc-inputs {
  margin-bottom: 15px;
}

.calc-field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.calc-field label {
  font-size: 12px;
  color: #666;
}

.input-medium {
  width: 150px !important;
  text-align: right;
  padding: 6px 10px;
}

.calc-results {
  background: #e8f5e9;
  border-radius: 6px;
  padding: 12px 15px;
}

.result-item {
  display: flex;
  align-items: center;
  gap: 10px;
}

.result-item.highlight {
  background: #c8e6c9;
  margin: -12px -15px;
  padding: 12px 15px;
  border-radius: 6px;
}

.result-label {
  font-size: 13px;
  color: #555;
  min-width: 90px;
}

.result-value {
  font-weight: 600;
  color: #1565c0;
  font-size: 18px;
}

.result-hint {
  font-size: 11px;
  color: #888;
}

.apply-section {
  margin-top: 15px;
  text-align: center;
}
</style>
