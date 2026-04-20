<script setup lang="ts">
import { ref, reactive, onMounted, watch, computed, onUnmounted } from 'vue';
import { 
  Bot, Zap, Clock, Battery, Thermometer, 
  Settings, AlertTriangle, ShieldCheck, Cpu 
} from 'lucide-vue-next';
import * as echarts from 'echarts';

interface State {
  time: string;
  priceType: string;
  soc: number;
  temp: number;
  tempA: number;
  tempB: number;
  loadA: number;
  loadB: number;
  voltage: number;
  power: number;
  status: 'success' | 'warning' | 'danger';
  action: string;
  logic: string;
  show_modal: boolean;
  currentScenario: 'night' | 'noon' | 'imbalance';
}

const state = reactive<State>({
  time: '23:30:00',
  priceType: '谷电时段',
  soc: 35,
  temp: 26,
  tempA: 26,
  tempB: 26,
  loadA: 40,
  loadB: 40,
  voltage: 3.82,
  power: 150.0,
  status: 'success', 
  action: '谷电充电中',
  logic: '低电价 + 温度正常',
  show_modal: false,
  currentScenario: 'night'
});

const aiExplanations = ref([
  "时间：23:30 | 电价：谷电 | SOC：35% | 温度：26℃",
  "AI 建议动作：启动充电",
  "原因分析：现在是谷电时段，电费最便宜。电池电量较低且温度处于正常区间，适合大功率补能。"
]);

const jsonInput = ref('');
const showAiNotify = ref(false);
const notifyTimer = ref<number | null>(null);

let gaugeChart: echarts.ECharts | null = null;
let trendChart: echarts.ECharts | null = null;
let chartData = { 
  soc: Array(20).fill(35), 
  power: Array(20).fill(150),
  temp: Array(20).fill(26)
};

// 计算属性
const statusColor = computed(() => {
  if (state.status === 'danger') return '#ff0055';
  if (state.status === 'warning') return '#ffaa00';
  return '#00ff9d';
});

const statusLightClass = computed(() => ({
  'light-success': state.status === 'success',
  'light-warning': state.status === 'warning',
  'light-danger': state.status === 'danger'
}));

const statusBorderClass = computed(() => ({
  'neon-border-success': state.status === 'success',
  'neon-border-warning': state.status === 'warning',
  'neon-border-danger': state.status === 'danger'
}));

const kpiData = computed(() => {
  const tempNote = state.status === 'warning' && state.temp >= 40 ? '温度偏高，建议降功率运行' : null;
  return [
    { label: '当前时段 (Period)', value: state.priceType, unit: '', percent: 100, color: '#ffaa00', icon: Clock },
    { label: '电池电量 (SOC)', value: Math.floor(state.soc), unit: '%', percent: state.soc, color: '#00f2ff', icon: Battery },
    { label: '当前功率 (Power)', value: Math.floor(state.power), unit: 'kW', percent: (state.power/200)*100, color: '#bc00ff', icon: Zap },
    { label: '环境温度 (Temp)', value: state.temp, unit: '℃', note: tempNote, percent: (state.temp/80)*100, color: statusColor.value, icon: Thermometer }
  ];
});

const statusLights = computed(() => [
  { id: 1, label: 'BMS 通讯状态', active: true, class: 'light-success' },
  { id: 2, label: '电池组均衡度', active: state.status !== 'danger', class: state.status === 'warning' ? 'light-warning' : 'light-success' },
  { id: 3, label: '绝缘检测告警', active: state.status === 'danger', class: 'light-danger' },
  { id: 4, label: '充放电回路', active: state.status !== 'danger', class: 'light-success' }
]);

const updateGauge = () => {
  if (!gaugeChart) return;
  gaugeChart.setOption({
    backgroundColor: 'transparent',
    series: [{
      type: 'gauge',
      startAngle: 90,
      endAngle: -270,
      min: 0,
      max: 6,
      radius: '85%',
      center: ['50%', '50%'],
      axisLine: { lineStyle: { width: 12, color: [[state.voltage/6, statusColor.value], [1, 'rgba(255,255,255,0.05)']], shadowBlur: 15, shadowColor: statusColor.value } },
      progress: { show: true, width: 12, itemStyle: { color: statusColor.value, shadowBlur: 10, shadowColor: statusColor.value } },
      pointer: { show: false },
      axisTick: { show: false },
      splitLine: { show: false },
      axisLabel: { show: false },
      anchor: { show: false },
      title: { show: true, offsetCenter: [0, '25%'], color: '#94a3b8', fontSize: 10, fontWeight: 'bold' },
      detail: { valueAnimation: true, fontSize: 42, offsetCenter: [0, '-5%'], color: '#fff', formatter: '{value}', textShadowBlur: 15, textShadowColor: statusColor.value, fontFamily: 'Orbitron' },
      data: [{ value: state.voltage, name: 'VOLTAGE (V)' }]
    }]
  }, true);
};

