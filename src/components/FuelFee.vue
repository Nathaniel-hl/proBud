<template>
  <div class="section">
    <h3 class="section-title">燃料动力费</h3>
    <div class="input-group">
      <label>单位：</label>
      <select v-model="unit" style="max-width: 100px">
        <option value="yuan">元</option>
        <option value="wan">万元</option>
      </select>
    </div>
    <div class="input-group">
      <label>燃料动力费：</label>
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

const formatNumber = (num) => {
  if (!num || isNaN(num)) return '0'
  return Number(num).toFixed(2)
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
</style>
