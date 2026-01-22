<template>
  <div class="section">
    <h3 class="section-title">经费测算表</h3>
    
    <!-- 操作按钮 -->
    <div class="calc-trigger">
      <button class="btn btn-primary btn-small" @click="showCalcModal = true">
        计算间接费用上限
      </button>
      <button class="btn btn-small" :class="showDesc ? 'btn-info' : ''" @click="showDesc = !showDesc">
        {{ showDesc ? '隐藏说明' : '显示说明' }}
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
        </div>
        <div class="modal-footer">
          <button class="btn btn-small" @click="showCalcModal = false">关闭</button>
        </div>
      </div>
    </div>

    <div class="table-container">
      <table class="summary-table">
        <thead>
          <tr>
            <th style="width: 40px">序号</th>
            <th style="width: 180px">预算科目名称</th>
            <th style="width: 70px">比例要求</th>
            <th style="width: 90px">国拨(万)</th>
            <th style="width: 55px">国拨占比</th>
            <th style="width: 90px">其他来源(万)</th>
            <th style="width: 55px">其他占比</th>
            <th style="width: 90px">合计(万)</th>
            <th style="width: 55px">合计占比</th>
            <th v-if="showDesc">说明</th>
          </tr>
        </thead>
        <tbody>
          <!-- 直接费用 -->
          <tr>
            <td>1</td>
            <td>（一）直接费用</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(directTotal) }}</td>
            <td class="percent-display">{{ formatPercent(directTotal) }}</td>
            <td class="amount-display">{{ formatNumber(otherDirectTotal) }}</td>
            <td class="percent-display">{{ formatOtherPercent(otherDirectTotal) }}</td>
            <td class="amount-display">{{ formatNumber(directTotal + otherDirectTotal) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(directTotal + otherDirectTotal) }}</td>
            <td v-if="showDesc" class="desc-cell"></td>
          </tr>
          <!-- 设备费 -->
          <tr :class="{ 'warning-row': equipmentExceeded }">
            <td>2</td>
            <td class="indent-1">1.设备费</td>
            <td class="ratio-cell">≤10%</td>
            <td class="amount-display">{{ formatNumber(equipmentTotal) }}</td>
            <td class="percent-display">{{ formatPercent(equipmentTotal) }}</td>
            <td class="amount-display">{{ formatNumber(otherEquipmentTotal) }}</td>
            <td class="percent-display">{{ formatOtherPercent(otherEquipmentTotal) }}</td>
            <td class="amount-display">{{ formatNumber(equipmentTotal + otherEquipmentTotal) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(equipmentTotal + otherEquipmentTotal) }}</td>
            <td v-if="showDesc" class="desc-cell">
              国拨设备费比例不超10%
              <span v-if="equipmentExceeded" class="warning-text">（国拨{{ formatPercent(equipmentTotal) }}，超限）</span>
              <span v-else class="success-text">（符合）</span>
            </td>
          </tr>
          <!-- 购置设备费 -->
          <tr>
            <td>3</td>
            <td class="indent-2">1.1 购置设备费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(purchaseEquipmentTotal) }}</td>
            <td class="percent-display">{{ formatPercent(purchaseEquipmentTotal) }}</td>
            <td><input type="number" class="inline-input" v-model.number="otherFunding.purchaseEquipment" min="0" step="0.01"></td>
            <td class="percent-display">{{ formatOtherPercent(otherFunding.purchaseEquipment) }}</td>
            <td class="amount-display">{{ formatNumber(purchaseEquipmentTotal + otherFunding.purchaseEquipment) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(purchaseEquipmentTotal + otherFunding.purchaseEquipment) }}</td>
            <td v-if="showDesc" class="desc-cell">专用设备</td>
          </tr>
          <!-- 试制设备费 -->
          <tr>
            <td>4</td>
            <td class="indent-2">1.2 试制设备费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(trialEquipmentTotal) }}</td>
            <td class="percent-display">{{ formatPercent(trialEquipmentTotal) }}</td>
            <td><input type="number" class="inline-input" v-model.number="otherFunding.trialEquipment" min="0" step="0.01"></td>
            <td class="percent-display">{{ formatOtherPercent(otherFunding.trialEquipment) }}</td>
            <td class="amount-display">{{ formatNumber(trialEquipmentTotal + otherFunding.trialEquipment) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(trialEquipmentTotal + otherFunding.trialEquipment) }}</td>
            <td v-if="showDesc" class="desc-cell">样机或原型系统</td>
          </tr>
          <!-- 业务费 -->
          <tr>
            <td>5</td>
            <td class="indent-1">2.业务费</td>
            <td class="ratio-cell">按实际计算</td>
            <td class="amount-display">{{ formatNumber(businessTotal) }}</td>
            <td class="percent-display">{{ formatPercent(businessTotal) }}</td>
            <td class="amount-display">{{ formatNumber(otherBusinessTotal) }}</td>
            <td class="percent-display">{{ formatOtherPercent(otherBusinessTotal) }}</td>
            <td class="amount-display">{{ formatNumber(businessTotal + otherBusinessTotal) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(businessTotal + otherBusinessTotal) }}</td>
            <td v-if="showDesc" class="desc-cell">据实编制预算</td>
          </tr>
          <!-- 材料费 -->
          <tr>
            <td>6</td>
            <td class="indent-2">2.1 材料费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(materialTotal) }}</td>
            <td class="percent-display">{{ formatPercent(materialTotal) }}</td>
            <td><input type="number" class="inline-input" v-model.number="otherFunding.material" min="0" step="0.01"></td>
            <td class="percent-display">{{ formatOtherPercent(otherFunding.material) }}</td>
            <td class="amount-display">{{ formatNumber(materialTotal + otherFunding.material) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(materialTotal + otherFunding.material) }}</td>
            <td v-if="showDesc" class="desc-cell"></td>
          </tr>
          <!-- 测试化验加工费 -->
          <tr>
            <td>7</td>
            <td class="indent-2">2.2 测试化验加工费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(testingTotal) }}</td>
            <td class="percent-display">{{ formatPercent(testingTotal) }}</td>
            <td><input type="number" class="inline-input" v-model.number="otherFunding.testing" min="0" step="0.01"></td>
            <td class="percent-display">{{ formatOtherPercent(otherFunding.testing) }}</td>
            <td class="amount-display">{{ formatNumber(testingTotal + otherFunding.testing) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(testingTotal + otherFunding.testing) }}</td>
            <td v-if="showDesc" class="desc-cell">外单位检验测试费用</td>
          </tr>
          <!-- 燃料动力费 -->
          <tr>
            <td>8</td>
            <td class="indent-2">2.3 燃料动力费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(fuelTotal) }}</td>
            <td class="percent-display">{{ formatPercent(fuelTotal) }}</td>
            <td><input type="number" class="inline-input" v-model.number="otherFunding.fuel" min="0" step="0.01"></td>
            <td class="percent-display">{{ formatOtherPercent(otherFunding.fuel) }}</td>
            <td class="amount-display">{{ formatNumber(fuelTotal + otherFunding.fuel) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(fuelTotal + otherFunding.fuel) }}</td>
            <td v-if="showDesc" class="desc-cell"></td>
          </tr>
          <!-- 差旅费 -->
          <tr :class="{ 'warning-row': travelMeetingExceeded }">
            <td>9</td>
            <td class="indent-2">2.4 差旅费</td>
            <td class="ratio-cell" rowspan="3">合计≤10%</td>
            <td class="amount-display">{{ formatNumber(travelTotal) }}</td>
            <td class="percent-display">{{ formatPercent(travelTotal) }}</td>
            <td><input type="number" class="inline-input" v-model.number="otherFunding.travel" min="0" step="0.01"></td>
            <td class="percent-display">{{ formatOtherPercent(otherFunding.travel) }}</td>
            <td class="amount-display">{{ formatNumber(travelTotal + otherFunding.travel) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(travelTotal + otherFunding.travel) }}</td>
            <td v-if="showDesc" class="desc-cell" rowspan="3">
              差旅+会议+国际合作≤10%
              <span v-if="travelMeetingExceeded" class="warning-text">（国拨{{ formatPercent(travelMeetingTotal) }}，超限）</span>
              <span v-else class="success-text">（国拨{{ formatPercent(travelMeetingTotal) }}）</span>
            </td>
          </tr>
          <!-- 会议费 -->
          <tr :class="{ 'warning-row': travelMeetingExceeded }">
            <td>10</td>
            <td class="indent-2">2.5 会议费</td>
            <td class="amount-display">{{ formatNumber(meetingTotal) }}</td>
            <td class="percent-display">{{ formatPercent(meetingTotal) }}</td>
            <td><input type="number" class="inline-input" v-model.number="otherFunding.meeting" min="0" step="0.01"></td>
            <td class="percent-display">{{ formatOtherPercent(otherFunding.meeting) }}</td>
            <td class="amount-display">{{ formatNumber(meetingTotal + otherFunding.meeting) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(meetingTotal + otherFunding.meeting) }}</td>
          </tr>
          <!-- 国际合作与交流费 -->
          <tr :class="{ 'warning-row': travelMeetingExceeded }">
            <td>11</td>
            <td class="indent-2">2.6 国际合作与交流费</td>
            <td class="amount-display">{{ formatNumber(internationalTotal) }}</td>
            <td class="percent-display">{{ formatPercent(internationalTotal) }}</td>
            <td><input type="number" class="inline-input" v-model.number="otherFunding.international" min="0" step="0.01"></td>
            <td class="percent-display">{{ formatOtherPercent(otherFunding.international) }}</td>
            <td class="amount-display">{{ formatNumber(internationalTotal + otherFunding.international) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(internationalTotal + otherFunding.international) }}</td>
          </tr>
          <!-- 出版文献费 -->
          <tr>
            <td>12</td>
            <td class="indent-2">2.7 出版文献费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(publicationTotal) }}</td>
            <td class="percent-display">{{ formatPercent(publicationTotal) }}</td>
            <td><input type="number" class="inline-input" v-model.number="otherFunding.publication" min="0" step="0.01"></td>
            <td class="percent-display">{{ formatOtherPercent(otherFunding.publication) }}</td>
            <td class="amount-display">{{ formatNumber(publicationTotal + otherFunding.publication) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(publicationTotal + otherFunding.publication) }}</td>
            <td v-if="showDesc" class="desc-cell">对照课题成果数量计算</td>
          </tr>
          <!-- 其他费用 -->
          <tr>
            <td>13</td>
            <td class="indent-2">2.8 其他费用</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(otherTotal) }}</td>
            <td class="percent-display">{{ formatPercent(otherTotal) }}</td>
            <td><input type="number" class="inline-input" v-model.number="otherFunding.other" min="0" step="0.01"></td>
            <td class="percent-display">{{ formatOtherPercent(otherFunding.other) }}</td>
            <td class="amount-display">{{ formatNumber(otherTotal + otherFunding.other) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(otherTotal + otherFunding.other) }}</td>
            <td v-if="showDesc" class="desc-cell">审计费等</td>
          </tr>
          <!-- 劳务费 -->
          <tr :class="{ 'warning-row': laborExceeded }">
            <td>14</td>
            <td class="indent-1">3.劳务费</td>
            <td class="ratio-cell">≤35%</td>
            <td class="amount-display">{{ formatNumber(laborTotal) }}</td>
            <td class="percent-display">{{ formatPercent(laborTotal) }}</td>
            <td class="amount-display">{{ formatNumber(otherLaborTotal) }}</td>
            <td class="percent-display">{{ formatOtherPercent(otherLaborTotal) }}</td>
            <td class="amount-display">{{ formatNumber(laborTotal + otherLaborTotal) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(laborTotal + otherLaborTotal) }}</td>
            <td v-if="showDesc" class="desc-cell">
              国拨劳务费≤35%
              <span v-if="laborExceeded" class="warning-text">（国拨{{ formatPercent(laborTotal) }}，超限）</span>
              <span v-else class="success-text">（符合）</span>
            </td>
          </tr>
          <!-- 人员劳务费 -->
          <tr>
            <td>15</td>
            <td class="indent-2">3.1 人员劳务费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(personnelTotal) }}</td>
            <td class="percent-display">{{ formatPercent(personnelTotal) }}</td>
            <td><input type="number" class="inline-input" v-model.number="otherFunding.personnel" min="0" step="0.01"></td>
            <td class="percent-display">{{ formatOtherPercent(otherFunding.personnel) }}</td>
            <td class="amount-display">{{ formatNumber(personnelTotal + otherFunding.personnel) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(personnelTotal + otherFunding.personnel) }}</td>
            <td v-if="showDesc" class="desc-cell">研究生、博士后等劳务费</td>
          </tr>
          <!-- 专家咨询费 -->
          <tr>
            <td>16</td>
            <td class="indent-2">3.2 专家咨询费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(expertTotal) }}</td>
            <td class="percent-display">{{ formatPercent(expertTotal) }}</td>
            <td><input type="number" class="inline-input" v-model.number="otherFunding.expert" min="0" step="0.01"></td>
            <td class="percent-display">{{ formatOtherPercent(otherFunding.expert) }}</td>
            <td class="amount-display">{{ formatNumber(expertTotal + otherFunding.expert) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(expertTotal + otherFunding.expert) }}</td>
            <td v-if="showDesc" class="desc-cell">参照财科教（2017）128号</td>
          </tr>
          <!-- 间接费用 -->
          <tr :class="{ 'warning-row': indirectExceeded }">
            <td>17</td>
            <td>（二）间接费用</td>
            <td class="ratio-cell">按实际计算</td>
            <td class="amount-display">{{ formatNumber(indirectTotal) }}</td>
            <td class="percent-display">{{ formatPercent(indirectTotal) }}</td>
            <td><input type="number" class="inline-input" v-model.number="otherFunding.indirect" min="0" step="0.01"></td>
            <td class="percent-display">{{ formatOtherPercent(otherFunding.indirect) }}</td>
            <td class="amount-display">{{ formatNumber(indirectTotal + otherFunding.indirect) }}</td>
            <td class="percent-display">{{ formatCombinedPercent(indirectTotal + otherFunding.indirect) }}</td>
            <td v-if="showDesc" class="desc-cell">
              国拨间接费用上限{{ formatNumber(maxIndirectFee) }}万
              <span v-if="indirectExceeded" class="warning-text">（超出{{ formatNumber(indirectTotal - maxIndirectFee) }}万）</span>
              <span v-else-if="directTotal > 0" class="success-text">（符合）</span>
            </td>
          </tr>
          <!-- 合计行 -->
          <tr class="total-row">
            <td></td>
            <td><strong>合计</strong></td>
            <td></td>
            <td class="amount-display"><strong>{{ formatNumber(grandTotal) }}</strong></td>
            <td class="percent-display"><strong>100.00%</strong></td>
            <td class="amount-display"><strong>{{ formatNumber(otherGrandTotal) }}</strong></td>
            <td class="percent-display"><strong>100.00%</strong></td>
            <td class="amount-display"><strong>{{ formatNumber(combinedGrandTotal) }}</strong></td>
            <td class="percent-display"><strong>100.00%</strong></td>
            <td v-if="showDesc"></td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, toRef } from 'vue'