const updateTrend = () => {
  if (!trendChart) return;
  const statusColorVal = statusColor.value;
  const socColor = '#00f2ff';
  const powerColor = '#bc00ff';
  
  trendChart.setOption({
    backgroundColor: 'transparent',
    legend: {
      show: true,
      top: '2%',
      right: '5%',
      textStyle: { color: '#64748b', fontSize: 10 },
      icon: 'rect',
      itemWidth: 10,
      itemHeight: 2
    },
    tooltip: { 
      trigger: 'axis', 
      backgroundColor: 'rgba(15,23,42,0.9)', 
      borderColor: '#334155',
      textStyle: { color: '#fff', fontSize: 11 }
    },
    grid: { top: '18%', left: '4%', right: '2%', bottom: '8%', containLabel: true },
    xAxis: { type: 'category', boundaryGap: false, data: Array(20).fill('').map((_,i) => i+':00'), axisLine: { lineStyle: { color: '#334155' } }, axisLabel: { color: '#64748b', fontSize: 9 } },
    yAxis: { type: 'value', splitLine: { lineStyle: { color: 'rgba(255,255,255,0.03)' } }, axisLabel: { color: '#64748b', fontSize: 9 } },
    series: [
      { 
        name: '电量 (SOC)', type: 'line', smooth: true, data: chartData.soc, 
        lineStyle: { color: socColor, width: 2, shadowBlur: 10, shadowColor: socColor },
        areaStyle: { color: new echarts.graphic.LinearGradient(0,0,0,1, [{offset:0, color: socColor + '22'}, {offset:1, color: 'transparent'}]) },
        symbol: 'none'
      },
      { 
        name: '功率 (kW)', type: 'line', smooth: true, data: chartData.power, 
        lineStyle: { color: powerColor, width: 2, shadowBlur: 10, shadowColor: powerColor },
        areaStyle: { color: new echarts.graphic.LinearGradient(0,0,0,1, [{offset:0, color: powerColor + '22'}, {offset:1, color: 'transparent'}]) },
        symbol: 'none'
      },
      { 
        name: '温度 (℃)', type: 'line', smooth: true, data: chartData.temp, 
        lineStyle: { color: statusColorVal, width: 2, type: 'dashed', shadowBlur: 10, shadowColor: statusColorVal },
        symbol: 'none'
      }
    ]
  }, true);
};

const applyScenario = (type: State['currentScenario']) => {
  const generateNoise = (base: number, noise: number) => Array(20).fill(0).map(() => +(base + (Math.random() - 0.5) * noise).toFixed(1));
  
  state.currentScenario = type;
  if (type === 'night') {
    Object.assign(state, { 
      time: '23:30:00', priceType: '谷电时段', soc: 35, temp: 26, tempA: 26, tempB: 26, voltage: 3.82, power: 150, 
      status: 'success', action: '开始充电', logic: '低电价 + 温度正常', show_modal: false 
    });
    chartData.soc = Array(20).fill(35);
    chartData.power = generateNoise(150, 15);
    chartData.temp = generateNoise(26, 2);
    
    aiExplanations.value = [
      "老师向 AI 咨询结论：启动充电。",
      "核心逻辑：谷电时段（低电价） + 电池电量低 + 温度正常。",
      "界面表达：绿色状态标签显示“开始充电”，趋势图显示电量上升。"
    ];
  } else if (type === 'noon') {
    Object.assign(state, { 
      time: '12:40:00', priceType: '峰电时段', soc: 88, temp: 46, tempA: 46, tempB: 45, voltage: 4.25, power: 45, 
      status: 'warning', action: '停止继续充电，进入观察状态', logic: '温度偏高，建议降功率运行', show_modal: false 
    });
    chartData.soc = Array(20).fill(88);
    chartData.power = generateNoise(45, 30);
    chartData.temp = generateNoise(46, 3);
    
    aiExplanations.value = [
      "场景切换：中午 12:40，光伏发电升高，但电池温度已达 46℃。",
      "AI 决策：停止继续充电，进入观察状态。KPI 卡片已给出描述性警告。",
      "界面表达：颜色切换为橙色，强调“温度偏高”这一非结构化信息。"
    ];
  } else if (type === 'imbalance') {
    Object.assign(state, { 
      time: '15:15:00', priceType: '平电时段', soc: 60, temp: 37, tempA: 37, tempB: 32, 
      loadA: 85, loadB: 15,
      voltage: 4.10, power: 80, 
      status: 'danger', action: '调整充放电分配策略', logic: '电池组温差过大 (5℃) + 总负荷上升', show_modal: false 
    });
    chartData.soc = Array(20).fill(60);
    chartData.power = generateNoise(80, 25);
    chartData.temp = generateNoise(37, 2);

    aiExplanations.value = [
      "检测到极端异常：电池组 A (37℃) 高于电池组 B (32℃) 5℃。",
      "提示：场景 3 已移除置顶提示与弹窗强制提醒，仅通过界面状态色反馈风险。",
      "AI 决策：强制进入策略调整流程，锁定关键操作直到人工介入确认。"
    ];

    // 触发侧边 AI 通知
    showAiNotify.value = true;
    if (notifyTimer.value) clearTimeout(notifyTimer.value);
    notifyTimer.value = window.setTimeout(() => {
      showAiNotify.value = false;
    }, 15000);
  }
  updateTrend();
  syncJson();
};

