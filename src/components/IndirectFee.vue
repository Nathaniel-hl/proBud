<template>
  <div class="section">
    <h3 class="section-title">间接费用</h3>
    
    <!-- 计算参考按钮 -->
    <div class="calc-trigger">
      <button class="btn btn-primary btn-small" @click="showCalcModal = true">
        🧮 计算间接费用上限
      </button>
    </div>

    <!-- 间接费用计算弹窗 -->
    <div v-if="showCalcModal" class="modal-overlay" @click.self="showCalcModal = false">
      <div class="modal-content">
        <div class="modal-header">
          <h4>间接费用上限计算</h4>
          <button class="modal-close" @click="showCalcModal = false">×</button>
        </div>
        <div class="modal-body">
          <div class="calc-rule">
            <div class="rule-title">计算规则</div>
            <div class="rule-content">
              <p><strong>间接费用 = (直接费用 - 设备购置费) × 间接费用比例</strong></p>
              <ul>
                <li>500万元及以下部分：不超过 <strong>30%</strong></li>
                <li>500万元至1000万元部分：不超过 <strong>25%</strong></li>
                <li>1000万元以上部分：不超过 <strong>20%</strong></li>
              </ul>
            </div>
          </div>
          <div class="calc-inputs">
            <div class="calc-field">
              <label>预计总费用(万元)</label>
              <input type="number" v-model.number="modalExpectedTotal" min="0" step="1" class="input-medium">
            </div>
            <div class="calc-field">
              <label>设备购置费(万元)</label>
              <input type="number" v-model.number="modalPurchaseEquipment" min="0" step="0.01" class="input-medium">
            </div>
          </div>
          <div class="calc-results">
            <div class="result-item">
              <span class="result-label">直接费用基数</span>
              <span class="result-value">{{ formatNumber(modalDirectBase) }} 万元</span>
              <span class="result-hint">(预计总费用 - 设备购置费 - 间接费用)</span>
            </div>
            <div class="result-item highlight">
              <span class="result-label">间接费用上限</span>
              <span class="result-value">{{ formatNumber(modalMaxIndirect) }} 万元</span>
            </div>
          </div>
          <div class="apply-section">
            <button class="btn btn-success btn-small" @click="applyCalculatedAmount">应用此金额</button>
          </div>
        </div>
        <div class="modal-footer">
          <button class="btn btn-small" @click="showCalcModal = false">关闭</button>
        </div>
      </div>
    </div>

    <div class="info-box">
      <p>间接费用是指承担单位在组织实施项目过程中发生的无法在直接费用中列支的相关费用，主要包括：</p>
      <ul>
        <li>承担单位为项目研究提供的现有仪器设备及房屋</li>
        <li>水、电、气、暖消耗</li>
        <li>有关管理费用的补助支出</li>
        <li>绩效支出等</li>
      </ul>
    </div>
    <div class="input-group">
      <label>单位：</label>
      <select v-model="unit" style="max-width: 100px">
        <option value="yuan">元</option>
        <option value="wan">万元</option>
      </select>
    </div>
    <div class="input-group">
      <label>间接费用：</label>
      <input type="number" v-model.number="amount" min="0" step="0.01" placeholder="请输入金额">
      <span class="unit-label">{{ unit === 'yuan' ? '元' : '万元' }}</span>
    </div>
    <div class="input-group">
      <label>费用说明：</label>
      <input type="text" v-model="description" placeholder="请输入费用说明" style="max-width: 400px">
    </div>
    <div class="summary">
      <span>合计：</span>
      <span class="amount-display">{{ formatNumber(amount) }} {{ unit === 'yuan' ? '元' : '万元' }}</span>
      <span v-if="unit === 'yuan'" class="convert-display">（折合 {{ formatNumber(amount / 10000) }} 万元）</span>
    </div>
    <div class="tip-box">
      <strong>提示：</strong>间接费用一般按照直接费用扣除设备购置费后的一定比例核定。
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  modelValue: {
    type: Object,
    default: () => ({
      amount: 0,
      unit: 'wan',
      description: ''
    })
  }
})

const emit = defineEmits(['update:modelValue'])

const unit = ref(props.modelValue.unit || 'wan')
const amount = ref(props.modelValue.amount || 0)
const description = ref(props.modelValue.description || '')

