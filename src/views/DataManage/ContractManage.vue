<template>
  <div class="container-box">
    <div class="select-box">
      <span>筛选：</span>
      <el-select v-model="selectedItem" placeholder="请选择统计类型" style="width: 240px; margin-left: 20px" @change="clearSelectedItem()">
        <el-option v-for="item in selectCondition" :key="item.value" :label="item.label" :value="item.value" />
      </el-select>
      <el-date-picker
        v-model="checkedValue"
        :type="selectedItem"
        range-separator="至"
        start-placeholder="开始时间"
        end-placeholder="结束时间"
        placeholder="请选择统计时间"
        v-if="selectedItem === 'monthrange' || selectedItem === 'daterange'"
        style="margin-left: 20px"
        unlink-panels
        :disabled-date="disableFutureDates"
      />
      <div v-if="selectedItem === 'week'">
        <el-date-picker
          v-model="startWeekValue"
          :type="selectedItem"
          format="[第] ww [周]"
          placeholder="请选择开始周"
          style="margin-left: 20px"
          :disabled-date="disableFutureDates"
        />
        <span style="margin-left: 20px">至</span>
        <el-date-picker
          v-model="endWeekValue"
          :type="selectedItem"
          format="[第] ww [周]"
          placeholder="请选择结束周"
          style="margin-left: 20px"
          :disabled-date="disableEndWeekDates"
        />
      </div>
      <el-button type="primary" @click="getStatisData" style="margin-left: 20px">搜索</el-button>
    </div>
    <div style="height: 80vh; width: 100%">
      <ContractDataChart :data="tradeData" />
    </div>
  </div>
</template>
<script setup lang="ts">
import { onMounted, ref } from 'vue'
import { ContractApi } from '@/api/modules/contract'
import moment from 'moment'
import { getFormattedDateRange } from '@/hooks/useMergeTime'
import { ElMessage } from 'element-plus'
import ContractDataChart from './components/ContractDataChart.vue'

// 1. 同步子组件的 TradeArray 接口：包含 amountList（合同总金额）
interface TradeArray {
  timeList: string[]
  countList: number[] // 合同数量
  amountList: number[] // 合同总金额（新增）
}

// 2. 响应式变量初始化（补充 amountList 初始值）
const selectedItem = ref('day')
const checkedValue = ref<Array<Date>>([])
const startWeekValue = ref<Date>()
const endWeekValue = ref<Date>()
const selectCondition = ref([
  { label: '日合同数据统计', value: 'day' },
  { label: '周合同数据统计', value: 'week' },
  { label: '月合同数据统计', value: 'monthrange' },
  { label: '自定义时间合同数据统计', value: 'daterange' }
])

// 初始值补充 amountList: []（确保类型匹配）
const tradeData = ref<TradeArray>({
  timeList: [],
  countList: [],
  amountList: []
})

// 接口响应类型：补充 amountList
interface TradeResponse extends IResponse {
  data: TradeArray
}

// 请求参数类型（不变）
interface TradeParams {
  transactionType: string
  timeRange?: [string, string]
}

// 初始化数据（补充 amountList 处理）
const initData = async () => {
  try {
    const res = (await ContractApi.trendData({ transactionType: selectedItem.value })) as TradeResponse
    // 双重兜底：确保三个数组都存在
    const resData = res.data || { timeList: [], countList: [], amountList: [] }
    tradeData.value = {
      timeList: Array.isArray(resData.timeList) ? resData.timeList : [],
      countList: Array.isArray(resData.countList) ? resData.countList : [],
      amountList: Array.isArray(resData.amountList) ? resData.amountList : [] // 新增 amountList 处理
    }
  } catch (error) {
    console.error('获取客户统计数据失败:', error)
    tradeData.value = { timeList: [], countList: [], amountList: [] } // 错误时兜底
  }
}

onMounted(() => {
  initData()
})

// 禁用未来日期（类型优化）
const disableFutureDates = (time: Date): boolean => {
  const now = Date.now()
  const endOfToday = new Date(now)
  endOfToday.setHours(23, 59, 59, 999)
  return time.getTime() > endOfToday.getTime()
}

// 禁用结束周（类型优化）
const disableEndWeekDates = (date: Date): boolean => {
  if (!startWeekValue.value) {
    return date > new Date()
  }
  const start = new Date(startWeekValue.value)
  return date < start || date > new Date()
}

// 清空时间选择器
const clearSelectedItem = () => {
  checkedValue.value = []
  startWeekValue.value = undefined
  endWeekValue.value = undefined
}

// 搜索统计数据（补充 amountList 处理）
const getStatisData = async () => {
  try {
    let param: TradeParams = {
      transactionType: selectedItem.value
    }

    // 校验时间选择
    if (
      (checkedValue.value.length === 0 && (selectedItem.value === 'daterange' || selectedItem.value === 'monthrange')) ||
      (startWeekValue.value === undefined && endWeekValue.value === undefined && selectedItem.value === 'week')
    ) {
      ElMessage({ type: 'warning', message: '请选择有效的时间范围' })
      return
    }

    // 处理时间参数
    if (selectedItem.value === 'monthrange' && checkedValue.value.length === 2) {
      const [start, end] = checkedValue.value
      const startMonth = moment(start).format('YYYY-MM-01 00:00:00')
      const endMonth = moment(end).endOf('month').format('YYYY-MM-DD 23:59:59')
      param.timeRange = [startMonth, endMonth]
    } else if (selectedItem.value === 'daterange' && checkedValue.value.length === 2) {
      const [start, end] = checkedValue.value
      const startDay = moment(start).format('YYYY-MM-DD 00:00:00')
      const endDay = moment(end).format('YYYY-MM-DD 23:59:59')
      param.timeRange = [startDay, endDay]
    } else if (selectedItem.value === 'week') {
      if (startWeekValue.value && endWeekValue.value) {
        const [startWeek] = getFormattedDateRange(startWeekValue.value)
        const [, endWeek] = getFormattedDateRange(endWeekValue.value)
        param.timeRange = [startWeek, endWeek]
      } else {
        ElMessage({ type: 'warning', message: '请选择完整的周时间范围' })
        return
      }
    }

    // 请求数据并处理（补充 amountList）
    const res = (await ContractApi.trendData(param)) as TradeResponse
    const resData = res.data || { timeList: [], countList: [], amountList: [] }
    tradeData.value = {
      timeList: Array.isArray(resData.timeList) ? resData.timeList : [],
      countList: Array.isArray(resData.countList) ? resData.countList : [],
      amountList: Array.isArray(resData.amountList) ? resData.amountList : [] // 新增 amountList 处理
    }
  } catch (error) {
    console.error('获取交易统计数据失败:', error)
    tradeData.value = { timeList: [], countList: [], amountList: [] } // 错误时兜底
  }
}
</script>
<style scoped>
.container-box {
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: flex-start;
  width: 100%;
  height: 100vh;
  padding: 20px;
  box-sizing: border-box;
}

.select-box {
  display: flex;
  justify-content: flex-start;
  align-items: center;
  width: 100%;
  margin-bottom: 20px;
  flex-wrap: wrap;
  gap: 15px;
}
</style>