const syncJson = () => {
  jsonInput.value = JSON.stringify(state, null, 2);
};

let intervalId: number | null = null;
const handleResize = () => {
  gaugeChart?.resize();
  trendChart?.resize();
};

onMounted(() => {
  gaugeChart = echarts.init(document.getElementById('gauge-chart'), 'dark');
  trendChart = echarts.init(document.getElementById('trend-chart'), 'dark');
  updateGauge();
  updateTrend();
  window.addEventListener('resize', handleResize);

  // 初始化一些基础噪声数据
  chartData.power = Array(20).fill(0).map(() => +(150 + (Math.random() - 0.5) * 15).toFixed(1));
  chartData.temp = Array(20).fill(0).map(() => +(26 + (Math.random() - 0.5) * 2).toFixed(1));
  syncJson();
  
  intervalId = window.setInterval(() => {
    // 模拟数据微动
    if (state.status !== 'danger') {
      state.voltage = +(state.voltage + (Math.random() - 0.5) * 0.01).toFixed(2);
      
      // 场景特定趋势：充电时 SOC 变动
      if (state.currentScenario === 'night') {
        // 按照用户要求，场景一电量固定为 35，不再随时间增长
        state.soc = 35;
        state.power = 150 + (Math.random() - 0.5) * 10;
      }
      
      // 功率随机变动
      let powerNoise = (state.currentScenario === 'noon' ? 10 : 5);
      let currentPower = +(state.power + (Math.random() - 0.5) * powerNoise).toFixed(1);
      
      chartData.power.push(currentPower);
      chartData.power.shift();
      chartData.soc.push(+(state.soc).toFixed(2));
      chartData.soc.shift();
      chartData.temp.push(+(state.temp + (Math.random() - 0.5) * 0.5).toFixed(1));
      chartData.temp.shift();
      
      updateGauge();
      updateTrend();
    }
  }, 1000);
});

onUnmounted(() => {
  if (intervalId) clearInterval(intervalId);
  window.removeEventListener('resize', handleResize);
  gaugeChart?.dispose();
  trendChart?.dispose();
});

watch(() => state.status, () => {
  updateGauge();
  updateTrend();
});
</script>

