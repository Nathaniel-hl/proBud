<template>
  <div class="section">
    <h3 class="section-title">国际合作与交流费</h3>
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
            <th style="width: 200px">交流内容</th>
            <th style="width: 120px">国家/地区</th>
            <th style="width: 100px">人数</th>
            <th style="width: 100px">天数</th>
            <th style="width: 120px">金额</th>
            <th>说明</th>
            <th style="width: 80px">操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in exchanges" :key="index">
            <td>{{ index + 1 }}</td>
            <td><input type="text" v-model="item.content" placeholder="交流内容"></td>
            <td><input type="text" v-model="item.country" placeholder="国家/地区"></td>
            <td><input type="number" v-model.number="item.people" min="1" step="1"></td>
            <td><input type="number" v-model.number="item.days" min="1" step="1"></td>
            <td><input type="number" v-model.number="item.amount" min="0" step="0.01"></td>
            <td><input type="text" v-model="item.description" placeholder="说明"></td>
            <td><button class="delete-btn" @click="removeRow(index)">删除</button></td>
          </tr>
          <tr class="total-row">
            <td colspan="5" style="text-align: right">合计：</td>
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
      exchanges: [],
      unit: 'wan'
    })
  }
})

const emit = defineEmits(['update:modelValue'])

const unit = ref(props.modelValue.unit || 'wan')
const exchanges = ref(props.modelValue.exchanges?.length > 0 
  ? props.modelValue.exchanges 
  : [{ content: '', country: '', people: 1, days: 1, amount: 0, description: '' }])

const formatNumber = (num) => {
  if (!num || isNaN(num)) return '0'
  return Number(num).toFixed(2)
}

const total = computed(() => {
  return exchanges.value.reduce((sum, item) => {
    return sum + (item.amount || 0)
  }, 0)
})

const totalWan = computed(() => {
  return unit.value === 'yuan' ? total.value / 10000 : total.value
})

const addRow = () => {
  exchanges.value.push({ content: '', country: '', people: 1, days: 1, amount: 0, description: '' })
}

const removeRow = (index) => {
  if (exchanges.value.length > 1) {
    exchanges.value.splice(index, 1)
  }
}

watch([exchanges, unit], () => {
  emit('update:modelValue', {
    exchanges: exchanges.value,
    unit: unit.value,
    totalWan: totalWan.value
  })
}, { deep: true, immediate: true })

watch(() => props.modelValue, (newVal) => {
  if (newVal.exchanges) exchanges.value = newVal.exchanges
  if (newVal.unit) unit.value = newVal.unit
}, { deep: true })
</script>

<style scoped>
</style>
