<template>
  <div class="section">
    <h3 class="section-title">出版文献、信息传播和知识产权事务费</h3>
    
    <!-- 打印凑整工具 -->
    <div class="print-calculator">
      <div class="calc-header">
        <span class="calc-label">打印凑整</span>
        <button class="btn btn-small" @click="showCalculator = !showCalculator">
          {{ showCalculator ? '收起' : '展开' }}
        </button>
      </div>
      <div v-if="showCalculator" class="calc-content">
        <div class="calc-row">
          <div class="calc-field">
            <label>项目总预算(万)</label>
            <input type="number" v-model.number="targetTotal" min="0" step="0.1" class="input-small">
          </div>
          <div class="calc-field">
            <label>当前其他费用(万)</label>
            <input type="number" v-model.number="currentOtherTotal" min="0" step="0.01" class="input-small" readonly>
          </div>
          <div class="calc-field">
            <label>差额(万)</label>
            <span class="calc-value">{{ formatNumber(gap) }}</span>
          </div>
        </div>
        <div class="calc-result" v-if="printItem">
          <div class="result-info">
            <span>打印单价：<strong>{{ printItem.unitPrice }}</strong> 元</span>
            <span>需打印：<strong>{{ neededPrintQuantity }}</strong> 份</span>
            <span>打印费：<strong>{{ formatNumber(neededPrintQuantity * printItem.unitPrice / 10000) }}</strong> 万</span>
          </div>
          <button class="btn btn-success btn-small" @click="applyPrintQuantity" :disabled="neededPrintQuantity <= 0">
            应用
          </button>
        </div>
        <div v-else class="no-print-hint">
          请确保表格中有"打印"条目
        </div>
      </div>
    </div>

    <div class="table-container">
      <table>
        <thead>
          <tr>
            <th style="width: 40px">序号</th>
            <th style="width: 120px">费用类型</th>
            <th style="width: 80px">单价(元)</th>
            <th style="width: 50px">数量</th>
            <th style="width: 80px">金额(万)</th>
            <th style="width: 50px">操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in publications" :key="index">
            <td>{{ index + 1 }}</td>
            <td><input type="text" v-model="item.type" placeholder="类型"></td>
            <td><input type="number" v-model.number="item.unitPrice" min="0" step="1"></td>
            <td><input type="number" v-model.number="item.quantity" min="0" step="1"></td>
            <td class="amount-display">{{ formatNumber((item.unitPrice || 0) * (item.quantity || 0) / 10000) }}</td>
            <td><button class="delete-btn" @click="removeRow(index)">删</button></td>
          </tr>
          <tr class="total-row">
            <td colspan="4" style="text-align: right">合计：</td>
            <td class="amount-display">{{ formatNumber(totalWan) }}</td>
            <td></td>
          </tr>
        </tbody>
      </table>
    </div>
    <button class="btn btn-primary btn-small add-row-btn" @click="addRow">+ 新增行</button>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const DEFAULT_ITEMS = [
  { type: '论文版面费', unitPrice: 20000, quantity: 0 },
  { type: '专利申请费', unitPrice: 8000, quantity: 0 },
  { type: '软件著作权', unitPrice: 5000, quantity: 0 },
  { type: '查新费与检索费', unitPrice: 2000, quantity: 0 },
  { type: '打印', unitPrice: 50, quantity: 0 },
  { type: '图书购买', unitPrice: 180, quantity: 0 }
]

const props = defineProps({
  modelValue: {
    type: Object,
    default: () => ({
      publications: []
    })
  },
  currentTotalExcludePrint: {
    type: Number,
    default: 0
  }
})

const emit = defineEmits(['update:modelValue'])

const showCalculator = ref(false)
const targetTotal = ref(100)

const publications = ref(props.modelValue.publications?.length > 0 
  ? props.modelValue.publications 
  : DEFAULT_ITEMS.map(item => ({ ...item })))

const formatNumber = (num) => {
  if (!num || isNaN(num)) return '0.00'
  return Number(num).toFixed(2)
}

const totalWan = computed(() => {
  return publications.value.reduce((sum, item) => {
    return sum + (item.unitPrice || 0) * (item.quantity || 0) / 10000
  }, 0)
})

const printItem = computed(() => {
  return publications.value.find(item => item.type === '打印')
})

const totalExcludePrint = computed(() => {
  return publications.value.reduce((sum, item) => {
    if (item.type === '打印') return sum
    return sum + (item.unitPrice || 0) * (item.quantity || 0) / 10000
  }, 0)
})

const currentOtherTotal = computed(() => {
  return props.currentTotalExcludePrint + totalExcludePrint.value
})

const gap = computed(() => {
  return targetTotal.value - currentOtherTotal.value
})

const neededPrintQuantity = computed(() => {
  if (!printItem.value || printItem.value.unitPrice <= 0 || gap.value <= 0) return 0
  return Math.ceil(gap.value * 10000 / printItem.value.unitPrice)
})

const applyPrintQuantity = () => {
  if (printItem.value && neededPrintQuantity.value > 0) {
    printItem.value.quantity = neededPrintQuantity.value
  }
}

const addRow = () => {
  publications.value.push({ type: '', unitPrice: 0, quantity: 0 })
}

const removeRow = (index) => {
  if (publications.value.length > 1) {
    publications.value.splice(index, 1)
  }
}

watch(publications, () => {
  emit('update:modelValue', {
    publications: publications.value,
    totalWan: totalWan.value
  })
}, { deep: true, immediate: true })

watch(() => props.modelValue, (newVal) => {
  if (newVal.publications) publications.value = newVal.publications
}, { deep: true })
</script>

<style scoped>
.print-calculator {
  background: #e3f2fd;
  border: 1px solid #90caf9;
  border-radius: 6px;
  padding: 10px 12px;
  margin-bottom: 12px;
}

.calc-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.calc-label {
  font-weight: 600;
  color: #1565c0;
  font-size: 14px;
}

.calc-content {
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px dashed #90caf9;
}

.calc-row {
  display: flex;
  gap: 20px;
  flex-wrap: wrap;
  margin-bottom: 10px;
}

.calc-field {
  display: flex;
  align-items: center;
  gap: 6px;
}

.calc-field label {
  font-size: 12px;
  color: #555;
  white-space: nowrap;
}

.input-small {
  width: 70px !important;
  text-align: right;
}

.calc-value {
  font-weight: 600;
  color: #1565c0;
  font-size: 14px;
}

.calc-result {
  display: flex;
  align-items: center;
  gap: 15px;
  padding: 8px 10px;
  background: #bbdefb;
  border-radius: 4px;
}

.result-info {
  display: flex;
  gap: 15px;
  font-size: 12px;
  color: #333;
}

.result-info strong {
  color: #1565c0;
}

.no-print-hint {
  padding: 10px;
  background: #fff3e0;
  border: 1px solid #ffcc80;
  border-radius: 4px;
  color: #e65100;
  font-size: 12px;
  text-align: center;
}
</style>