<template>
  <div class="h-screen w-screen flex flex-col gap-3 relative p-4 lg:p-6 overflow-hidden select-none box-border" :class="{'alarm-active': state.status === 'danger'}">
    <!-- 背景装饰 -->
    <div class="tech-marking top-2 left-4">OP_STREAM::CONNECTED</div>
    <div class="tech-marking bottom-2 right-4 italic">BMS_OS_V4.2.0_SECURED</div>
    <div class="scanline"></div>

    <!-- 右侧 AI 通知弹窗 (场景3专用) -->
    <transition name="slide-right">
      <div v-if="showAiNotify" class="fixed top-24 right-6 z-[100] w-[400px] p-6 border-l-4 border-rose-600 bg-white shadow-[0_20px_50px_rgba(0,0,0,0.3)] rounded-r-lg">
        <div class="flex items-start gap-5">
          <div class="w-12 h-12 rounded-full bg-rose-50 flex items-center justify-center text-rose-600 flex-shrink-0 shadow-sm animate-pulse">
            <svg class="w-7 h-7" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path></svg>
          </div>
          <div class="flex-1">
            <h3 class="text-sm font-black text-rose-600 uppercase tracking-widest mb-3 flex justify-between items-center">
              AI 实时诊断建议
              <button @click="showAiNotify = false" class="text-slate-300 hover:text-rose-600 transition-colors text-xl">×</button>
            </h3>
            <ul class="text-[13px] text-slate-600 space-y-2 leading-relaxed font-medium">
              <li class="flex items-start gap-2">
                <span class="text-rose-500 font-bold">•</span>
                当前电池组 A 温度高于电池组 B <span class="text-rose-600 font-bold">5℃</span>
              </li>
              <li class="flex items-start gap-2">
                <span class="text-rose-500 font-bold">•</span>
                监测到系统总负荷持续上升
              </li>
              <li class="mt-4 p-3 bg-rose-50 rounded border border-rose-100 text-slate-800 font-bold">
                建议动作：<span class="text-rose-700 underline underline-offset-4">调整充放电分配策略</span>
              </li>
            </ul>
          </div>
        </div>
        <div class="absolute bottom-0 left-0 h-1 bg-rose-600 animate-[progress_15s_linear]"></div>
      </div>
    </transition>

    <!-- Header: 看不见的电厂 -->
    <header class="h-16 flex-shrink-0 flex justify-between items-center industrial-panel relative px-6">
      <div class="corner-accent top-l"></div><div class="corner-accent top-r"></div>
      
      <div class="flex items-center gap-4">
        <div class="flex flex-col">
          <h1 class="text-lg font-bold tracking-[0.1em] text-white neon-text-cyan">
            锂电储能智慧调度平台
          </h1>
          <div class="flex items-center gap-2">
            <div class="status-light" :class="statusLightClass"></div>
            <span class="text-[9px] font-mono text-slate-500 uppercase tracking-widest">SYS_STATUS: {{ state.status === 'success' ? 'OK' : 'ATTENTION' }}</span>
          </div>
        </div>
        
        <div class="h-8 w-px bg-slate-800 hidden lg:block"></div>

        <!-- 核心参数组 -->
        <div class="hidden xl:flex items-center gap-2">
          <div class="data-chip px-2 py-0.5">
            <span class="text-[9px] text-slate-500">时段</span>
            <span class="data-value text-[10px]">{{ state.priceType }}</span>
          </div>
          <div class="data-chip px-2 py-0.5">
            <span class="text-[9px] text-slate-500">SOC</span>
            <span class="data-value text-[10px]">{{ state.soc }}%</span>
          </div>
          <div class="data-chip px-2 py-0.5">
            <span class="text-[9px] text-slate-500">功率</span>
            <span class="data-value text-[10px]">{{ state.power }}kW</span>
          </div>
          <div class="data-chip px-2 py-0.5">
            <span class="text-[9px] text-slate-500">温度</span>
            <span class="data-value text-[10px]">{{ state.temp }}℃</span>
          </div>
        </div>
      </div>

      <!-- 时间组件 -->
      <div class="striking-time-box text-xl py-1 px-4">
        {{ state.time }}
      </div>
    </header>

    <!-- 场景二警告横幅 -->
    <transition name="fade">
      <div v-if="state.status === 'warning'" 
           class="h-12 flex-shrink-0 industrial-panel px-6 flex items-center justify-between overflow-hidden shadow-[0_4px_15px_rgba(249,115,22,0.4)]"
           style="background: #f97316 !important; border: none !important;">
        <div class="flex items-center gap-3">
          <div class="flex items-center gap-2 text-white font-bold text-xs uppercase tracking-widest italic animate-pulse">
            <AlertTriangle class="w-4 h-4 fill-white text-orange-500" />
            <span>警告: 系统环境风险预警</span>
          </div>
        </div>
        <div class="flex items-center gap-3">
          <div class="flex flex-col items-end">
            <span class="text-[9px] text-white font-black italic uppercase tracking-tighter">STATUS: ABNORMAL</span>
            <span class="text-[8px] text-orange-100/90 font-mono uppercase tracking-[0.1em]">Manual Action Suggested</span>
          </div>
          <div class="w-2 h-2 rounded-full bg-white animate-ping"></div>
        </div>
      </div>
    </transition>

    <main class="flex-1 flex flex-col lg:flex-row gap-3 min-h-0">
      <!-- 左侧：教学演示大屏 (Digital Twin) -->
      <section class="flex-[3] flex flex-col gap-3 min-h-0">
        <!-- KPI 卡片组 -->
        <div class="grid grid-cols-2 xl:grid-cols-4 gap-3 flex-shrink-0">
          <div v-for="(item, index) in kpiData" :key="index" 
               class="industrial-panel p-3 flex flex-col justify-between transition-all duration-500"
               :style="{ borderLeft: '3px solid ' + item.color, boxShadow: 'inset 0 0 15px ' + item.color + '15' }">
            <div class="corner-accent top-r" :style="{ borderColor: item.color }"></div>
            <div class="flex justify-between items-start">
              <span class="text-[9px] text-slate-400 font-bold uppercase tracking-widest">{{ item.label }}</span>
              <div class="p-1 rounded bg-slate-800/50">
                <component :is="item.icon" class="w-3 h-3" :style="{ color: item.color }" />
              </div>
            </div>
            <div class="flex items-baseline gap-1.5 mt-1">
              <span class="text-2xl font-orbitron font-bold text-white" :style="{ textShadow: '0 0-8px ' + item.color }">{{ item.value }}</span>
              <span class="text-[10px] text-slate-500 font-mono">{{ item.unit }}</span>
            </div>
            
            <!-- 动态描述提示 -->
            <div v-if="item.note" class="text-[12px] font-black text-orange-400 animate-pulse mt-1 leading-tight tracking-tight h-4 truncate">
              {{ item.note }}
            </div>

            <div v-else class="w-full h-1 bg-slate-800 rounded-full overflow-hidden mt-1">
              <div class="h-full transition-all duration-1000" 
                   :style="{ width: item.percent + '%', backgroundColor: item.color }"></div>
            </div>
          </div>
        </div>

        <!-- 核心图表区 -->
        <div class="flex-1 grid grid-cols-1 xl:grid-cols-3 gap-3 min-h-0">
          <!-- 仪表盘：电压监控 -->
          <div class="industrial-panel col-span-1 flex flex-col relative min-h-0">
            <div class="corner-accent top-l"></div><div class="corner-accent top-r"></div>
            <div class="absolute top-2 left-4 z-20 flex items-center gap-2">
              <span class="w-1 h-3 bg-cyan-500 shadow-[0_0_8px_#00f2ff]"></span>
              <span class="text-[9px] font-bold text-slate-400 uppercase tracking-tighter">核心电压 (BMS)</span>
            </div>
            <div id="gauge-chart" class="flex-1 w-full h-full"></div>
            <div class="absolute bottom-1 right-2 text-[8px] text-slate-600 italic">UI表达：实时电压波动</div>
          </div>

          <!-- 趋势图：功率与SOC -->
          <div class="industrial-panel col-span-1 xl:col-span-2 flex flex-col relative min-h-0">
            <div class="corner-accent top-l"></div><div class="corner-accent top-r"></div>
            <div class="absolute top-2 left-4 z-20 flex items-center gap-2">
              <span class="w-1 h-3 bg-purple-500 shadow-[0_0_8px_#bc00ff]"></span>
              <span class="text-[9px] font-bold text-slate-400 uppercase tracking-tighter">实时趋势负载曲线</span>
            </div>
            <div id="trend-chart" class="flex-1 w-full h-full"></div>
            <div class="absolute bottom-1 right-2 text-[8px] text-slate-600 italic">UI表达：多维度数据关联分析</div>
          </div>
        </div>

        <!-- 底部：策略与状态 -->
        <div class="h-32 flex-shrink-0 flex flex-col lg:flex-row gap-3">
          <!-- 启动说明板块 (代替了电池监控) -->
          <div class="w-full lg:w-[450px] industrial-panel p-3 flex items-center justify-between bg-slate-900/40"
               :style="{ borderLeft: '3px solid ' + statusColor }">
            <div class="corner-accent top-l"></div><div class="corner-accent bot-r"></div>
            
            <div class="flex flex-col flex-1">
              <div class="flex items-center gap-2 mb-1">
                <Cpu class="w-3.5 h-3.5 text-slate-500" />
                <span class="text-[9px] text-slate-500 font-bold uppercase tracking-widest">启动说明 (Action)</span>
              </div>
              <div class="text-2xl font-black italic tracking-wider transition-all duration-500 leading-tight" :style="{ color: statusColor, textShadow: '0 0 15px ' + statusColor + '44' }">
                {{ state.action }}
              </div>
              <div class="mt-1 text-[8px] text-slate-600 font-mono">EXECUTION_ID: {{ (Math.random() * 1000).toFixed(0) }}</div>
            </div>

            <!-- 智能电池动画状态 -->
            <div class="relative w-24 h-14 flex items-center justify-center scale-110 ml-4 border-l border-slate-800 pl-4">
              <!-- 电池外壳 -->
              <div class="w-14 h-7 border-2 rounded-sm p-0.5 relative flex items-center transition-all duration-500" 
                   :style="{ borderColor: statusColor, color: statusColor }"
                   :class="{ 
                     'battery-shell-charging': state.currentScenario === 'night',
                     'battery-shell-warning': state.currentScenario === 'noon',
                     'battery-shell-danger': state.currentScenario === 'imbalance'
                   }">
                <!-- 电池极耳 -->
                <div class="absolute -right-1.5 w-1 h-3 bg-current rounded-e-sm"></div>
                
                <!-- 电池电量填充 & 动画 -->
                <div class="h-full bg-current opacity-80 transition-all duration-500 relative overflow-hidden" 
                     :style="{ width: state.soc + '%' }"
                     :class="{ 
                       'charging-flow': state.currentScenario === 'night',
                       'warning-pulse': state.currentScenario === 'noon',
                       'danger-shake': state.currentScenario === 'imbalance'
                     }">
                  <!-- 波纹光效 (仅充电时) -->
                  <div v-if="state.currentScenario === 'night'" class="absolute inset-0 bg-white/30 -skew-x-12 animate-[flow_1.5s_infinite]"></div>
                </div>

                <!-- 状态图标叠加 -->
                <div class="absolute inset-0 flex items-center justify-center">
                  <Zap v-if="state.currentScenario === 'night'" class="w-3.5 h-3.5 text-white fill-white animate-pulse" />
                  <span v-else-if="state.currentScenario === 'noon'" class="text-[10px] font-black text-white animate-bounce-slow">!</span>
                  <AlertTriangle v-else class="w-3.5 h-3.5 text-white fill-white animate-pulse" />
                </div>
              </div>
            </div>
          </div>

          <!-- 策略说明区 (大脑) -->
          <div class="flex-1 industrial-panel p-3 flex flex-col justify-center relative overflow-hidden"
               :class="statusBorderClass">
            <div class="corner-accent top-r" :style="{ borderColor: statusColor }"></div>
            <div class="corner-accent bot-l" :style="{ borderColor: statusColor }"></div>
            <div class="absolute top-0 right-0 p-1.5 text-[9px] font-mono text-slate-600">AI_STRATEGY_ENGINE</div>
            <div class="flex items-center gap-4">
              <div class="flex-1 border-slate-800 gap-1 flex flex-col">
                <div class="text-[9px] text-slate-500 font-bold uppercase tracking-widest flex items-center gap-2">
                  <span class="w-1.5 h-1.5 rounded-full bg-cyan-500/50"></span>
                  策略说明区 (Strategy Selection)
                </div>
                <div class="text-[11px] text-slate-400">
                  当前策略依据：
                  <span class="text-3xl font-black underline underline-offset-[10px] decoration-3 decoration-slate-600/50 block mt-2 tracking-tighter leading-none" :style="{ color: statusColor, textShadow: '0 0 15px ' + statusColor + '33' }">
                    "{{ state.logic }}"
                  </span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- 右侧：AI 助手解释区 (AI Control Hub) -->
      <aside class="flex-1 flex flex-col gap-3 min-h-0">
        <div class="industrial-panel p-4 flex flex-col gap-4 h-full min-h-0">
          <div class="corner-accent top-l"></div><div class="corner-accent top-r"></div>
          <div class="corner-accent bot-l"></div><div class="corner-accent bot-r"></div>
          
          <div class="flex items-center justify-between border-b border-slate-800 pb-3 flex-shrink-0">
            <div class="flex items-center gap-2">
              <div class="relative w-8 h-8 bg-white rounded-lg border border-slate-200 flex items-center justify-center p-0.5 shadow-sm">
                <div class="w-full h-full bg-[#1e40af] rounded flex items-center justify-center relative overflow-hidden">
                  <Zap class="w-4 h-4 text-cyan-300 fill-cyan-300" />
                </div>
              </div>
              <h2 class="text-base font-bold text-white font-orbitron italic">AI 储能助手</h2>
            </div>
            <span class="px-1.5 py-0.5 bg-cyan-500/10 text-cyan-400 text-[8px] font-bold border border-cyan-500/20 uppercase tracking-tighter">Teaching Mode</span>
          </div>

          <!-- 场景切换 -->
          <div class="space-y-2 flex-shrink-0">
            <p class="text-[9px] text-slate-500 uppercase font-black tracking-widest">Select Scenario</p>
            <div class="grid grid-cols-1 gap-1.5">
              <button v-for="sc in ['night', 'noon', 'imbalance']" :key="sc" @click="applyScenario(sc)"
                      class="px-3 py-2 border transition-all text-left"
                      :class="state.currentScenario === sc ? 'bg-cyan-500/10 border-cyan-500' : 'bg-slate-900/50 border-slate-800 hover:border-slate-700'">
                <div class="text-[11px] font-bold flex justify-between items-center" 
                     :class="state.currentScenario === sc ? 'text-cyan-400' : 'text-slate-400'">
                  {{ sc === 'night' ? '场景 1：深夜谷电蓄能' : sc === 'noon' ? '场景 2：中午高温观察' : '场景 3：负荷失衡预警' }}
                  <span v-if="state.currentScenario === sc" class="text-[8px]">●</span>
                </div>
                <div class="text-[9px] text-slate-600 truncate">
                  {{ sc === 'night' ? '23:30 | 谷电 | SOC 35%' : sc === 'noon' ? '12:40 | 峰电 | SOC 88%' : '负载上升 | 组温差 > 5℃' }}
                </div>
              </button>
            </div>
          </div>

          <!-- AI 解释区 -->
          <div class="flex-1 flex flex-col gap-2 min-h-0">
            <p class="text-[9px] text-slate-500 uppercase font-black tracking-widest flex-shrink-0">Live Analysis</p>
            <div class="flex-1 bg-black/40 border border-slate-800 rounded p-3 overflow-hidden">
              <div class="space-y-3">
                <div v-for="(msg, i) in aiExplanations" :key="i" class="flex gap-2">
                  <div class="w-5 h-5 rounded bg-cyan-500/20 flex items-center justify-center flex-shrink-0 text-cyan-500 font-black text-[8px] border border-cyan-500/10">AI</div>
                  <div class="text-[10px] text-slate-300 leading-relaxed">{{ msg }}</div>
                </div>
              </div>
            </div>
          </div>

          <!-- JSON Bridge -->
          <details class="group flex-shrink-0">
            <summary class="text-[9px] text-slate-700 cursor-pointer hover:text-slate-500 transition-colors uppercase font-mono">Stream Data</summary>
            <div class="mt-1 h-20">
              <textarea v-model="jsonInput" readonly
                        class="w-full h-full bg-black/60 border border-slate-800 rounded p-1.5 font-mono text-[8px] text-slate-600 resize-none focus:outline-none"></textarea>
            </div>
          </details>
        </div>
      </aside>
    </main>

    <!-- 紧急告警/提示弹窗 -->
    <transition name="fade">
      <div v-if="state.show_modal" class="fixed inset-0 z-[100] flex items-center justify-center bg-black/90 backdrop-blur-xl p-4">
        <div class="industrial-panel p-10 max-w-md w-full text-center border-2 transition-all duration-500 shadow-[0_0_50px_rgba(0,0,0,0.4)]"
             :class="state.status === 'danger' ? 'border-rose-500 shadow-rose-500/20' : 'border-orange-500 shadow-orange-500/20'">
          
          <!-- 动态图标 -->
          <div class="w-20 h-20 rounded-full flex items-center justify-center mx-auto mb-6 border pulse-glow"
               :class="state.status === 'danger' ? 'bg-rose-500/20 border-rose-500 text-rose-500' : 'bg-orange-500/20 border-orange-500 text-orange-500'">
            <svg v-if="state.status === 'danger'" class="w-10 h-10" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"></path>
            </svg>
            <svg v-else class="w-10 h-10" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path>
            </svg>
          </div>

          <h2 class="text-3xl font-bold mb-4 font-orbitron tracking-widest uppercase"
              :class="state.status === 'danger' ? 'text-rose-500' : 'text-orange-500'">
            {{ state.status === 'danger' ? '系统安全干预' : '系统风险提示' }}
          </h2>
          
          <p class="text-slate-300 mb-8 leading-relaxed">
            {{ state.status === 'danger' ? 'AI 检测到极端风险：' : 'AI 检测到异常状态：' }}
            <span class="block mt-2 text-white font-bold text-lg underline decoration-slate-500 underline-offset-4">
              {{ state.logic }}
            </span>
          </p>

          <button @click="state.show_modal = false" 
                  class="w-full py-4 text-white font-bold rounded transition-all shadow-[0_0_20px_rgba(0,0,0,0.3)]"
                  :class="state.status === 'danger' ? 'bg-rose-600 hover:bg-rose-500' : 'bg-orange-600 hover:bg-orange-500'">
            {{ state.status === 'danger' ? '确认并解除锁定' : '已知晓并进入观察模式' }}
          </button>
          
          <div class="mt-4 text-[9px] text-slate-500 uppercase font-mono italic">
            SECURE_PROTOCOL_ACTIVE :: {{ state.status === 'danger' ? 'HIGH_PRIORITY' : 'MODERATE_RISK' }}
          </div>
        </div>
      </div>
    </transition>
  </div>
