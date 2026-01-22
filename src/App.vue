<script setup>
import { ref, reactive, watch, computed, onMounted } from 'vue'
import * as XLSX from 'xlsx'
import BudgetSummary from './components/BudgetSummary.vue'
import EquipmentFee from './components/EquipmentFee.vue'
import MaterialFee from './components/MaterialFee.vue'
import TestingFee from './components/TestingFee.vue'
import FuelFee from './components/FuelFee.vue'
import TravelFee from './components/TravelFee.vue'
import MeetingFee from './components/MeetingFee.vue'
import InternationalFee from './components/InternationalFee.vue'
import PublicationFee from './components/PublicationFee.vue'
import OtherFee from './components/OtherFee.vue'
import LaborFee from './components/LaborFee.vue'
import IndirectFee from './components/IndirectFee.vue'

const STORAGE_KEY = 'budget_data_cache'
const ACTIVE_TAB_KEY = 'budget_active_tab'

const activeTab = ref('summary')

const tabs = [
  { key: 'summary', label: '经费测算表' },
  { key: 'equipment', label: '设备费' },
  { key: 'material', label: '材料费' },
  { key: 'testing', label: '测试化验加工费' },
  { key: 'fuel', label: '燃料动力费' },
  { key: 'travel', label: '差旅费' },
  { key: 'meeting', label: '会议费' },
  { key: 'international', label: '国际合作与交流费' },
  { key: 'publication', label: '出版文献费' },
  { key: 'other', label: '其他费用' },
  { key: 'labor', label: '劳务费' },
  { key: 'indirect', label: '间接费用' }
]

const budgetData = reactive({
  otherFunding: {
    equipment: 0,
    purchaseEquipment: 0,
    trialEquipment: 0,
    material: 0,
    testing: 0,
    fuel: 0,
    travel: 0,
    meeting: 0,
    international: 0,
    publication: 0,
    other: 0,
    personnel: 0,
    expert: 0,
    indirect: 0
  },
  equipment: {
    purchaseEquipments: [],
    trialEquipments: [],
    purchaseUnit: 'wan',
    trialUnit: 'wan',
    purchaseTotalWan: 0,
    trialTotalWan: 0
  },
  material: {
    materials: [],
    unit: 'wan',
    totalWan: 0
  },
  testing: {
    testings: [],
    unit: 'wan',
    totalWan: 0
  },
  fuel: {
    amount: 0,
    unit: 'wan',
    description: '',
    totalWan: 0
  },
  travel: {
    travels: [],
    totalWan: 0,
    plannerConfig: {
      targetAmount: 5,
      tolerancePercent: 15,
      cityConfigs: []
    }
  },
  meeting: {
    meetings: [],
    totalWan: 0,
    plannerConfig: {
      targetAmount: 2,
      tolerancePercent: 15,
      meetingConfigs: []
    }
  },
  international: {
    exchanges: [],
    unit: 'wan',
    totalWan: 0
  },
  publication: {
    publications: [],
    totalWan: 0
  },
  other: {
    others: [],
    unit: 'wan',
    totalWan: 0
  },
  labor: {
    personnels: [],
    experts: [],
    personnelUnit: 'wan',
    expertUnit: 'wan',
    personnelTotalWan: 0,
    expertTotalWan: 0,
    plannerConfig: {
      targetAmount: 10,
      tolerancePercent: 15,
      personnelConfigs: []
    },
    expertPlannerConfig: {
      targetAmount: 2,
      tolerancePercent: 15,
      expertConfigs: []
    }
  },
  indirect: {
    amount: 0,
    unit: 'wan',
    description: '',
    totalWan: 0
  }
})

const fileInput = ref(null)

const saveToLocalStorage = () => {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(budgetData))
    localStorage.setItem(ACTIVE_TAB_KEY, activeTab.value)
  } catch (err) {
    console.error('保存到本地缓存失败:', err)
  }
}

const loadFromLocalStorage = () => {
  try {
    const savedData = localStorage.getItem(STORAGE_KEY)
    if (savedData) {
      const data = JSON.parse(savedData)
      Object.keys(data).forEach(key => {
        if (budgetData[key] !== undefined) {
          Object.assign(budgetData[key], data[key])
        }
      })
    }
    const savedTab = localStorage.getItem(ACTIVE_TAB_KEY)
    if (savedTab && tabs.some(t => t.key === savedTab)) {
      activeTab.value = savedTab
    }
  } catch (err) {
    console.error('从本地缓存加载失败:', err)
  }
}

