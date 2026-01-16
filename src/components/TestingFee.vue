<template>
  <div class="section">
    <h3 class="section-title">测试化验加工费</h3>
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
            <th style="width: 250px">测试/加工内容</th>
            <th style="width: 150px">测试/加工类别</th>
            <th style="width: 150px">内部/外部委托</th>
            <th style="width: 150px">金额</th>
            <th style="width: 80px">操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in testings" :key="index">
            <td>{{ index + 1 }}</td>
            <td><input type="text" v-model="item.content" placeholder="测试/加工内容"></td>
            <td>
              <select v-model="item.category">
                <option value="">请选择</option>
                <option value="测试">测试</option>
                <option value="化验">化验</option>
                <option value="加工">加工</option>
              </select>
            </td>
            <td>
              <select v-model="item.delegateType">
                <option value="">请选择</option>
                <option value="内部">内部</option>
                <option value="外部委托">外部委托</option>
              </select>
            </td>
            <td><input type="number" v-model.number="item.amount" min="0" step="0.01"></td>
            <td><button class="delete-btn" @click="removeRow(index)">删除</button></td>
          </tr>
          <tr class="total-row">
            <td colspan="4" style="text-align: right">合计：</td>
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
import { ref, computed, watch } from 'vue'

const props = defineProps({
  modelValue: {
    type: Object,
    default: () => ({
      testings: [],
      unit: 'wan'
    })
  }
})

const emit = defineEmits(['update:modelValue'])

const unit = ref(props.modelValue.unit || 'wan')
const testings = ref(props.modelValue.testings?.length > 0 
  ? props.modelValue.testings 
  : [{ content: '', category: '', delegateType: '', amount: 0 }])

const formatNumber = (num) => {
  if (!num || isNaN(num)) return '0'
  return Number(num).toFixed(2)
}

const total = computed(() => {
  return testings.value.reduce((sum, item) => {
    return sum + (item.amount || 0)
  }, 0)
})

const totalWan = computed(() => {
  return unit.value === 'yuan' ? total.value / 10000 : total.value
})

const addRow = () => {
  testings.value.push({ content: '', category: '', delegateType: '', amount: 0 })
}

const removeRow = (index) => {
  if (testings.value.length > 1) {
    testings.value.splice(index, 1)
  }
}

watch([testings, unit], () => {
  emit('update:modelValue', {
    testings: testings.value,
    unit: unit.value,
    totalWan: totalWan.value
  })
}, { deep: true, immediate: true })

watch(() => props.modelValue, (newVal) => {
  if (newVal.testings) testings.value = newVal.testings
  if (newVal.unit) unit.value = newVal.unit
}, { deep: true })
</script>

<style scoped>
</style>
