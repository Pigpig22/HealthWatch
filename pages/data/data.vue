<template>
  <view class="container">
    <view class="background-blur"></view>

    <view class="content-wrapper">
      <view class="header">
        <text class="title">数据趋势分析</text>
        <text class="subtitle">实时监测 · 历史记录</text>
      </view>

      <view class="tab-group">
        <button
          class="tab-btn"
          :class="{ active: currentType === 'temp' }"
          @click="switchType('temp')"
        >
          🌡️ 温度
        </button>
        <button
          class="tab-btn"
          :class="{ active: currentType === 'heart' }"
          @click="switchType('heart')"
        >
          ❤️ 心率
        </button>
        <button
          class="tab-btn"
          :class="{ active: currentType === 'steps' }"
          @click="switchType('steps')"
        >
          👟 步数
        </button>
        <button
          class="tab-btn"
          :class="{ active: currentType === 'spo2' }"
          @click="switchType('spo2')"
        >
          🫁 血氧
        </button>
      </view>

      <view class="chart-card">
        <view class="current-data-display">
          <view class="value-badge" :class="currentType">
            <text class="value-number">{{ currentDisplayValue }}</text>
            <text class="value-unit">{{ currentUnit }}</text>
          </view>
          <view class="data-meta">
            <text class="meta-label">最新数据</text>
            <text class="meta-time">{{ latestTime }}</text>
          </view>
        </view>

        <view class="chart-container">
          <canvas canvas-id="lineChart" id="lineChart" class="chart-canvas"></canvas>
        </view>
      </view>

      <view class="stats-row">
        <view class="stat-card">
          <text class="stat-label">最高</text>
          <text class="stat-value">{{ stats.max }}</text>
        </view>
        <view class="stat-card">
          <text class="stat-label">最低</text>
          <text class="stat-value">{{ stats.min }}</text>
        </view>
        <view class="stat-card">
          <text class="stat-label">平均</text>
          <text class="stat-value">{{ stats.avg }}</text>
        </view>
      </view>

	  
      <view class="history-section">
        <view class="history-header">
          <text class="history-title">历史记录</text>
          <view class="history-actions">
            <text class="action-link" @click="clearHistory">清除</text>
            <text class="action-link" @click="exportData">导出</text>
          </view>
        </view>
        <scroll-view scroll-y="true" class="history-list">
          <view class="history-item" :class="currentType" v-for="(item, index) in reverseHistory" :key="index">
            <text class="history-time">{{ item.time }}</text>
            <text class="history-value">{{ item.value }}{{ currentUnit }}</text>
          </view>
          <view class="history-empty" v-if="reverseHistory.length === 0">
            <text class="empty-text">暂无历史数据</text>
          </view>
        </scroll-view>
      </view>
    </view>
  </view>
</template>

<script setup>
import { ref, computed, watch, nextTick, onMounted, onUnmounted } from 'vue';

const currentType = ref('temp');

const healthData = ref({ temp: '--', heart: '--', steps: '--', spo2: '--' });

const historyData = ref({ temp: [], heart: [], steps: [], spo2: [] });

const typeNames = { temp: '温度', heart: '心率', steps: '步数', spo2: '血氧' };
const typeUnits = { temp: '°C', heart: ' bpm', steps: ' 步', spo2: ' %' };
const colorMap = {
  temp: { r: 255, g: 112, b: 67 },
  heart: { r: 233, g: 30, b: 99 },
  steps: { r: 33, g: 150, b: 243 },
  spo2: { r: 156, g: 39, b: 176 }
};

const getTypeName = computed(() => typeNames[currentType.value]);
const currentUnit = computed(() => typeUnits[currentType.value]);
const currentDisplayValue = computed(() => healthData.value[currentType.value]);

const latestTime = computed(() => {
  const history = historyData.value[currentType.value];
  if (history.length === 0) return '--:--:--';
  return history[history.length - 1].time;
});

const dataVersion = ref(0);