watch(budgetData, saveToLocalStorage, { deep: true })
watch(activeTab, saveToLocalStorage)

onMounted(() => {
  loadFromLocalStorage()
})

const purchaseEquipmentTotal = computed(() => budgetData.equipment.purchaseTotalWan || 0)

const directTotal = computed(() => {
  const equipmentTotal = (budgetData.equipment.purchaseTotalWan || 0) + (budgetData.equipment.trialTotalWan || 0)
  const materialTotal = budgetData.material.totalWan || 0
  const testingTotal = budgetData.testing.totalWan || 0
  const fuelTotal = budgetData.fuel.totalWan || 0
  const travelTotal = budgetData.travel.totalWan || 0
  const meetingTotal = budgetData.meeting.totalWan || 0
  const internationalTotal = budgetData.international.totalWan || 0
  const publicationTotal = budgetData.publication.totalWan || 0
  const otherTotal = budgetData.other.totalWan || 0
  const laborTotal = (budgetData.labor.personnelTotalWan || 0) + (budgetData.labor.expertTotalWan || 0)
  
  return equipmentTotal + materialTotal + testingTotal + fuelTotal + travelTotal + 
         meetingTotal + internationalTotal + publicationTotal + otherTotal + laborTotal
})

const totalExcludePrint = computed(() => {
  const indirectTotal = budgetData.indirect.totalWan || 0
  return directTotal.value + indirectTotal
})

const exportData = () => {
  const dataStr = JSON.stringify(budgetData, null, 2)
  const blob = new Blob([dataStr], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  link.download = `项目预算申报书_${new Date().toLocaleDateString('zh-CN').replace(/\//g, '-')}.json`
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)
  URL.revokeObjectURL(url)
}