const showCalcModal = ref(false)
const modalExpectedTotal = ref(100)
const modalPurchaseEquipment = ref(0)

const formatNumber = (num) => {
  if (!num || isNaN(num)) return '0'
  return Number(num).toFixed(2)
}

const calcMaxIndirectForBase = (base) => {
  if (base <= 0) return 0
  let fee = 0
  if (base <= 500) {
    fee = base * 0.30
  } else if (base <= 1000) {
    fee = 500 * 0.30 + (base - 500) * 0.25
  } else {
    fee = 500 * 0.30 + 500 * 0.25 + (base - 1000) * 0.20
  }
  return fee
}

const modalDirectBase = computed(() => {
  const total = modalExpectedTotal.value
  const purchase = modalPurchaseEquipment.value
  
  if (total <= purchase) return 0
  
  let low = 0
  let high = total - purchase
  
  for (let i = 0; i < 20; i++) {
    const mid = (low + high) / 2
    const indirect = calcMaxIndirectForBase(mid)
    const sum = purchase + mid + indirect
    
    if (sum < total) {
      low = mid
    } else {
      high = mid
    }
  }
  
  return (low + high) / 2
})

const modalMaxIndirect = computed(() => {
  return calcMaxIndirectForBase(modalDirectBase.value)
})

const applyCalculatedAmount = () => {
  if (unit.value === 'yuan') {
    amount.value = modalMaxIndirect.value * 10000
  } else {
    amount.value = modalMaxIndirect.value
  }
  showCalcModal.value = false
}

const totalWan = computed(() => {
  return unit.value === 'yuan' ? amount.value / 10000 : amount.value
})

watch([amount, unit, description], () => {
  emit('update:modelValue', {
    amount: amount.value,
    unit: unit.value,
    description: description.value,
    totalWan: totalWan.value
  })
}, { immediate: true })

watch(() => props.modelValue, (newVal) => {
  if (newVal.amount !== undefined) amount.value = newVal.amount
  if (newVal.unit) unit.value = newVal.unit
  if (newVal.description !== undefined) description.value = newVal.description
}, { deep: true })
</script>

<style scoped>
.info-box {
  background: #f0f7ff;
  border: 1px solid #b3d7ff;
  border-radius: 8px;
  padding: 15px;
  margin-bottom: 20px;
}

.info-box p {
  color: #333;
  margin-bottom: 10px;
}

.info-box ul {
  margin-left: 20px;
  color: #555;
}

.info-box li {
  margin-bottom: 5px;
}

.input-group {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 15px;
}

.input-group label {
  min-width: 100px;
  font-weight: 500;
  color: #555;
}

.input-group input[type="number"] {
  max-width: 200px;
}

.unit-label {
  color: #666;
  font-size: 14px;
}

.summary {
  margin-top: 20px;
  padding: 15px;
  background: #e8f4fd;
  border-radius: 8px;
  display: flex;
  align-items: center;
  gap: 10px;
}

.summary span:first-child {
  font-weight: 600;
  color: #333;
}

.convert-display {
  color: #888;
  font-size: 14px;
}

.tip-box {
  margin-top: 15px;
  padding: 10px 15px;
  background: #fff8e6;
  border: 1px solid #ffd666;
  border-radius: 6px;
  font-size: 13px;
  color: #8a6d3b;
}

.calc-trigger {
  margin-bottom: 15px;
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
  width: 500px;
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
  margin: 0 0 8px 0;
}

.rule-content ul {
  margin: 0;
  padding-left: 20px;
}

.rule-content li {
  margin-bottom: 4px;
}

.calc-inputs {
  display: flex;
  gap: 20px;
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
  width: 120px !important;
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
  margin-bottom: 8px;
}

.result-item:last-child {
  margin-bottom: 0;
}

.result-item.highlight {
  background: #c8e6c9;
  margin: 0 -15px -12px -15px;
  padding: 12px 15px;
  border-radius: 0 0 6px 6px;
}

.result-label {
  font-size: 13px;
  color: #555;
  min-width: 100px;
}

.result-value {
  font-weight: 600;
  color: #2e7d32;
  font-size: 16px;
}

.result-item.highlight .result-value {
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
