<template>
  <div class="section">
    <h3 class="section-title">设备费</h3>
    
    <!-- 购置设备费 -->
    <div class="subsection">
      <h4 class="subsection-title">1.1 购置设备费</h4>
      <div class="unit-selector">
        <label>单位：</label>
        <select v-model="purchaseUnit">
          <option value="yuan">元</option>
          <option value="wan">万元</option>
        </select>
      </div>
      <div class="table-container">
        <table>
          <thead>
            <tr>
              <th style="width: 40px">序号</th>
              <th style="width: 100px">设备名称</th>
              <th style="width: 120px">型号/性能</th>
              <th style="width: 70px">单价</th>
              <th style="width: 50px">数量</th>
              <th style="width: 80px">总价</th>
              <th style="width: 100px">用途</th>
              <th style="width: 80px">采购单位</th>
              <th style="width: 50px">操作</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(item, index) in purchaseEquipments" :key="index">
              <td>{{ index + 1 }}</td>
              <td><input type="text" v-model="item.name" placeholder="名称"></td>
              <td><input type="text" v-model="item.model" placeholder="型号"></td>
              <td><input type="number" v-model.number="item.unitPrice" min="0" step="0.01"></td>
              <td><input type="number" v-model.number="item.quantity" min="0" step="1"></td>
              <td class="amount-display">{{ formatNumber(item.unitPrice * item.quantity) }}</td>
              <td><input type="text" v-model="item.purpose" placeholder="用途"></td>
              <td><input type="text" v-model="item.supplier" placeholder="单位"></td>
              <td><button class="delete-btn" @click="removePurchaseRow(index)">删</button></td>
            </tr>
            <tr class="total-row">
              <td colspan="5" style="text-align: right">合计：</td>
              <td class="amount-display">{{ formatNumber(purchaseTotal) }}</td>
              <td colspan="3"></td>
            </tr>
          </tbody>
        </table>
      </div>
      <button class="btn btn-primary btn-small add-row-btn" @click="addPurchaseRow">+ 新增行</button>
    </div>

    <!-- 试制设备费 -->
    <div class="subsection" style="margin-top: 30px">
      <h4 class="subsection-title">1.2 试制设备费</h4>
      <div class="unit-selector">
        <label>单位：</label>
        <select v-model="trialUnit">
          <option value="yuan">元</option>
          <option value="wan">万元</option>
        </select>
      </div>
      <div class="table-container">
        <table>
          <thead>
            <tr>
              <th style="width: 40px">序号</th>
              <th style="width: 100px">设备名称</th>
              <th style="width: 120px">型号/性能</th>
              <th style="width: 70px">单价</th>
              <th style="width: 50px">数量</th>
              <th style="width: 80px">总价</th>
              <th style="width: 100px">用途</th>
              <th style="width: 80px">采购单位</th>
              <th style="width: 50px">操作</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(item, index) in trialEquipments" :key="index">
              <td>{{ index + 1 }}</td>
              <td><input type="text" v-model="item.name" placeholder="名称"></td>
              <td><input type="text" v-model="item.model" placeholder="型号"></td>
              <td><input type="number" v-model.number="item.unitPrice" min="0" step="0.01"></td>
              <td><input type="number" v-model.number="item.quantity" min="0" step="1"></td>
              <td class="amount-display">{{ formatNumber(item.unitPrice * item.quantity) }}</td>
              <td><input type="text" v-model="item.purpose" placeholder="用途"></td>
              <td><input type="text" v-model="item.supplier" placeholder="单位"></td>
              <td><button class="delete-btn" @click="removeTrialRow(index)">删</button></td>
            </tr>
            <tr class="total-row">
              <td colspan="5" style="text-align: right">合计：</td>
              <td class="amount-display">{{ formatNumber(trialTotal) }}</td>
              <td colspan="3"></td>
            </tr>
          </tbody>
        </table>
      </div>
      <button class="btn btn-primary btn-small add-row-btn" @click="addTrialRow">+ 新增行</button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  modelValue: {
    type: Object,
    default: () => ({
      purchaseEquipments: [],
      trialEquipments: [],
      purchaseUnit: 'wan',
      trialUnit: 'wan'
    })
  }
})

const emit = defineEmits(['update:modelValue'])

const purchaseUnit = ref(props.modelValue.purchaseUnit || 'wan')
const trialUnit = ref(props.modelValue.trialUnit || 'wan')

const purchaseEquipments = ref(props.modelValue.purchaseEquipments?.length > 0 
  ? props.modelValue.purchaseEquipments 
  : [{ name: '', model: '', unitPrice: 0, quantity: 1, purpose: '', supplier: '' }])

const trialEquipments = ref(props.modelValue.trialEquipments?.length > 0 
  ? props.modelValue.trialEquipments 
  : [{ name: '', model: '', unitPrice: 0, quantity: 1, purpose: '', supplier: '' }])

const formatNumber = (num) => {
  if (!num || isNaN(num)) return '0'
  return Number(num).toFixed(2)
}

const purchaseTotal = computed(() => {
  return purchaseEquipments.value.reduce((sum, item) => {
    return sum + (item.unitPrice || 0) * (item.quantity || 0)
  }, 0)
})

const trialTotal = computed(() => {
  return trialEquipments.value.reduce((sum, item) => {
    return sum + (item.unitPrice || 0) * (item.quantity || 0)
  }, 0)
})

const purchaseTotalWan = computed(() => {
  const total = purchaseTotal.value
  return purchaseUnit.value === 'yuan' ? total / 10000 : total
})

const trialTotalWan = computed(() => {
  const total = trialTotal.value
  return trialUnit.value === 'yuan' ? total / 10000 : total
})

const addPurchaseRow = () => {
  purchaseEquipments.value.push({ name: '', model: '', unitPrice: 0, quantity: 1, purpose: '', supplier: '' })
}

const removePurchaseRow = (index) => {
  if (purchaseEquipments.value.length > 1) {
    purchaseEquipments.value.splice(index, 1)
  }
}

const addTrialRow = () => {
  trialEquipments.value.push({ name: '', model: '', unitPrice: 0, quantity: 1, purpose: '', supplier: '' })
}

const removeTrialRow = (index) => {
  if (trialEquipments.value.length > 1) {
    trialEquipments.value.splice(index, 1)
  }
}

watch([purchaseEquipments, trialEquipments, purchaseUnit, trialUnit], () => {
  emit('update:modelValue', {
    purchaseEquipments: purchaseEquipments.value,
    trialEquipments: trialEquipments.value,
    purchaseUnit: purchaseUnit.value,
    trialUnit: trialUnit.value,
    purchaseTotalWan: purchaseTotalWan.value,
    trialTotalWan: trialTotalWan.value
  })
}, { deep: true, immediate: true })

watch(() => props.modelValue, (newVal) => {
  if (newVal.purchaseEquipments) purchaseEquipments.value = newVal.purchaseEquipments
  if (newVal.trialEquipments) trialEquipments.value = newVal.trialEquipments
  if (newVal.purchaseUnit) purchaseUnit.value = newVal.purchaseUnit
  if (newVal.trialUnit) trialUnit.value = newVal.trialUnit
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
</style>
