<template>
  <div class="section">
    <h3 class="section-title">会议费</h3>
    
    <!-- 智能规划按钮 -->
    <div class="planner-trigger">
      <button class="btn btn-primary btn-small" @click="showPlanner = true">
        🧮 智能规划
      </button>
    </div>

    <!-- 智能规划组件 -->
    <MeetingSmartPlanner
      v-model:visible="showPlanner"
      v-model:plannerConfig="plannerConfig"
      :meetings="meetings"
      @select="handlePlannerSelect"
    />

    <div class="standard-info">
      <span>默认标准：</span>
      <strong>0.055 万元/人/天</strong>
      <span class="standard-hint">（可在表格中自定义修改）</span>
    </div>
    <div class="table-container">
      <table>
        <thead>
          <tr>
            <th style="width: 40px">序号</th>
            <th style="width: 150px">会议名称</th>
            <th style="width: 50px">次数</th>
            <th style="width: 50px">天数</th>
            <th style="width: 50px">人数</th>
            <th style="width: 80px">标准(万)</th>
            <th style="width: 80px">总计(万)</th>
            <th style="width: 50px">操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in meetings" :key="index">
            <td>{{ index + 1 }}</td>
            <td><input type="text" v-model="item.name" placeholder="名称"></td>
            <td><input type="number" v-model.number="item.times" min="1" step="1"></td>
            <td><input type="number" v-model.number="item.days" min="1" step="1"></td>
            <td><input type="number" v-model.number="item.attendees" min="1" step="1"></td>
            <td><input type="number" v-model.number="item.standard" min="0" step="0.001"></td>
            <td class="amount-display">{{ formatNumber(item.times * item.days * item.attendees * item.standard) }}</td>
            <td><button class="delete-btn" @click="removeRow(index)">删</button></td>
          </tr>
          <tr class="total-row">
            <td colspan="6" style="text-align: right">合计：</td>
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
import MeetingSmartPlanner from './MeetingSmartPlanner.vue'

const DEFAULT_STANDARD = 0.055

const props = defineProps({
  modelValue: {
    type: Object,
    default: () => ({
      meetings: [],
      plannerConfig: null
    })
  }
})

const emit = defineEmits(['update:modelValue'])

const showPlanner = ref(false)

const plannerConfig = ref(props.modelValue.plannerConfig || {
  targetAmount: 2,
  tolerancePercent: 15,
  meetingConfigs: []
})

const meetings = ref(props.modelValue.meetings?.length > 0 
  ? props.modelValue.meetings 
  : [{ name: '', times: 1, days: 1, attendees: 1, standard: DEFAULT_STANDARD }])

const formatNumber = (num) => {
  if (!num || isNaN(num)) return '0.00'
  return Number(num).toFixed(2)
}

const total = computed(() => {
  return meetings.value.reduce((sum, item) => {
    return sum + (item.times || 1) * (item.days || 1) * (item.attendees || 1) * (item.standard || 0)
  }, 0)
})

const handlePlannerSelect = (combo) => {
  for (const plan of combo.meetings) {
    const meeting = meetings.value.find(m => m.name === plan.name)
    if (meeting) {
      meeting.attendees = plan.attendees
      meeting.times = plan.times
    }
  }
}

const addRow = () => {
  meetings.value.push({ name: '', times: 1, days: 1, attendees: 1, standard: DEFAULT_STANDARD })
}

const removeRow = (index) => {
  if (meetings.value.length > 1) {
    meetings.value.splice(index, 1)
  }
}

watch([meetings, plannerConfig], () => {
  emit('update:modelValue', {
    meetings: meetings.value,
    totalWan: total.value,
    plannerConfig: plannerConfig.value
  })
}, { deep: true, immediate: true })

watch(() => props.modelValue, (newVal) => {
  if (newVal.meetings) meetings.value = newVal.meetings
  if (newVal.plannerConfig) plannerConfig.value = newVal.plannerConfig
}, { deep: true })
</script>

<style scoped>
.standard-info {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 10px;
  font-size: 13px;
  color: #555;
}

.standard-info strong {
  color: #2d3748;
}

.standard-hint {
  color: #888;
  font-size: 12px;
}

.planner-trigger {
  margin-bottom: 12px;
}
</style>
