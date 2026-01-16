<script setup>
import { ref, reactive, watch, computed, onMounted } from 'vue'
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
    totalWan: 0
  },
  meeting: {
    meetings: [],
    totalWan: 0
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
    expertTotalWan: 0
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

const totalExcludePrint = computed(() => {
  const equipmentTotal = (budgetData.equipment.purchaseTotalWan || 0) + (budgetData.equipment.trialTotalWan || 0)
  const materialTotal = budgetData.material.totalWan || 0
  const testingTotal = budgetData.testing.totalWan || 0
  const fuelTotal = budgetData.fuel.totalWan || 0
  const travelTotal = budgetData.travel.totalWan || 0
  const meetingTotal = budgetData.meeting.totalWan || 0
  const internationalTotal = budgetData.international.totalWan || 0
  const otherTotal = budgetData.other.totalWan || 0
  const laborTotal = (budgetData.labor.personnelTotalWan || 0) + (budgetData.labor.expertTotalWan || 0)
  const indirectTotal = budgetData.indirect.totalWan || 0
  
  return equipmentTotal + materialTotal + testingTotal + fuelTotal + travelTotal + 
         meetingTotal + internationalTotal + otherTotal + laborTotal + indirectTotal
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
      totalWan: 0
    })
    Object.assign(budgetData.meeting, {
      meetings: [{ name: '', times: 1, days: 1, attendees: 1, standard: 0.055 }],
      totalWan: 0
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
      experts: [{ type: '', count: 1, unitPrice: 0, description: '' }],
      personnelUnit: 'wan',
      expertUnit: 'wan',
      personnelTotalWan: 0,
      expertTotalWan: 0
    })
    Object.assign(budgetData.indirect, {
      amount: 0,
      unit: 'wan',
      description: '',
      totalWan: 0
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
        📤 导出数据
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
      <IndirectFee v-else-if="activeTab === 'indirect'" v-model="budgetData.indirect" />
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