const exportToExcel = () => {
  const wb = XLSX.utils.book_new()
  
  // 计算其他资金来源合计
  const of = budgetData.otherFunding || {}
  const otherEquipment = (of.purchaseEquipment || 0) + (of.trialEquipment || 0)
  const otherBusiness = (of.material || 0) + (of.testing || 0) + (of.fuel || 0) + 
                        (of.travel || 0) + (of.meeting || 0) + (of.international || 0) + 
                        (of.publication || 0) + (of.other || 0)
  const otherLabor = (of.personnel || 0) + (of.expert || 0)
  const otherDirect = otherEquipment + otherBusiness + otherLabor
  const otherTotal = otherDirect + (of.indirect || 0)
  const combinedTotal = totalExcludePrint.value + otherTotal

  // 计算业务费合计（国拨）
  const businessTotal = (budgetData.material.totalWan || 0) + (budgetData.testing.totalWan || 0) + 
                        (budgetData.fuel.totalWan || 0) + (budgetData.travel.totalWan || 0) + 
                        (budgetData.meeting.totalWan || 0) + (budgetData.international.totalWan || 0) + 
                        (budgetData.publication.totalWan || 0) + (budgetData.other.totalWan || 0)

  // 1. 经费测算汇总表
  const summaryData = [
    ['经费测算汇总表'],
    ['序号', '科目名称', '国拨(万)', '其他来源(万)', '合计(万)'],
    ['1', '（一）直接费用', directTotal.value.toFixed(2), otherDirect.toFixed(2), (directTotal.value + otherDirect).toFixed(2)],
    ['2', '1.设备费', ((budgetData.equipment.purchaseTotalWan || 0) + (budgetData.equipment.trialTotalWan || 0)).toFixed(2), otherEquipment.toFixed(2), ((budgetData.equipment.purchaseTotalWan || 0) + (budgetData.equipment.trialTotalWan || 0) + otherEquipment).toFixed(2)],
    ['3', '  1.1 购置设备费', (budgetData.equipment.purchaseTotalWan || 0).toFixed(2), (of.purchaseEquipment || 0).toFixed(2), ((budgetData.equipment.purchaseTotalWan || 0) + (of.purchaseEquipment || 0)).toFixed(2)],
    ['4', '  1.2 试制设备费', (budgetData.equipment.trialTotalWan || 0).toFixed(2), (of.trialEquipment || 0).toFixed(2), ((budgetData.equipment.trialTotalWan || 0) + (of.trialEquipment || 0)).toFixed(2)],
    ['5', '2.业务费', businessTotal.toFixed(2), otherBusiness.toFixed(2), (businessTotal + otherBusiness).toFixed(2)],
    ['6', '  2.1 材料费', (budgetData.material.totalWan || 0).toFixed(2), (of.material || 0).toFixed(2), ((budgetData.material.totalWan || 0) + (of.material || 0)).toFixed(2)],
    ['7', '  2.2 测试化验加工费', (budgetData.testing.totalWan || 0).toFixed(2), (of.testing || 0).toFixed(2), ((budgetData.testing.totalWan || 0) + (of.testing || 0)).toFixed(2)],
    ['8', '  2.3 燃料动力费', (budgetData.fuel.totalWan || 0).toFixed(2), (of.fuel || 0).toFixed(2), ((budgetData.fuel.totalWan || 0) + (of.fuel || 0)).toFixed(2)],
    ['9', '  2.4 差旅费', (budgetData.travel.totalWan || 0).toFixed(2), (of.travel || 0).toFixed(2), ((budgetData.travel.totalWan || 0) + (of.travel || 0)).toFixed(2)],
    ['10', '  2.5 会议费', (budgetData.meeting.totalWan || 0).toFixed(2), (of.meeting || 0).toFixed(2), ((budgetData.meeting.totalWan || 0) + (of.meeting || 0)).toFixed(2)],
    ['11', '  2.6 国际合作与交流费', (budgetData.international.totalWan || 0).toFixed(2), (of.international || 0).toFixed(2), ((budgetData.international.totalWan || 0) + (of.international || 0)).toFixed(2)],
    ['12', '  2.7 出版文献费', (budgetData.publication.totalWan || 0).toFixed(2), (of.publication || 0).toFixed(2), ((budgetData.publication.totalWan || 0) + (of.publication || 0)).toFixed(2)],
    ['13', '  2.8 其他费用', (budgetData.other.totalWan || 0).toFixed(2), (of.other || 0).toFixed(2), ((budgetData.other.totalWan || 0) + (of.other || 0)).toFixed(2)],
    ['14', '3.劳务费', ((budgetData.labor.personnelTotalWan || 0) + (budgetData.labor.expertTotalWan || 0)).toFixed(2), otherLabor.toFixed(2), ((budgetData.labor.personnelTotalWan || 0) + (budgetData.labor.expertTotalWan || 0) + otherLabor).toFixed(2)],
    ['15', '  3.1 人员劳务费', (budgetData.labor.personnelTotalWan || 0).toFixed(2), (of.personnel || 0).toFixed(2), ((budgetData.labor.personnelTotalWan || 0) + (of.personnel || 0)).toFixed(2)],
    ['16', '  3.2 专家咨询费', (budgetData.labor.expertTotalWan || 0).toFixed(2), (of.expert || 0).toFixed(2), ((budgetData.labor.expertTotalWan || 0) + (of.expert || 0)).toFixed(2)],
    ['17', '（二）间接费用', (budgetData.indirect.totalWan || 0).toFixed(2), (of.indirect || 0).toFixed(2), ((budgetData.indirect.totalWan || 0) + (of.indirect || 0)).toFixed(2)],
    ['', '合计', totalExcludePrint.value.toFixed(2), otherTotal.toFixed(2), combinedTotal.toFixed(2)]
  ]
  const wsSummary = XLSX.utils.aoa_to_sheet(summaryData)
  wsSummary['!cols'] = [{ wch: 8 }, { wch: 20 }, { wch: 12 }, { wch: 14 }, { wch: 12 }]
  XLSX.utils.book_append_sheet(wb, wsSummary, '经费汇总')
  
  // 2. 设备费-购置设备
  if (budgetData.equipment.purchaseEquipments?.length > 0) {
    const purchaseData = [
      ['购置设备明细'],
      ['序号', '设备名称', '型号规格', '单价', '数量', '金额', '用途', '供应商'],
      ...budgetData.equipment.purchaseEquipments.map((item, i) => [
        i + 1,
        item.name || '',
        item.model || '',
        item.unitPrice || 0,
        item.quantity || 0,
        ((item.unitPrice || 0) * (item.quantity || 0)).toFixed(2),
        item.purpose || '',
        item.supplier || ''
      ]),
      ['', '', '', '', '合计', (budgetData.equipment.purchaseTotalWan || 0).toFixed(2), '', '']
    ]
    const wsPurchase = XLSX.utils.aoa_to_sheet(purchaseData)
    XLSX.utils.book_append_sheet(wb, wsPurchase, '购置设备')
  }
  
  // 3. 设备费-试制设备
  if (budgetData.equipment.trialEquipments?.length > 0) {
    const trialData = [
      ['试制设备明细'],
      ['序号', '设备名称', '型号规格', '单价', '数量', '金额', '用途', '供应商'],
      ...budgetData.equipment.trialEquipments.map((item, i) => [
        i + 1,
        item.name || '',
        item.model || '',
        item.unitPrice || 0,
        item.quantity || 0,
        ((item.unitPrice || 0) * (item.quantity || 0)).toFixed(2),
        item.purpose || '',
        item.supplier || ''
      ]),
      ['', '', '', '', '合计', (budgetData.equipment.trialTotalWan || 0).toFixed(2), '', '']
    ]
    const wsTrial = XLSX.utils.aoa_to_sheet(trialData)
    XLSX.utils.book_append_sheet(wb, wsTrial, '试制设备')
  }
  
  // 4. 材料费
  if (budgetData.material.materials?.length > 0) {
    const materialData = [
      ['材料费明细'],
      ['序号', '材料名称', '单价', '数量', '金额', '测算依据'],
      ...budgetData.material.materials.map((item, i) => [
        i + 1,
        item.name || '',
        item.unitPrice || 0,
        item.quantity || 0,
        ((item.unitPrice || 0) * (item.quantity || 0)).toFixed(2),
        item.basis || ''
      ]),
      ['', '', '', '合计', (budgetData.material.totalWan || 0).toFixed(2), '']
    ]
    const wsMaterial = XLSX.utils.aoa_to_sheet(materialData)
    XLSX.utils.book_append_sheet(wb, wsMaterial, '材料费')
  }
  
  // 5. 测试化验加工费
  if (budgetData.testing.testings?.length > 0) {
    const testingData = [
      ['测试化验加工费明细'],
      ['序号', '测试内容', '测试类别', '委托类型', '金额'],
      ...budgetData.testing.testings.map((item, i) => [
        i + 1,
        item.content || '',
        item.category || '',
        item.delegateType || '',
        item.amount || 0
      ]),
      ['', '', '', '合计', (budgetData.testing.totalWan || 0).toFixed(2)]
    ]
    const wsTesting = XLSX.utils.aoa_to_sheet(testingData)
    XLSX.utils.book_append_sheet(wb, wsTesting, '测试化验加工费')
  }
  
  // 6. 燃料动力费
  const fuelData = [
    ['燃料动力费'],
    ['金额(万元)', '说明'],
    [(budgetData.fuel.totalWan || 0).toFixed(2), budgetData.fuel.description || '']
  ]
  const wsFuel = XLSX.utils.aoa_to_sheet(fuelData)
  XLSX.utils.book_append_sheet(wb, wsFuel, '燃料动力费')
  
  // 7. 差旅费
  if (budgetData.travel.travels?.length > 0) {
    // 获取差旅费计算公式配置
    const travelFormula = budgetData.travel.formula || {
      transportMultiplier: 1,
      accommodationDays: 'days-1',
      foodDays: 'days',
      localTransportDays: 'days'
    }
    const getDaysValue = (days, mode) => mode === 'days-1' ? Math.max(0, days - 1) : days
    const calcTravelAmount = (item) => {
      const days = item.days || 1
      const transportCost = (item.transport || 0) * travelFormula.transportMultiplier
      const accCost = (item.accommodation || 0) * getDaysValue(days, travelFormula.accommodationDays)
      const foodCost = (item.food || 0) * getDaysValue(days, travelFormula.foodDays)
      const localCost = (item.localTransport || 0) * getDaysValue(days, travelFormula.localTransportDays)
      const tripCost = transportCost + accCost + foodCost + localCost
      return (tripCost * (item.people || 1) * (item.times || 1) / 10000).toFixed(4)
    }
    const travelData = [
      ['差旅费明细'],
      ['序号', '出差地点', '出差事由', '交通费', '住宿费', '伙食费', '市内交通', '天数', '人数', '次数', '金额(万)'],
      ...budgetData.travel.travels.map((item, i) => [
        i + 1,
        item.city || '',
        item.purpose || '',
        item.transport || 0,
        item.accommodation || 0,
        item.food || 0,
        item.localTransport || 0,
        item.days || 0,
        item.people || 0,
        item.times || 0,
        calcTravelAmount(item)
      ]),
      ['', '', '', '', '', '', '', '', '', '合计', (budgetData.travel.totalWan || 0).toFixed(2)]
    ]
    const wsTravel = XLSX.utils.aoa_to_sheet(travelData)
    XLSX.utils.book_append_sheet(wb, wsTravel, '差旅费')
  }
  
  // 8. 会议费
  if (budgetData.meeting.meetings?.length > 0) {
    const meetingData = [
      ['会议费明细'],
      ['序号', '会议名称', '次数', '天数', '人数', '标准(万)', '金额'],
      ...budgetData.meeting.meetings.map((item, i) => [
        i + 1,
        item.name || '',
        item.times || 0,
        item.days || 0,
        item.attendees || 0,
        item.standard || 0,
        ((item.times || 0) * (item.days || 0) * (item.attendees || 0) * (item.standard || 0)).toFixed(2)
      ]),
      ['', '', '', '', '', '合计', (budgetData.meeting.totalWan || 0).toFixed(2)]
    ]
    const wsMeeting = XLSX.utils.aoa_to_sheet(meetingData)
    XLSX.utils.book_append_sheet(wb, wsMeeting, '会议费')
  }
  
  // 9. 国际合作与交流费
  if (budgetData.international.exchanges?.length > 0) {
    const intlData = [
      ['国际合作与交流费明细'],
      ['序号', '交流内容', '国家/地区', '人数', '天数', '金额', '说明'],
      ...budgetData.international.exchanges.map((item, i) => [
        i + 1,
        item.content || '',
        item.country || '',
        item.people || 0,
        item.days || 0,
        item.amount || 0,
        item.description || ''
      ]),
      ['', '', '', '', '合计', (budgetData.international.totalWan || 0).toFixed(2), '']
    ]
    const wsIntl = XLSX.utils.aoa_to_sheet(intlData)
    XLSX.utils.book_append_sheet(wb, wsIntl, '国际合作与交流费')
  }
  
  // 10. 出版文献费
  if (budgetData.publication.publications?.length > 0) {
    const pubData = [
      ['出版文献费明细'],
      ['序号', '类型', '单价(元)', '数量', '金额(万)'],
      ...budgetData.publication.publications.map((item, i) => [
        i + 1,
        item.type || '',
        item.unitPrice || 0,
        item.quantity || 0,
        ((item.unitPrice || 0) * (item.quantity || 0) / 10000).toFixed(4)
      ]),
      ['', '', '', '合计', (budgetData.publication.totalWan || 0).toFixed(2)]
    ]
    const wsPub = XLSX.utils.aoa_to_sheet(pubData)
    XLSX.utils.book_append_sheet(wb, wsPub, '出版文献费')
  }
  
  // 11. 其他费用
  if (budgetData.other.others?.length > 0) {
    const otherData = [
      ['其他费用明细'],
      ['序号', '费用名称', '金额', '说明'],
      ...budgetData.other.others.map((item, i) => [
        i + 1,
        item.name || '',
        item.amount || 0,
        item.description || ''
      ]),
      ['', '', (budgetData.other.totalWan || 0).toFixed(2), '']
    ]
    const wsOther = XLSX.utils.aoa_to_sheet(otherData)
    XLSX.utils.book_append_sheet(wb, wsOther, '其他费用')
  }
  
  // 12. 劳务费-人员劳务费
  if (budgetData.labor.personnels?.length > 0) {
    const personnelData = [
      ['人员劳务费明细'],
      ['序号', '人员类型', '人数', '月数', '月均费用(万)', '金额(万)', '说明'],
      ...budgetData.labor.personnels.map((item, i) => [
        i + 1,
        item.type === '自定义' ? item.customType : item.type || '',
        item.count || 0,
        item.months || 0,
        item.monthlyCost || 0,
        ((item.count || 0) * (item.months || 0) * (item.monthlyCost || 0)).toFixed(2),
        item.description || ''
      ]),
      ['', '', '', '', '合计', (budgetData.labor.personnelTotalWan || 0).toFixed(2), '']
    ]
    const wsPersonnel = XLSX.utils.aoa_to_sheet(personnelData)
    XLSX.utils.book_append_sheet(wb, wsPersonnel, '人员劳务费')
  }
  
  // 13. 劳务费-专家咨询费
  if (budgetData.labor.experts?.length > 0) {
    const expertData = [
      ['专家咨询费明细'],
      ['序号', '会议内容', '人数', '会期(天)', '次数', '人均标准(元)', '金额(万)'],
      ...budgetData.labor.experts.map((item, i) => [
        i + 1,
        item.name || '',
        item.people || 0,
        item.days || 0,
        item.times || 0,
        item.standard || 0,
        ((item.people || 0) * (item.days || 0) * (item.times || 0) * (item.standard || 0) / 10000).toFixed(4)
      ]),
      ['', '', '', '', '', '合计', (budgetData.labor.expertTotalWan || 0).toFixed(2)]
    ]
    const wsExpert = XLSX.utils.aoa_to_sheet(expertData)
    XLSX.utils.book_append_sheet(wb, wsExpert, '专家咨询费')
  }
  
  // 14. 间接费用
  const indirectData = [
    ['间接费用'],
    ['金额(万元)', '说明'],
    [(budgetData.indirect.totalWan || 0).toFixed(2), budgetData.indirect.description || '']
  ]
  const wsIndirect = XLSX.utils.aoa_to_sheet(indirectData)
  XLSX.utils.book_append_sheet(wb, wsIndirect, '间接费用')
  
  // 导出文件
  const fileName = `项目预算申报书_${new Date().toLocaleDateString('zh-CN').replace(/\//g, '-')}.xlsx`
  XLSX.writeFile(wb, fileName)
}