const props = defineProps({
  budgetData: {
    type: Object,
    required: true
  }
})

const otherFunding = toRef(props.budgetData, 'otherFunding')

const showCalcModal = ref(false)
const showDesc = ref(true)
const modalExpectedTotal = ref(100)
const modalPurchaseEquipment = ref(0)

watch(() => showCalcModal.value, (val) => {
  if (val) {
    modalPurchaseEquipment.value = purchaseEquipmentTotal.value
  }
})

const formatNumber = (num) => {
  if (!num || isNaN(num)) return '0.00'
  return Number(num).toFixed(2)
}

const purchaseEquipmentTotal = computed(() => props.budgetData.equipment?.purchaseTotalWan || 0)
const trialEquipmentTotal = computed(() => props.budgetData.equipment?.trialTotalWan || 0)
const equipmentTotal = computed(() => purchaseEquipmentTotal.value + trialEquipmentTotal.value)

const materialTotal = computed(() => props.budgetData.material?.totalWan || 0)
const testingTotal = computed(() => props.budgetData.testing?.totalWan || 0)
const fuelTotal = computed(() => props.budgetData.fuel?.totalWan || 0)
const travelTotal = computed(() => props.budgetData.travel?.totalWan || 0)
const meetingTotal = computed(() => props.budgetData.meeting?.totalWan || 0)
const internationalTotal = computed(() => props.budgetData.international?.totalWan || 0)
const publicationTotal = computed(() => props.budgetData.publication?.totalWan || 0)
const otherTotal = computed(() => props.budgetData.other?.totalWan || 0)