const calcAxisFromData = (type) => {
  const history = historyData.value[type];
  if (history.length === 0) {
    if (type === 'temp') return { min: 35, max: 39, step: 1, labels: ['35', '36', '37', '38', '39'] };
    if (type === 'heart') return { min: 50, max: 130, step: 20, labels: ['50', '70', '90', '110', '130'] };
    if (type === 'steps') return { min: 0, max: 10000, step: 2500, labels: ['0', '2500', '5000', '7500', '10000'] };
    if (type === 'spo2') return { min: 90, max: 100, step: 2.5, labels: ['90', '92.5', '95', '97.5', '100'] };
  }
  const values = history.map(item => item.value);
  const dataMax = Math.max(...values);
  const dataMin = Math.min(...values);
  const dataRange = dataMax - dataMin;
  let axisMin, axisMax;
  if (dataRange === 0) {
    const center = dataMax;
    if (type === 'temp') {
      axisMin = center - 1;
      axisMax = center + 1;
    } else {
      const half = Math.max(10, Math.ceil(center * 0.2));
      axisMin = Math.max(0, center - half);
      axisMax = center + half;
    }
  } else {
    const padding = dataRange * 0.25;
    axisMin = dataMin - padding;
    axisMax = dataMax + padding;
  }
  if (type === 'steps' || type === 'spo2') axisMin = Math.max(0, axisMin);
  const getStep = (range) => {
    if (range <= 2) return 0.5;
    if (range <= 10) return 1;
    if (range <= 30) return 5;
    if (range <= 100) return 10;
    if (range <= 300) return 50;
    if (range <= 1000) return 100;
    if (range <= 3000) return 500;
    return 1000;
  };
  const step = getStep(axisMax - axisMin);
  axisMin = Math.floor(axisMin / step) * step;
  axisMax = Math.ceil(axisMax / step) * step;
  const labels = [];
  for (let i = 0; i < 5; i++) {
    const val = axisMin + (axisMax - axisMin) * (i / 4);
    labels.push(type === 'temp' ? val.toFixed(1) : Math.round(val).toString());
  }
  return { min: axisMin, max: axisMax, step, labels };
};

const stats = computed(() => {
  const history = historyData.value[currentType.value];
  if (history.length === 0) return { max: '--', min: '--', avg: '--' };
  const values = history.map(item => item.value);
  const max = Math.max(...values);
  const min = Math.min(...values);
  const avg = (values.reduce((a, b) => a + b, 0) / values.length).toFixed(1);
  if (currentType.value === 'temp') return { max: max.toFixed(1), min: min.toFixed(1), avg };
  return { max, min, avg };
});

const reverseHistory = computed(() => [...historyData.value[currentType.value]].reverse());

const switchType = (type) => { currentType.value = type; };

const drawChart = () => {
  const type = currentType.value;
  const history = historyData.value[type];
  const ctx = uni.createCanvasContext('lineChart');
  const sysInfo = uni.getSystemInfoSync();
  const canvasW = Math.floor((sysInfo.windowWidth - 60) / 1);
  const canvasH = 180;
  const padL = 28
  const padR = 0
  const padT = 10;
  const padB = 20;
  const chartW = canvasW - padL - padR;
  const chartH = canvasH - padT - padB;
  ctx.clearRect(0, 0, canvasW, canvasH);
  const axis = calcAxisFromData(type);
  const c = colorMap[type];
  const lineColor = `rgb(${c.r},${c.g},${c.b})`;
  const fillColor = `rgba(${c.r},${c.g},${c.b},0.15)`;
  ctx.setStrokeStyle('#e0e0e0');
  ctx.setLineWidth(0.5);
  for (let i = 0; i < 5; i++) {
    const y = padT + (chartH / 4) * i;
    ctx.beginPath();
    ctx.moveTo(padL, y);
    ctx.lineTo(padL + chartW, y);
    ctx.stroke();
  }
  ctx.setFontSize(9);
  ctx.setFillStyle('#999');
  ctx.setTextAlign('right');
  const labels = [...axis.labels].reverse();
  for (let i = 0; i < 5; i++) {
    const y = padT + (chartH / 4) * i;
    ctx.fillText(labels[i], padL - 4, y + 3);
  }
  if (history.length > 0) {
    const points = history.map((item, idx) => {
      const x = history.length === 1 ? padL + chartW / 2 : padL + (idx / (history.length - 1)) * chartW;
      const ratio = (item.value - axis.min) / (axis.max - axis.min);
      const y = padT + chartH - ratio * chartH;
      return { x, y };
    });
    ctx.beginPath();
    ctx.moveTo(points[0].x, padT + chartH);
    points.forEach(p => ctx.lineTo(p.x, p.y));
    ctx.lineTo(points[points.length - 1].x, padT + chartH);
    ctx.closePath();
    ctx.setFillStyle(fillColor);
    ctx.fill();
    ctx.beginPath();
    ctx.setStrokeStyle(lineColor);
    ctx.setLineWidth(2);
    points.forEach((p, i) => { if (i === 0) ctx.moveTo(p.x, p.y); else ctx.lineTo(p.x, p.y); });
    ctx.stroke();
  }
  ctx.draw();
};