const triggerImport = () => {
  fileInput.value?.click()
}

const importData = (event) => {
  const file = event.target.files?.[0]
  if (!file) return
  
  const reader = new FileReader()
  reader.onload = (e) => {
    try {
      const data = JSON.parse(e.target.result)
      Object.keys(data).forEach(key => {
        if (budgetData[key] !== undefined) {
          Object.assign(budgetData[key], data[key])
        }
      })
      saveToLocalStorage()
      alert('数据导入成功！')
    } catch (err) {
      alert('导入失败：文件格式不正确')
      console.error(err)
    }
  }
  reader.readAsText(file)
  event.target.value = ''
}

const clearData = () => {
  if (confirm('确定要清空所有数据吗？此操作不可恢复。')) {
    Object.assign(budgetData.equipment, {
      purchaseEquipments: [{ name: '', model: '', unitPrice: 0, quantity: 1, purpose: '', supplier: '' }],
      trialEquipments: [{ name: '', model: '', unitPrice: 0, quantity: 1, purpose: '', supplier: '' }],
      purchaseUnit: 'wan',
      trialUnit: 'wan',
      purchaseTotalWan: 0,
      trialTotalWan: 0
    })
    Object.assign(budgetData.material, {
      materials: [{ name: '', unitPrice: 0, quantity: 1, basis: '' }],
      unit: 'wan',
      totalWan: 0
    })
    Object.assign(budgetData.testing, {
      testings: [{ content: '', category: '', delegateType: '', amount: 0 }],
      unit: 'wan',
      totalWan: 0
    })
    Object.assign(budgetData.fuel, {
      amount: 0,
      unit: 'wan',
      description: '',
      totalWan: 0
    })
    Object.assign(budgetData.travel, {
      travels: [{ city: '', purpose: '', transport: 0, accommodation: 0, food: 0, localTransport: 80, days: 1, people: 1, times: 1 }],
      totalWan: 0,
      plannerConfig: {
        targetAmount: 5,
        tolerancePercent: 15,
        cityConfigs: []
      }
    })
    Object.assign(budgetData.meeting, {
      meetings: [{ name: '', times: 1, days: 1, attendees: 1, standard: 0.055 }],
      totalWan: 0,
      plannerConfig: {
        targetAmount: 2,
        tolerancePercent: 15,
        meetingConfigs: []
      }
    })
    Object.assign(budgetData.international, {
      exchanges: [{ content: '', country: '', people: 1, days: 1, amount: 0, description: '' }],
      unit: 'wan',
      totalWan: 0
    })
    Object.assign(budgetData.publication, {
      publications: [
        { type: '论文版面费', unitPrice: 20000, quantity: 0 },
        { type: '专利申请费', unitPrice: 8000, quantity: 0 },
        { type: '软件著作权', unitPrice: 5000, quantity: 0 },
        { type: '查新费与检索费', unitPrice: 2000, quantity: 0 },
        { type: '打印', unitPrice: 50, quantity: 0 },
        { type: '图书购买', unitPrice: 180, quantity: 0 }
      ],
      totalWan: 0
    })
    Object.assign(budgetData.other, {
      others: [{ name: '', amount: 0, description: '' }],
      unit: 'wan',
      totalWan: 0
    })
    Object.assign(budgetData.labor, {
      personnels: [{ type: '', count: 1, months: 1, monthlyCost: 0, description: '' }],
      experts: [{ name: '', people: 5, days: 1, times: 1, standard: 2800 }],
      personnelUnit: 'wan',
      expertUnit: 'wan',
      personnelTotalWan: 0,
      expertTotalWan: 0,
      plannerConfig: {
        targetAmount: 10,
        tolerancePercent: 15,
        personnelConfigs: []
      },
      expertPlannerConfig: {
        targetAmount: 2,
        tolerancePercent: 15,
        expertConfigs: []
      }
    })
    Object.assign(budgetData.indirect, {
      amount: 0,
      unit: 'wan',
      description: '',
      totalWan: 0
    })
    Object.assign(budgetData.otherFunding, {
      equipment: 0,
      purchaseEquipment: 0,
      trialEquipment: 0,
      material: 0,
      testing: 0,
      fuel: 0,
      travel: 0,
      meeting: 0,
      international: 0,
      publication: 0,
      other: 0,
      personnel: 0,
      expert: 0,
      indirect: 0
    })
    saveToLocalStorage()
  }
}
</script>