const businessTotal = computed(() => {
  return materialTotal.value + testingTotal.value + fuelTotal.value + 
         travelTotal.value + meetingTotal.value + internationalTotal.value + 
         publicationTotal.value + otherTotal.value
})

const personnelTotal = computed(() => props.budgetData.labor?.personnelTotalWan || 0)
const expertTotal = computed(() => props.budgetData.labor?.expertTotalWan || 0)
const laborTotal = computed(() => personnelTotal.value + expertTotal.value)

const directTotal = computed(() => equipmentTotal.value + businessTotal.value + laborTotal.value)

const indirectTotal = computed(() => props.budgetData.indirect?.totalWan || 0)

const grandTotal = computed(() => directTotal.value + indirectTotal.value)

const travelMeetingTotal = computed(() => travelTotal.value + meetingTotal.value + internationalTotal.value)

const equipmentExceeded = computed(() => {
  if (grandTotal.value <= 0) return false
  return (equipmentTotal.value / grandTotal.value) > 0.10
})

const travelMeetingExceeded = computed(() => {
  if (grandTotal.value <= 0) return false
  return (travelMeetingTotal.value / grandTotal.value) > 0.10
})

const laborExceeded = computed(() => {
  if (grandTotal.value <= 0) return false
  return (laborTotal.value / grandTotal.value) > 0.35
})

