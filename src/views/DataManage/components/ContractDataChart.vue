<template>
  <div id="contractPieChart" style="height: 100%; width: 100%"></div>
</template>

<script lang="ts" name="ContractDataChart" setup>
import * as echarts from 'echarts'
import { onMounted, ref, onUnmounted, PropType, watch } from 'vue'

// 扩展接口：包含合同数量和总金额两个维度的数据
interface TradeArray {
  timeList: string[] // 分类名称（如：2023年1月、2023年2月等）
  countList: (number | string | boolean)[] // 合同数量
  amountList: (number | string | boolean)[] // 合同总金额
}

// Props 定义（强化类型校验）
const props = defineProps({
  data: {
    type: Object as PropType<TradeArray>,
    required: true,
    // 严格校验：确保三个数组存在且长度匹配
    validator: (value: TradeArray): boolean => {
      return (
        Array.isArray(value.timeList) &&
        Array.isArray(value.countList) &&
        Array.isArray(value.amountList) &&
        true
      )
    },
    default: (): TradeArray => ({ timeList: [], countList: [], amountList: [] })
  }
})

const myChart = ref<echarts.ECharts | null>(null)

// 安全处理数据：统一格式化单个维度数据
const formatDimensionData = (
  dimensionList: (number | string | boolean)[]
): { name: string; value: number }[] => {
  const { timeList = [] } = props.data
  const validLength = Math.min(timeList.length, dimensionList.length)

  return Array.from({ length: validLength }, (_, index) => {
    const name = timeList[index] || `分类${index + 1}`
    const value = Number(dimensionList[index]) || 0 // 强制转为数字，无效值兜底0
    return { name, value }
  }).filter(item => item.value > 0) // 过滤无效数据
}

// 获取两个维度的饼图数据
const getPieData = (): {
  countData: { name: string; value: number }[]
  amountData: { name: string; value: number }[]
} => {
  const { countList = [], amountList = [] } = props.data
  return {
    countData: formatDimensionData(countList),
    amountData: formatDimensionData(amountList)
  }
}

// 初始化双层饼图
const initChart = () => {
  const { countData, amountData } = getPieData()
  const chartDom = document.getElementById('contractPieChart')
  if (!chartDom) return

  // 无数据提示（两个维度都无数据时显示）
  if (countData.length === 0 && amountData.length === 0) {
    chartDom.innerHTML = '<div style="width:100%;height:100%;display:flex;align-items:center;justify-content:center;color:#999;font-size:14px;">暂无有效统计数据</div>'
    return
  }

  // 销毁已有实例
  if (myChart.value) {
    myChart.value.dispose()
  }

  myChart.value = echarts.init(chartDom)

  // 颜色配置（两个维度使用不同色系，避免混淆）
  const countColors = [
    '#409EFF', '#67C23A', '#FF9F7F', '#FFC53D',
    '#8592AD', '#52C41A', '#FA8C16', '#F5222D'
  ]
  const amountColors = [
    '#722ED1', '#1890FF', '#00B42A', '#FF7D00',
    '#F7BA1E', '#E53E3E', '#9F7AEA', '#38B2AC'
  ]

  const option: echarts.EChartsOption = {
    title: {
      text: '合同数据统计（数量/金额）',
      left: 'center',
      textStyle: { fontSize: 16, fontWeight: 500 }
    },
    tooltip: {
      trigger: 'item',
      formatter: '{a} <br/>{b}: {c} ({d}%)',
      textStyle: { fontSize: 12 }
    },
    legend: {
      orient: 'vertical',
      left: 'right',
      top: 'center',
      textStyle: { fontSize: 12 },
      type: 'scroll',
      // 合并两个维度的图例（去重）
      data: Array.from(new Set([
        ...countData.map(item => item.name),
        ...amountData.map(item => item.name)
      ]))
    },
    series: [
      // 内层：合同数量（小半径）
      {
        name: '合同数量',
        type: 'pie',
        selectedMode: 'single', // 支持单选高亮
        radius: [0, '35%'], // 内层半径范围
        center: ['50%', '50%'], // 图表居中
        label: {
          position: 'inner', // 标签在饼图内部
          formatter: '{c}', // 只显示数值
          fontSize: 11,
          fontWeight: 500
        },
        labelLine: { show: false }, // 隐藏标签线
        itemStyle: {
          borderRadius: 6,
          borderColor: '#fff',
          borderWidth: 2
        },
        data: countData,
        color: countColors
      },
      // 外层：合同总金额（大半径）
      {
        name: '合同总金额',
        type: 'pie',
        radius: ['45%', '70%'], // 外层半径范围（与内层保持10%间距）
        center: ['50%', '50%'],
        labelLine: {
          show: true,
          length: 15,
          length2: 20,
          lineStyle: { width: 1 }
        },
        label: {
          formatter: '{b}: {c}', // 显示名称和数值
          fontSize: 11
        },
        itemStyle: {
          borderRadius: 6,
          borderColor: '#fff',
          borderWidth: 2
        },
        data: amountData,
        color: amountColors
      }
    ]
  }

  myChart.value.setOption(option)
}

// 窗口自适应
const handleResize = () => {
  myChart.value?.resize()
}

// 监听数据变化，重新初始化图表
watch(
  () => props.data,
  () => {
    const chartDom = document.getElementById('contractPieChart')
    if (chartDom) chartDom.innerHTML = ''
    initChart()
  },
  { immediate: true, deep: true } // 深度监听对象内部变化
)

// 生命周期管理
onMounted(() => {
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
  myChart.value?.dispose()
})
</script>