<template>
  <div class="app-container">
    <header class="app-header">
      <h1>项目预算申报书</h1>
      <p>经费测算与预算编制系统</p>
    </header>

    <div class="toolbar">
      <button class="btn btn-primary" @click="exportData">
        📤 导出 JSON
      </button>
      <button class="btn btn-excel" @click="exportToExcel">
        📊 导出 Excel
      </button>
      <button class="btn btn-success" @click="triggerImport">
        📥 导入数据
      </button>
      <button class="btn btn-danger" @click="clearData">
        🗑️ 清空数据
      </button>
      <input 
        ref="fileInput" 
        type="file" 
        accept=".json" 
        class="file-input" 
        @change="importData"
      >
    </div>

    <nav class="nav-tabs">
      <button 
        v-for="tab in tabs" 
        :key="tab.key"
        :class="['nav-tab', { active: activeTab === tab.key }]"
        @click="activeTab = tab.key"
      >
        {{ tab.label }}
      </button>
    </nav>

    <main class="main-content">
      <BudgetSummary v-if="activeTab === 'summary'" :budget-data="budgetData" />
      <EquipmentFee v-else-if="activeTab === 'equipment'" v-model="budgetData.equipment" />
      <MaterialFee v-else-if="activeTab === 'material'" v-model="budgetData.material" />
      <TestingFee v-else-if="activeTab === 'testing'" v-model="budgetData.testing" />
      <FuelFee v-else-if="activeTab === 'fuel'" v-model="budgetData.fuel" />
      <TravelFee v-else-if="activeTab === 'travel'" v-model="budgetData.travel" />
      <MeetingFee v-else-if="activeTab === 'meeting'" v-model="budgetData.meeting" />
      <InternationalFee v-else-if="activeTab === 'international'" v-model="budgetData.international" />
      <PublicationFee v-else-if="activeTab === 'publication'" v-model="budgetData.publication" :current-total-exclude-print="totalExcludePrint" />
      <OtherFee v-else-if="activeTab === 'other'" v-model="budgetData.other" />
      <LaborFee v-else-if="activeTab === 'labor'" v-model="budgetData.labor" />
      <IndirectFee v-else-if="activeTab === 'indirect'" v-model="budgetData.indirect" :direct-total="directTotal" :purchase-equipment-total="purchaseEquipmentTotal" />
    </main>

    <footer class="app-footer">
      <p>© 2026 项目预算申报系统 | 数据仅保存在本地浏览器</p>
    </footer>
  </div>
</template>

<style>
@import './styles/main.css';

.app-footer {
  margin-top: 30px;
  padding-top: 20px;
  border-top: 1px solid #eee;
  text-align: center;
  color: #888;
  font-size: 13px;
}

.main-content {
  min-height: 400px;
}
</style>