const indirectBase = computed(() => {
  return directTotal.value - purchaseEquipmentTotal.value
})

const maxIndirectFee = computed(() => {
  return calcMaxIndirectForBase(indirectBase.value)
})

const indirectExceeded = computed(() => {
  if (maxIndirectFee.value <= 0) return false
  return indirectTotal.value > maxIndirectFee.value
})

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

const formatPercent = (amount) => {
  if (grandTotal.value <= 0) return '0.00%'
  const percent = (amount / grandTotal.value) * 100
  return percent.toFixed(2) + '%'
}

// 其他资金来源计算
const otherEquipmentTotal = computed(() => {
  return (otherFunding.value?.purchaseEquipment || 0) + (otherFunding.value?.trialEquipment || 0)
})

const otherBusinessTotal = computed(() => {
  return (otherFunding.value?.material || 0) + 
         (otherFunding.value?.testing || 0) + 
         (otherFunding.value?.fuel || 0) + 
         (otherFunding.value?.travel || 0) + 
         (otherFunding.value?.meeting || 0) + 
         (otherFunding.value?.international || 0) + 
         (otherFunding.value?.publication || 0) + 
         (otherFunding.value?.other || 0)
})

const otherLaborTotal = computed(() => {
  return (otherFunding.value?.personnel || 0) + (otherFunding.value?.expert || 0)
})