const addHistoryRecord = (type, value, time) => {
  healthData.value[type] = type === 'temp' ? value.toFixed(1) : value;
  historyData.value[type].push({ value, time });
  if (historyData.value[type].length > 50) historyData.value[type].shift();
  dataVersion.value++;
};

let lastProcessedTime = 0;
let lastProcessedDataStr = '';

const handleHealthDataUpdate = (data) => {
  const now = Date.now();
  const dataStr = JSON.stringify(data);
  if (lastProcessedDataStr === dataStr && now - lastProcessedTime < 1000) return;
  lastProcessedTime = now;
  lastProcessedDataStr = dataStr;
  const date = new Date();
  const currentTime = `${date.getHours()}:${String(date.getMinutes()).padStart(2, '0')}:${String(date.getSeconds()).padStart(2, '0')}`;
  if (data.temp !== undefined) addHistoryRecord('temp', parseFloat(data.temp), currentTime);
  if (data.heart !== undefined) addHistoryRecord('heart', parseInt(data.heart), currentTime);
  if (data.steps !== undefined) addHistoryRecord('steps', parseInt(data.steps), currentTime);
  if (data.spo2 !== undefined) addHistoryRecord('spo2', parseInt(data.spo2), currentTime);
};

const clearHistory = () => {
  historyData.value[currentType.value] = [];
  dataVersion.value++;
  uni.showToast({ title: '历史已清除', icon: 'success' });
};

const exportData = () => {
  const history = historyData.value[currentType.value];
  if (history.length === 0) {
    uni.showToast({ title: '暂无数据可导出', icon: 'none' });
    return;
  }
  const csvContent = history.map(item => `${item.time},${item.value}${currentUnit.value}`).join('\n');
  const header = `时间,${getTypeName.value}\n`;
  uni.setClipboardData({
    data: header + csvContent,
    success: () => uni.showToast({ title: '数据已复制到剪贴板', icon: 'success' })
  });
};

onMounted(() => {
  uni.$off('healthDataUpdate');
  uni.$on('healthDataUpdate', handleHealthDataUpdate);
  const globalHealthData = uni.getStorageSync('globalHealthData');
  if (globalHealthData) {
    try { handleHealthDataUpdate(JSON.parse(globalHealthData)); } catch (e) {}
  }
  nextTick(() => drawChart());
});

onUnmounted(() => {
  uni.$off('healthDataUpdate', handleHealthDataUpdate);
});

watch([dataVersion, currentType], () => nextTick(() => drawChart()));
</script>

