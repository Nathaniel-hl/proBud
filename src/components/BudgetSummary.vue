<template>
  <div class="section">
    <h3 class="section-title">经费测算表</h3>
    
    <!-- 间接费用计算按钮 -->
    <div class="calc-trigger">
      <button class="btn btn-primary btn-small" @click="showCalcModal = true">
        计算间接费用上限
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
            <th style="width: 200px">预算科目名称</th>
            <th style="width: 80px">比例要求</th>
            <th style="width: 100px">金额(万)</th>
            <th style="width: 60px">占比</th>
            <th>说明</th>
          </tr>
        </thead>
        <tbody>
          <tr>
            <td>1</td>
            <td>（一）直接费用</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(directTotal) }}</td>
            <td class="percent-display">{{ formatPercent(directTotal) }}</td>
            <td class="desc-cell">自筹</td>
          </tr>
          <tr :class="{ 'warning-row': equipmentExceeded }">
            <td>2</td>
            <td class="indent-1">1.设备费</td>
            <td class="ratio-cell">≤10%</td>
            <td class="amount-display">{{ formatNumber(equipmentTotal) }}</td>
            <td class="percent-display">{{ formatPercent(equipmentTotal) }}</td>
            <td class="desc-cell">
              国拨设备费比例不超10%，50万元以上设备需提供详细说明
              <span v-if="equipmentExceeded" class="warning-text">（当前{{ formatPercent(equipmentTotal) }}，超过10%限制）</span>
              <span v-else class="success-text">（符合要求）</span>
            </td>
          </tr>
          <tr>
            <td>3</td>
            <td class="indent-2">1.1 购置设备费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(purchaseEquipmentTotal) }}</td>
            <td class="percent-display">{{ formatPercent(purchaseEquipmentTotal) }}</td>
            <td class="desc-cell">用软件、计算类设备服务器、数控机床、工业机器人等专用设备</td>
          </tr>
          <tr>
            <td>4</td>
            <td class="indent-2">1.2 试制设备费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(trialEquipmentTotal) }}</td>
            <td class="percent-display">{{ formatPercent(trialEquipmentTotal) }}</td>
            <td class="desc-cell">为特定研发目标定制开发或改造的设备、样机或原型系统</td>
          </tr>
          <tr>
            <td>5</td>
            <td class="indent-1">2.业务费</td>
            <td class="ratio-cell">按实际计算</td>
            <td class="amount-display">{{ formatNumber(businessTotal) }}</td>
            <td class="percent-display">{{ formatPercent(businessTotal) }}</td>
            <td class="desc-cell">据实编制预算，说明主要用途和测算理由</td>
          </tr>
          <tr>
            <td>6</td>
            <td class="indent-2">2.1 材料费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(materialTotal) }}</td>
            <td class="percent-display">{{ formatPercent(materialTotal) }}</td>
            <td class="desc-cell"></td>
          </tr>
          <tr>
            <td>7</td>
            <td class="indent-2">2.2 测试化验加工费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(testingTotal) }}</td>
            <td class="percent-display">{{ formatPercent(testingTotal) }}</td>
            <td class="desc-cell">用于支付给外单位的检验、测试、化验及加工等费用</td>
          </tr>
          <tr>
            <td>8</td>
            <td class="indent-2">2.3 燃料动力费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(fuelTotal) }}</td>
            <td class="percent-display">{{ formatPercent(fuelTotal) }}</td>
            <td class="desc-cell"></td>
          </tr>
          <tr :class="{ 'warning-row': travelMeetingExceeded }">
            <td>9</td>
            <td class="indent-2">2.4 差旅费</td>
            <td class="ratio-cell" rowspan="3">合计≤10%</td>
            <td class="amount-display">{{ formatNumber(travelTotal) }}</td>
            <td class="percent-display">{{ formatPercent(travelTotal) }}</td>
            <td class="desc-cell" rowspan="3">
              差旅费+会议费+国际合作交流费合计一般不超过10%
              <span v-if="travelMeetingExceeded" class="warning-text">（当前合计{{ formatPercent(travelMeetingTotal) }}，超过10%限制）</span>
              <span v-else class="success-text">（当前合计{{ formatPercent(travelMeetingTotal) }}）</span>
            </td>
          </tr>
          <tr :class="{ 'warning-row': travelMeetingExceeded }">
            <td>10</td>
            <td class="indent-2">2.5 会议费</td>
            <!-- 比例列被rowspan占用 -->
            <td class="amount-display">{{ formatNumber(meetingTotal) }}</td>
            <td class="percent-display">{{ formatPercent(meetingTotal) }}</td>
            <!-- 说明列被rowspan占用 -->
          </tr>
          <tr :class="{ 'warning-row': travelMeetingExceeded }">
            <td>11</td>
            <td class="indent-2">2.6 国际合作与交流费</td>
            <!-- 比例列被rowspan占用 -->
            <td class="amount-display">{{ formatNumber(internationalTotal) }}</td>
            <td class="percent-display">{{ formatPercent(internationalTotal) }}</td>
            <!-- 说明列被rowspan占用 -->
          </tr>
          <tr>
            <td>12</td>
            <td class="indent-2">2.7 出版文献、信息传播和知识产权事务费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(publicationTotal) }}</td>
            <td class="percent-display">{{ formatPercent(publicationTotal) }}</td>
            <td class="desc-cell">对照课题成果数量计算</td>
          </tr>
          <tr>
            <td>13</td>
            <td class="indent-2">2.8 其他费用</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(otherTotal) }}</td>
            <td class="percent-display">{{ formatPercent(otherTotal) }}</td>
            <td class="desc-cell">审计费，按6‰计算</td>
          </tr>
          <tr :class="{ 'warning-row': laborExceeded }">
            <td>14</td>
            <td class="indent-1">3.劳务费</td>
            <td class="ratio-cell">≤35%</td>
            <td class="amount-display">{{ formatNumber(laborTotal) }}</td>
            <td class="percent-display">{{ formatPercent(laborTotal) }}</td>
            <td class="desc-cell">
              国拨劳务费比例不超35%，参照行业平均工资标准
              <span v-if="laborExceeded" class="warning-text">（当前{{ formatPercent(laborTotal) }}，超过35%限制）</span>
              <span v-else class="success-text">（符合要求）</span>
            </td>
          </tr>
          <tr>
            <td>15</td>
            <td class="indent-2">3.1 人员劳务费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(personnelTotal) }}</td>
            <td class="percent-display">{{ formatPercent(personnelTotal) }}</td>
            <td class="desc-cell">项目实施过程中支付给参与项目的研究生、博士后、访问学者和项目聘用的研究人员、科研辅助人员等的劳务性费用。</td>
          </tr>
          <tr>
            <td>16</td>
            <td class="indent-2">3.2 专家咨询费</td>
            <td></td>
            <td class="amount-display">{{ formatNumber(expertTotal) }}</td>
            <td class="percent-display">{{ formatPercent(expertTotal) }}</td>
            <td class="desc-cell">专家咨询费参照财政部关于印发《中央财政科研项目专家咨询费管理办法》的通知 财科教（2017）128号</td>
          </tr>
          <tr>
            <td>17</td>
            <td>（二）间接费用</td>
            <td class="ratio-cell">按实际计算</td>
            <td class="amount-display">{{ formatNumber(indirectTotal) }}</td>
            <td class="percent-display">{{ formatPercent(indirectTotal) }}</td>
            <td class="desc-cell">间接费用的计算基数是国拨经费中直接费用扣除设备购置费后的金额（一）500万元及以下部分为30%；（二）超过500万元至1000万元的部分为25%；（三）超过1000万元以上的部分为20%</td>
          </tr>
          <tr class="total-row">
            <td></td>
            <td><strong>合计</strong></td>
            <td></td>
            <td class="amount-display"><strong>{{ formatNumber(grandTotal) }}</strong></td>
            <td class="percent-display"><strong>100.00%</strong></td>
            <td></td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  budgetData: {
    type: Object,
    required: true
  }
})

const showCalcModal = ref(false)
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
</script>

<style scoped>
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
</style>