</template>

<style>
:root {
  --bg-color: #020617;
  --panel-bg: rgba(7, 12, 28, 0.85);
  --neon-green: #00ff9d;
  --neon-orange: #ffaa00;
  --neon-red: #ff0055;
  --neon-cyan: #00f2ff;
  --border-color: rgba(0, 242, 255, 0.15);
  --accent-blue: #3b82f6;
}

body {
  background-color: var(--bg-color);
  background-image: 
    radial-gradient(circle at 50% 50%, rgba(59, 130, 246, 0.1) 0%, transparent 70%),
    linear-gradient(rgba(0, 242, 255, 0.05) 1px, transparent 1px),
    linear-gradient(90deg, rgba(0, 242, 255, 0.05) 1px, transparent 1px);
  background-size: cover, 30px 30px, 30px 30px;
  color: #f8fafc;
  font-family: 'Noto Sans SC', sans-serif;
  margin: 0;
  min-height: 100vh;
  overflow: hidden;
}

.font-orbitron { font-family: 'Orbitron', sans-serif; }
.font-mono { font-family: 'JetBrains Mono', monospace; }

/* 赛博几何面板 */
.industrial-panel {
  background: var(--panel-bg);
  backdrop-filter: blur(16px);
  border: 1px solid var(--border-color);
  position: relative;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.6);
  transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  clip-path: polygon(
    0 0, 
    calc(100% - 10px) 0, 100% 10px, 
    100% 100%, 
    10px 100%, 0 calc(100% - 10px)
  );
}