<style scoped>
.container {
  min-height: 100vh;
  background: linear-gradient(135deg, #e8f5e9 0%, #c8e6c9 50%, #a5d6a7 100%);
  position: relative;
}

.background-blur {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: url('data:image/svg+xml,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 100 100"><circle cx="20" cy="20" r="15" fill="rgba(255,255,255,0.1)"/><circle cx="80" cy="60" r="25" fill="rgba(255,255,255,0.08)"/><circle cx="40" cy="80" r="20" fill="rgba(255,255,255,0.1)"/></svg>');
  background-size: 200px;
}

.content-wrapper {
  position: relative;
  z-index: 1;
  padding: 90rpx 30rpx;
  padding-bottom: 120rpx;
}

.header {
  text-align: center;
  margin-bottom: 40rpx;
}

.title {
  font-size: 48rpx;
  font-weight: bold;
  color: #2e7d32;
  display: block;
}

.subtitle {
  font-size: 24rpx;
  color: #666;
  margin-top: 8rpx;
  display: block;
}

.tab-group {
  display: flex;
  gap: 10rpx;
  margin-bottom: 30rpx;
  background: rgba(255, 255, 255, 0.5);
  padding: 8rpx;
  border-radius: 50rpx;
}

.tab-btn {
  flex: 1;
  padding: 14rpx 0;
  font-size: 24rpx;
  border-radius: 40rpx;
  border: none;
  background: transparent;
  color: #666;
}

.tab-btn.active {
  background: #2e7d32;
  color: white;
  box-shadow: 0 4rpx 15rpx rgba(46, 125, 50, 0.3);
}

.chart-card {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 30rpx;
  padding: 30rpx;
  box-shadow: 0 10rpx 40rpx rgba(0, 0, 0, 0.1);
  margin-bottom: 30rpx;
}

.current-data-display {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 30rpx;
  padding: 20rpx;
  background: linear-gradient(135deg, #e8f5e9 0%, #c8e6c9 100%);
  border-radius: 20rpx;
}

.value-badge {
  display: flex;
  align-items: baseline;
  gap: 8rpx;
}

.value-badge.temp { color: #ff7043; }
.value-badge.heart { color: #e91e63; }
.value-badge.steps { color: #2196f3; }
.value-badge.spo2 { color: #9c27b0; }

.value-number {
  font-size: 72rpx;
  font-weight: bold;
}

.value-unit {
  font-size: 32rpx;
  color: #666;
}

.data-meta {
  text-align: right;
}

.meta-label {
  font-size: 22rpx;
  color: #999;
  display: block;
}

.meta-time {
  font-size: 24rpx;
  color: #666;
  display: block;
  margin-top: 4rpx;
}

.chart-container {
  background: #fafafa;
  border-radius: 20rpx;
  padding: 20rpx;
}

.chart-canvas {
  width: 100%;
  height: 180px;
}

.stats-row {
  display: flex;
  gap: 20rpx;
  margin-bottom: 30rpx;
}

.stat-card {
  flex: 1;
  background: rgba(255, 255, 255, 0.6);
  padding: 20rpx;
  border-radius: 15rpx;
  text-align: center;
}

.stat-label {
  font-size: 22rpx;
  color: #999;
  display: block;
  margin-bottom: 8rpx;
}

.stat-value {
  font-size: 36rpx;
  font-weight: bold;
  color: #333;
}

.history-section {
  background: rgba(255, 255, 255, 0.6);
  border-radius: 30rpx;
  padding: 30rpx;
  box-shadow: 0 10rpx 40rpx rgba(0, 0, 0, 0.1);
  margin-bottom: 20rpx;
}

.history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15rpx;
}

.history-title {
  font-size: 28rpx;
  font-weight: bold;
  color: #333;
}

.history-actions {
  display: flex;
  gap: 20rpx;
}

.action-link {
  font-size: 24rpx;
  color: #2e7d32;
}

.history-list {
  height: 300rpx;
}

.history-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16rpx 20rpx;
  background: #f9f9f9;
  border-radius: 10rpx;
  margin-bottom: 10rpx;
  border-left: 6rpx solid;
}

.history-item.temp { border-left-color: #ff7043; }
.history-item.heart { border-left-color: #e91e63; }
.history-item.steps { border-left-color: #2196f3; }
.history-item.spo2 { border-left-color: #9c27b0; }

.history-time {
  font-size: 24rpx;
  color: #666;
}

.history-value {
  font-size: 28rpx;
  font-weight: bold;
  color: #333;
}

.history-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 60rpx 0;
}

.empty-text {
  font-size: 28rpx;
  color: #999;
}
</style>
