<template>
  <div class="section">
    <h3 class="section-title">材料费</h3>
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
            <th style="width: 40px">序号</th>
            <th style="width: 120px">材料名称</th>
            <th style="width: 70px">单价</th>
            <th style="width: 60px">数量</th>
            <th style="width: 80px">金额</th>
            <th style="width: 150px">测算依据</th>
            <th style="width: 50px">操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in materials" :key="index">
            <td>{{ index + 1 }}</td>
            <td><input type="text" v-model="item.name" placeholder="名称"></td>
            <td><input type="number" v-model.number="item.unitPrice" min="0" step="0.01"></td>
            <td><input type="number" v-model.number="item.quantity" min="0" step="1"></td>
            <td class="amount-display">{{ formatNumber(item.unitPrice * item.quantity) }}</td>
            <td><input type="text" v-model="item.basis" placeholder="依据"></td>
            <td><button class="delete-btn" @click="removeRow(index)">删</button></td>
          </tr>
          <tr class="total-row">
            <td colspan="4" style="text-align: right">合计：</td>
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
      materials: [],
      unit: 'wan'
    })
  }
})

const emit = defineEmits(['update:modelValue'])

const unit = ref(props.modelValue.unit || 'wan')
const materials = ref(props.modelValue.materials?.length > 0 
  ? props.modelValue.materials 
  : [{ name: '', unitPrice: 0, quantity: 1, basis: '' }])

const formatNumber = (num) => {
  if (!num || isNaN(num)) return '0'
  return Number(num).toFixed(2)
}

const total = computed(() => {
  return materials.value.reduce((sum, item) => {
    return sum + (item.unitPrice || 0) * (item.quantity || 0)
  }, 0)
})

const totalWan = computed(() => {
  return unit.value === 'yuan' ? total.value / 10000 : total.value
})

const addRow = () => {
  materials.value.push({ name: '', unitPrice: 0, quantity: 1, basis: '' })
}

const removeRow = (index) => {
  if (materials.value.length > 1) {
    materials.value.splice(index, 1)
  }
}

watch([materials, unit], () => {
  emit('update:modelValue', {
    materials: materials.value,
    unit: unit.value,
    totalWan: totalWan.value
  })
}, { deep: true, immediate: true })

watch(() => props.modelValue, (newVal) => {
  if (newVal.materials) materials.value = newVal.materials
  if (newVal.unit) unit.value = newVal.unit
}, { deep: true })
</script>

<style scoped>
</style>