.industrial-panel::before {
  content: '';
  position: absolute;
  top: 0; left: 0; width: 100%; height: 100%;
  background: linear-gradient(135deg, rgba(0, 242, 255, 0.05) 0%, transparent 40%);
  pointer-events: none;
}

/* 几何装饰角 */
.corner-accent {
  position: absolute;
  width: 8px;
  height: 8px;
  border-color: var(--neon-cyan);
  z-index: 5;
}
.top-l { top: 0; left: 0; border-top: 2px solid; border-left: 2px solid; }
.top-r { top: 0; right: 0; border-top: 2px solid; border-right: 2px solid; }
.bot-l { bottom: 0; left: 0; border-bottom: 2px solid; border-left: 2px solid; }
.bot-r { bottom: 0; right: 0; border-bottom: 2px solid; border-right: 2px solid; }

.neon-glow-cyan { box-shadow: 0 0 20px rgba(0, 242, 255, 0.2), inset 0 0 10px rgba(0, 242, 255, 0.1); }
.neon-text-cyan { text-shadow: 0 0 12px var(--neon-cyan); color: var(--neon-cyan); }
.neon-text-green { text-shadow: 0 0 12px var(--neon-green); color: var(--neon-green); }
.neon-text-orange { text-shadow: 0 0 12px var(--neon-orange); color: var(--neon-orange); }
.neon-text-red { text-shadow: 0 0 12px var(--neon-red); color: var(--neon-red); }