const otherDirectTotal = computed(() => {
  return otherEquipmentTotal.value + otherBusinessTotal.value + otherLaborTotal.value
})

const otherGrandTotal = computed(() => {
  return otherDirectTotal.value + (otherFunding.value?.indirect || 0)
})

const combinedGrandTotal = computed(() => {
  return grandTotal.value + otherGrandTotal.value
})

// 其他资金来源占比
const formatOtherPercent = (amount) => {
  if (otherGrandTotal.value <= 0) return '0.00%'
  const percent = (amount / otherGrandTotal.value) * 100
  return percent.toFixed(2) + '%'
}

// 合计占比
const formatCombinedPercent = (amount) => {
  if (combinedGrandTotal.value <= 0) return '0.00%'
  const percent = (amount / combinedGrandTotal.value) * 100
  return percent.toFixed(2) + '%'
}
</script>

<style scoped>
.calc-trigger {
  margin-bottom: 15px;
  display: flex;
  gap: 10px;
}

.btn-info {
  background: #17a2b8;
  color: white;
}

.btn-info:hover {
  background: #138496;
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
}

.modal-header h4 {
  margin: 0;
  font-size: 16px;
  color: #333;
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

.modal-footer {
  padding: 15px 20px;
  border-top: 1px solid #eee;
  text-align: right;
}

.summary-table {
  margin-top: 10px;
}

.summary-table td {
  padding: 8px 10px;
  font-size: 13px;
}

.summary-table .amount-display,
.summary-table .percent-display {
  text-align: right;
  font-family: 'Courier New', monospace;
}

.ratio-cell {
  text-align: center;
  color: #d32f2f;
  font-weight: 500;
  font-size: 12px;
}

.desc-cell {
  font-size: 11px;
  color: #666;
  line-height: 1.4;
}

.text-highlight {
  color: #1976d2;
  font-weight: 500;
}

.percent-display {
  color: #666;
  font-size: 12px;
}

.amount-display {
  font-size: 13px;
}

.indent-1 {
  padding-left: 25px !important;
}

.indent-2 {
  padding-left: 40px !important;
}

.total-row {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%) !important;
}

.total-row td {
  color: white !important;
}

.total-row .amount-display,
.total-row .percent-display {
  color: white !important;
}

.warning-row {
  background-color: #fff3e0 !important;
}

.warning-row td {
  color: #e65100 !important;
}

.warning-text {
  color: #d32f2f;
  font-weight: 600;
  font-size: 11px;
}

.success-text {
  color: #2e7d32;
  font-weight: 500;
  font-size: 11px;
}

.inline-input {
  width: 70px;
  padding: 4px 6px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 12px;
  text-align: right;
  font-family: 'Courier New', monospace;
}

.inline-input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 2px rgba(102, 126, 234, 0.2);
}

.inline-input::-webkit-outer-spin-button,
.inline-input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}
</style>