.industrial-panel::after {
  content: '';
  position: absolute;
  top: 0; left: 0; width: 100%; height: 100%;
  background: linear-gradient(135deg, rgba(255,255,255,0.05) 0%, transparent 50%);
  pointer-events: none;
}

/* 状态灯 */
.status-light {
  width: 10px;
  height: 10px;
  border-radius: 2px;
  transition: all 0.3s ease;
  transform: rotate(45deg);
}
.light-success { background: var(--neon-green); box-shadow: 0 0 12px var(--neon-green); }
.light-warning { background: var(--neon-orange); box-shadow: 0 0 12px var(--neon-orange); }
.light-danger { background: var(--neon-red); box-shadow: 0 0 12px var(--neon-red); }
.light-off { background: #1e293b; box-shadow: none; border: 1px solid rgba(255,255,255,0.1); }

/* 装饰性背景标线 */
.tech-marking {
  position: absolute;
  font-size: 8px;
  color: rgba(0, 242, 255, 0.2);
  font-family: 'JetBrains Mono';
  pointer-events: none;
  z-index: 100;
}

/* 扫描线 */
.scanline {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0; left: 0;
  background: linear-gradient(to bottom, transparent 50%, rgba(0, 242, 255, 0.01) 50%);
  background-size: 100% 4px;
  pointer-events: none;
  z-index: 10;
}

@keyframes pulse-glow {
  0%, 100% { opacity: 0.5; }
  50% { opacity: 1; }
}
.pulse-glow { animation: pulse-glow 2s infinite; }

/* 紧急告警全屏闪烁 */
@keyframes alarm-flash {
  0%, 100% { background-color: transparent; }
  50% { background-color: rgba(255, 0, 85, 0.05); }
}
.alarm-active { animation: alarm-flash 0.6s infinite; }

/* 自定义滚动条 */
::-webkit-scrollbar { width: 3px; }
::-webkit-scrollbar-track { background: var(--bg-color); }
::-webkit-scrollbar-thumb { background: rgba(0, 242, 255, 0.2); border-radius: 10px; }
::-webkit-scrollbar-thumb:hover { background: var(--neon-cyan); }

/* 突兀的时间组件 */
.striking-time-box {
  background: rgba(0, 0, 0, 0.4);
  color: #fff;
  padding: 6px 24px;
  font-family: 'Orbitron', sans-serif;
  font-weight: 800;
  border: 2px solid #00f2ff;
  clip-path: polygon(10% 0, 100% 0, 90% 100%, 0% 100%);
  box-shadow: 0 0 15px rgba(0, 242, 255, 0.5), inset 0 0 15px rgba(0, 242, 255, 0.2);
  position: relative;
  transform: skewX(-5deg);
  z-index: 50;
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 160px;
}

.data-chip {
  background: rgba(0, 242, 255, 0.05);
  border: 1px solid rgba(0, 242, 255, 0.1);
  padding: 2px 12px;
  display: flex;
  align-items: center;
  gap: 8px;
  font-family: 'JetBrains Mono';
}

.data-value {
  color: var(--neon-cyan);
  font-weight: bold;
}

@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.3; }
}
.blink { animation: blink 1s infinite; }

.fade-enter-active, .fade-leave-active { transition: opacity 0.5s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }

/* 右侧滑入动画 */
.slide-right-enter-active, .slide-right-leave-active {
  transition: all 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}
.slide-right-enter-from {
  transform: translateX(100%);
  opacity: 0;
}
.slide-right-leave-to {
  transform: translateX(50%);
  opacity: 0;
}

@keyframes progress {
  from { width: 100%; }
  to { width: 0%; }
}

/* 电池流光动画 */
@keyframes flow {
  from { transform: translateX(-100%) skewX(-20deg); }
  to { transform: translateX(200%) skewX(-20deg); }
}

@keyframes battery-pulse {
  0%, 100% { opacity: 0.8; filter: brightness(1); }
  50% { opacity: 0.4; filter: brightness(1.5); }
}

@keyframes battery-shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-2px); }
  75% { transform: translateX(2px); }
}

.charging-flow {
  box-shadow: 0 0 15px currentColor;
}

.warning-pulse {
  animation: battery-pulse 1.5s infinite ease-in-out;
}

.danger-shake {
  animation: battery-shake 0.2s infinite linear;
  background-color: var(--neon-red) !important;
}

@keyframes battery-grow {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.05); }
}

@keyframes shell-glow {
  0%, 100% { filter: drop-shadow(0 0 2px currentColor); }
  50% { filter: drop-shadow(0 0 8px currentColor); }
}

.battery-shell-charging {
  animation: battery-grow 2s infinite ease-in-out;
}

.battery-shell-warning {
  animation: shell-glow 1s infinite ease-in-out;
}

.battery-shell-danger {
  animation: battery-shake 0.1s infinite linear;
}

.animate-bounce-slow {
  animation: bounce 2s infinite;
}

/* 自定义滚动条 */
.custom-scrollbar::-webkit-scrollbar {
  width: 4px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: rgba(0, 0, 0, 0.1);
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background: rgba(100, 116, 139, 0.3);
  border-radius: 10px;
}
.custom-scrollbar::-webkit-scrollbar-thumb:hover {
  background: rgba(100, 116, 139, 0.5);
}
</style>
