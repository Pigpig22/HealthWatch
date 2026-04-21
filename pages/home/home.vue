<template>
  <view class="container">
    <view class="content-wrapper">
      <view class="header">
        <text class="title">健康助手</text>
        <text class="subtitle">{{ statusMsg }}</text>
        <view class="ble-control">
          <button class="connect-btn" :class="{ 'btn-connected': isConnected }" @click="toggleConnect">
            {{ isConnected ? '断开设备' : (isScanning ? '搜索中...' : '连接 智能手表') }}
          </button>
        </view>
      </view>

      <!-- 设备列表 -->
      <view class="device-list-section" v-if="!isConnected">
        <!-- 扫描状态 -->
        <view class="scan-status" v-if="isScanning">
          <view class="scan-animation">
            <view class="scan-dot"></view>
            <view class="scan-dot"></view>
            <view class="scan-dot"></view>
          </view>
          <text class="scan-text">正在扫描附近设备...</text>
        </view>
        
        <!-- 设备列表标题和操作 -->
        <view class="device-list-header">
          <text class="device-list-title">
            {{ isScanning ? '扫描中...' : (devices.length > 0 ? `发现 ${devices.length} 个设备` : '未发现设备') }}
          </text>
          <view class="device-actions">
            <text class="action-btn" v-if="isScanning" @click="stopScan">停止扫描</text>
            <text class="action-btn" v-else @click="initBluetooth">重新扫描</text>
          </view>
        </view>
        
        <!-- 设备列表 -->
        <view class="device-list" v-if="devices.length > 0">
          <view
            class="device-item"
            v-for="device in devices"
            :key="device.deviceId"
            :class="{ 'selected': currentDevice && currentDevice.deviceId === device.deviceId }"
            @click="selectDevice(device)"
          >
            <view class="device-icon">
              <text class="icon-bluetooth">📡</text>
            </view>
            <view class="device-info">
              <text class="device-name">{{ device.name || '未知设备' }}</text>
              <text class="device-id">{{ device.deviceId.substring(0, 17) }}...</text>
            </view>
            <view class="device-rssi" :class="getSignalClass(device.RSSI)">
              <text class="rssi-value">{{ device.RSSI || '--' }}</text>
              <text class="rssi-unit">dBm</text>
            </view>
          </view>
        </view>
        
        <!-- 空状态 -->
        <view class="empty-state" v-else-if="!isScanning">
          <text class="empty-icon">📱</text>
          <text class="empty-text">未发现 HealthWatch 设备</text>
          <text class="empty-hint">请确保设备已开启并正在广播</text>
        </view>
      </view>

      <view class="card main-card">
        <view class="data-grid">
          <view class="data-item">
            <text class="data-value temp-color">{{ healthData.temp }}</text>
            <text class="data-label">体温 (°C)</text>
          </view>
          <view class="data-item">
            <text class="data-value heart-color">{{ healthData.heart }}</text>
            <text class="data-label">心率 (bpm)</text>
          </view>
          <view class="data-item">
            <text class="data-value steps-color">{{ healthData.steps }}</text>
            <text class="data-label">今日步数</text>
          </view>
          <view class="data-item">
            <text class="data-value spo2-color">{{ healthData.spo2 }}</text>
            <text class="data-label">血氧 (%)</text>
          </view>
        </view>
      </view>

      <view class="emergency-section">
        <view class="emergency-header">
          <view class="title-row">
            <text class="emergency-title">设备消息紧急联络</text>
            <text class="edit-tag" @click="openEditModal">⚙️ 设置</text>
          </view>
          <text class="emergency-desc">收到设备消息时，自动呼叫预设联系人</text>
        </view>

        <view class="contact-list">
          <view 
            class="contact-card" 
            v-for="(item, index) in contacts" 
            :key="index"
            @click="makeCall(item.phone)"
            v-show="item.name || item.phone" 
          >
            <view class="contact-avatar">
              <text class="avatar-text">{{ item.name ? item.name[0] : '?' }}</text>
            </view>
            <view class="contact-info">
              <text class="contact-name">{{ item.name || '未设置' }}</text>
              <text class="contact-phone">{{ item.phone || '未设置号码' }}</text>
            </view>
            <view class="call-btn">
              <text class="icon-phone">📞</text>
            </view>
          </view>
        </view>
      </view>

      <view class="message-section">
        <view class="message-header">
          <text class="message-title">设备实时消息</text>
          <text class="message-time" v-if="msgTime">最后接收: {{ msgTime }}</text>
        </view>
        <view class="message-box" :class="{ 'has-msg': latestMessage !== '等待设备发送消息...' }">
          <text class="message-text">{{ latestMessage }}</text>
        </view>
      </view>
    </view>

    <view class="modal-mask" v-if="showModal">
      <view class="modal-content">
        <text class="modal-title">管理紧急联系人</text>
        <scroll-view scroll-y="true" class="modal-scroll">
          <view class="edit-group" v-for="(item, index) in tempContacts" :key="index">
            <text class="group-label">联系人 {{ index + 1 }}</text>
            <input class="modal-input" v-model="item.name" placeholder="姓名 (如: 张医生)" />
            <input class="modal-input" v-model="item.phone" type="number" placeholder="电话号码" />
          </view>
        </scroll-view>
        <view class="modal-btns">
          <button class="modal-btn cancel" @click="showModal = false">取消</button>
          <button class="modal-btn confirm" @click="saveSettings">保存</button>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

// 蓝牙配置 - ESP32 BLE UUID（根据实际设备配置）
// ESP32代码中的配置：
// #define DEVICE_NAME     "ESP32_BLE"
// #define UUID16_SERVICE  0xFF00
// #define UUID16_CHR      0xFF01
// ESP32使用NimBLE协议栈，16位UUID会自动扩展为128位格式
// 扩展规则：在16位UUID前后添加固定前缀和后缀
// 0xFF00 → 0000FF00-0000-1000-8000-00805f9b34fb
// 0xFF01 → 0000FF01-0000-1000-8000-00805f9b34fb (健康数据)
// 0xFF02 → 0000FF02-0000-1000-8000-00805f9b34fb (中文消息)
const ESP32_CONFIG = {
  SERVICE_UUID: '0000FF00-0000-1000-8000-00805f9b34fb',
  CHARACTERISTIC_UUID: '0000FF01-0000-1000-8000-00805f9b34fb',
  CHARACTERISTIC2_UUID: '0000FF02-0000-1000-8000-00805f9b34fb',
  DEVICE_NAME_FILTER: 'ESP32_BLE'
};

// 蓝牙状态
const isConnected = ref(false);
const isScanning = ref(false);
const statusMsg = ref('准备连接设备');
const healthData = ref({ temp: '--', heart: '--', steps: '--', spo2: '--' });

// 设备管理
const devices = ref([]);
const currentDevice = ref(null);
const deviceServices = ref([]);
const deviceCharacteristics = ref([]);

// 消息处理变量
const latestMessage = ref('等待设备发送消息...');
const msgTime = ref('');
const messageBuffer = ref([]);  // 中文消息缓冲区
const bufferTimeout = ref(null);  // 超时定时器
let isBLEListenerRegistered = false;  // 标志：是否已注册BLE监听器
let isDeviceFoundListenerRegistered = false;  // 标志：是否已注册设备发现监听器
let isConnecting = false;  // 防止重复连接

// 联系人数据逻辑
const showModal = ref(false);
const contacts = ref([
  { name: '张医生', phone: '13800138000' },
  { name: '', phone: '' },
  { name: '', phone: '' }
]);
const tempContacts = ref([]);

// 初始化蓝牙适配器
const initBluetooth = () => {
  initBLEListener();
  uni.openBluetoothAdapter({
    success: (res) => {
      console.log('蓝牙适配器初始化成功', res);
      statusMsg.value = '蓝牙就绪，点击连接开始扫描';
      startScan();
    },
    fail: (err) => {
      console.error('蓝牙适配器初始化失败', err);
      statusMsg.value = '蓝牙初始化失败，请检查权限';
      uni.showToast({ title: '蓝牙初始化失败', icon: 'error' });
    }
  });
};

const startScan = () => {
  isScanning.value = true;
  devices.value = [];
  statusMsg.value = '正在扫描设备...';
  
  // 只注册一次设备发现监听器
  if (!isDeviceFoundListenerRegistered) {
    uni.onBluetoothDeviceFound(handleDeviceFound);
    isDeviceFoundListenerRegistered = true;
  }
  
  uni.startBluetoothDevicesDiscovery({
    allowDuplicatesKey: false,
    success: (res) => {
      console.log('开始扫描设备', res);
      setTimeout(stopScan, 10000);
    },
    fail: (err) => {
      console.error('扫描设备失败', err);
      isScanning.value = false;
      statusMsg.value = '扫描失败';
      uni.showToast({ title: '扫描失败', icon: 'error' });
    }
  });
};

// 停止扫描
const stopScan = () => {
  uni.stopBluetoothDevicesDiscovery({
    success: () => {
      console.log('停止扫描设备');
      isScanning.value = false;
      if (devices.value.length === 0) {
        statusMsg.value = '未找到设备，点击重试';
      } else {
        statusMsg.value = '选择设备或点击重试';
      }
    }
  });
};

// 信号强度分类
const getSignalClass = (rssi) => {
  if (!rssi || rssi < -70) return 'signal-weak';
  if (rssi < -50) return 'signal-medium';
  return 'signal-strong';
};

// 处理发现的设备
const handleDeviceFound = (res) => {
  const newDevices = res.devices;
  console.log('=== 发现设备事件 ===');
  console.log('原始数据:', JSON.stringify(newDevices));
  console.log('设备数量:', newDevices.length);
  
  newDevices.forEach(device => {
    console.log('--- 单个设备信息 ---');
    console.log('name:', device.name);
    console.log('advertiseName:', device.advertiseName);
    console.log('deviceId:', device.deviceId);
    console.log('RSSI:', device.RSSI);
    console.log('localName:', device.localName);
    
    // 检查是否是ESP32设备（通过名称过滤）
    const deviceName = device.name || device.advertiseName || device.localName || '';
    const isESP32 = deviceName.includes(ESP32_CONFIG.DEVICE_NAME_FILTER);
    
    console.log('检查设备名:', deviceName, '是否匹配:', isESP32);
    
    if (isESP32) {
      const exists = devices.value.find(d => d.deviceId === device.deviceId);
      if (!exists) {
        devices.value.push(device);
        console.log('✓ 发现ESP32设备:', device.name, 'RSSI:', device.RSSI);
        
        // 如果只有一个设备，自动选择
        if (devices.value.length === 1) {
          selectDevice(device);
        }
      }
    }
  });
};

// 选择设备
const selectDevice = (device) => {
  currentDevice.value = device;
  statusMsg.value = `已选择: ${device.name}`;
  connectToDevice();
};

// 连接设备
const connectToDevice = () => {
  if (!currentDevice.value) {
    uni.showToast({ title: '请先选择设备', icon: 'none' });
    return;
  }
  
  // 防止重复连接
  if (isConnecting) {
    console.log('正在连接中，跳过重复操作');
    return;
  }
  
  isConnecting = true;
  statusMsg.value = `正在连接 ${currentDevice.value.name}...`;
  
  uni.createBLEConnection({
    deviceId: currentDevice.value.deviceId,
    success: (res) => {
      console.log('设备连接成功', res);
      isConnected.value = true;
      isConnecting = false;
      statusMsg.value = `已连接: ${currentDevice.value.name}`;
      
      // 获取设备服务
      getDeviceServices();
    },
    fail: (err) => {
      console.error('设备连接失败', err);
      isConnecting = false;
      statusMsg.value = '连接失败，请重试';
      uni.showToast({ title: '连接失败', icon: 'error' });
    }
  });
};

// 获取设备服务
const getDeviceServices = () => {
  uni.getBLEDeviceServices({
    deviceId: currentDevice.value.deviceId,
    success: (res) => {
      console.log('获取服务成功', res.services);
      deviceServices.value = res.services;
      
      // 查找目标服务
      const targetService = res.services.find(
        service => service.uuid.toLowerCase() === ESP32_CONFIG.SERVICE_UUID.toLowerCase()
      );
      
      if (targetService) {
        getDeviceCharacteristics(targetService.uuid);
      } else {
        console.error('未找到目标服务');
        uni.showToast({ title: '未找到目标服务', icon: 'error' });
      }
    },
    fail: (err) => {
      console.error('获取服务失败', err);
    }
  });
};

// 获取设备特征值
const getDeviceCharacteristics = (serviceId) => {
  uni.getBLEDeviceCharacteristics({
    deviceId: currentDevice.value.deviceId,
    serviceId: serviceId,
    success: (res) => {
      console.log('获取特征值成功', res.characteristics);
      deviceCharacteristics.value = res.characteristics;
      
      // 查找特征值1（健康数据）
      const healthChar = res.characteristics.find(
        char => char.uuid.toLowerCase() === ESP32_CONFIG.CHARACTERISTIC_UUID.toLowerCase()
      );
      
      // 查找特征值2（中文消息）
      const msgChar = res.characteristics.find(
        char => char.uuid.toLowerCase() === ESP32_CONFIG.CHARACTERISTIC2_UUID.toLowerCase()
      );
      
      if (healthChar) {
        if (healthChar.properties.notify) {
          enableNotifications(serviceId, healthChar.uuid, 'health');
        }
        if (healthChar.properties.write) {
          console.log('找到健康数据写入特征值');
        }
      }
      
      if (msgChar) {
        if (msgChar.properties.notify) {
          enableNotifications(serviceId, msgChar.uuid, 'message');
        }
        if (msgChar.properties.write) {
          console.log('找到消息写入特征值');
        }
      }
      
      if (!healthChar && !msgChar) {
        console.error('未找到目标特征值');
        uni.showToast({ title: '未找到特征值', icon: 'error' });
      }
    },
    fail: (err) => {
      console.error('获取特征值失败', err);
    }
  });
};

// 启用特征值通知
const enableNotifications = (serviceId, characteristicId, type) => {
  uni.notifyBLECharacteristicValueChange({
    deviceId: currentDevice.value.deviceId,
    serviceId: serviceId,
    characteristicId: characteristicId,
    state: true,
    success: () => {
      console.log(`启用${type === 'health' ? '健康数据' : '消息'}通知成功`);
    },
    fail: (err) => {
      console.error('启用通知失败', err);
    }
  });
};

// 只注册一次BLE特征值变化监听
const initBLEListener = () => {
  if (isBLEListenerRegistered) {
    console.log('BLE监听器已注册，跳过重复注册');
    return;
  }
  
  uni.onBLECharacteristicValueChange((res) => {
    const charId = res.characteristicId.toLowerCase();
    const healthUUID = ESP32_CONFIG.CHARACTERISTIC_UUID.toLowerCase();
    const msgUUID = ESP32_CONFIG.CHARACTERISTIC2_UUID.toLowerCase();
    
    if (charId === healthUUID) {
      handleCharacteristicValueChange(res, 'health');
    } else if (charId === msgUUID) {
      handleCharacteristicValueChange(res, 'message');
    }
  });
  
  isBLEListenerRegistered = true;
  console.log('BLE监听器已注册');
};

// 处理特征值变化
const handleCharacteristicValueChange = (res, type) => {
  console.log(`收到${type === 'health' ? '健康数据' : '消息'}特征值变化`, res);
  
  // 将ArrayBuffer转换为字节数组
  const uint8Array = new Uint8Array(res.value);
  
  if (type === 'health') {
    // 健康数据：直接处理（数值数据不受编码影响）
    const value = ab2str(res.value);
    console.log('接收到的数据:', value);
    processReceivedData(value);
  } else {
    // 中文消息：使用缓冲机制处理分包
    console.log('收到消息字节:', Array.from(uint8Array).map(b => b.toString(16)).join(' '));
    
    // 检查是否包含逗号（健康数据的特征）
    let hasComma = false;
    for (let i = 0; i < uint8Array.length; i++) {
      if (uint8Array[i] === 0x2C) {  // 逗号 ASCII
        hasComma = true;
        break;
      }
    }
    
    if (hasComma) {
      // 这是健康数据，不是消息，清空之前的消息缓冲区
      console.log('检测到逗号，这是健康数据，清空消息缓冲区');
      messageBuffer.value = [];
      if (bufferTimeout.value) {
        clearTimeout(bufferTimeout.value);
        bufferTimeout.value = null;
      }
      return;
    }
    
    // 将新字节添加到缓冲区
    for (let i = 0; i < uint8Array.length; i++) {
      messageBuffer.value.push(uint8Array[i]);
    }
    
    // 清除之前的超时定时器
    if (bufferTimeout.value) {
      clearTimeout(bufferTimeout.value);
    }
    
    // 检查最后是否有未完成的UTF-8字节
    const lastByte = uint8Array[uint8Array.length - 1];
    const needsMoreBytes = (lastByte >= 0x80 && lastByte < 0xC0) ||
                          (lastByte >= 0xC0 && lastByte < 0xE0) ||
                          (lastByte >= 0xE0 && lastByte < 0xF0);
    
    // 设置超时定时器（如果有不完整字节则等待更长时间）
    const timeoutMs = needsMoreBytes ? 800 : 400;
    
    bufferTimeout.value = setTimeout(() => {
      // 尝试解码完整消息
      const completeBuffer = new Uint8Array(messageBuffer.value);
      const message = ab2str(completeBuffer.buffer);
      console.log('完整消息:', message);
      receiveEsp32Message(message);
      
      // 清空缓冲区
      messageBuffer.value = [];
      bufferTimeout.value = null;
    }, timeoutMs);
  }
};

// 发送测试消息
const sendTestMessage = () => {
  // 可以发送一些初始化命令或查询状态
  const message = "HELLO";
  const buffer = str2ab(message);
  
  uni.writeBLECharacteristicValue({
    deviceId: currentDevice.value.deviceId,
    serviceId: ESP32_CONFIG.SERVICE_UUID,
    characteristicId: ESP32_CONFIG.CHARACTERISTIC_UUID,
    value: buffer,
    success: () => {
      console.log('发送消息成功:', message);
    },
    fail: (err) => {
      console.error('发送消息失败', err);
    }
  });
};

// 处理接收到的数据 - 简化版
const processReceivedData = (data) => {
  console.log('home.vue 处理数据:', data);
  
  // 支持两种格式：
  // 1. JSON格式: {"temp": 36.5, "heart": 75, "steps": 1234}
  // 2. 逗号分隔: 72,100,37.0 (心率,步数,体温)
  
  let parsedData = null;
  
  try {
    parsedData = JSON.parse(data);
  } catch (e) {
    const parts = data.split(',');
    if (parts.length >= 3) {
      const heart = parseInt(parts[0].trim());
      const steps = parseInt(parts[1].trim());
      const temp = parseFloat(parts[2].trim());
      const spo2 = parts.length >= 4 ? parseInt(parts[3].trim()) : undefined;
      
      if (!isNaN(heart) && !isNaN(steps) && !isNaN(temp)) {
        parsedData = { heart, steps, temp };
        if (!isNaN(spo2)) parsedData.spo2 = spo2;
      }
    }
  }
  
  if (parsedData) {
    if (parsedData.temp !== undefined) healthData.value.temp = parsedData.temp.toFixed(1);
    if (parsedData.heart !== undefined) healthData.value.heart = parsedData.heart;
    if (parsedData.steps !== undefined) healthData.value.steps = parsedData.steps;
    if (parsedData.spo2 !== undefined) healthData.value.spo2 = parsedData.spo2;
    
    // 直接发送事件，不做任何去重，因为 data.vue 已经会处理
    console.log('home.vue 发送事件:', parsedData);
    uni.$emit('healthDataUpdate', parsedData);
    uni.setStorageSync('globalHealthData', JSON.stringify(parsedData));
  }
};

// 检查健康警报
const disconnectDevice = () => {
  if (currentDevice.value) {
    uni.closeBLEConnection({
      deviceId: currentDevice.value.deviceId,
      success: () => {
        console.log('设备断开成功');
        resetConnection();
      },
      fail: (err) => {
        console.error('设备断开失败', err);
        resetConnection();
      }
    });
  } else {
    resetConnection();
  }
};

// 重置连接状态
const resetConnection = () => {
  isConnected.value = false;
  isScanning.value = false;
  currentDevice.value = null;
  devices.value = [];
  deviceServices.value = [];
  deviceCharacteristics.value = [];
  statusMsg.value = '设备已断开';
  latestMessage.value = '等待设备发送消息...';
  msgTime.value = '';
  healthData.value = { temp: '--', heart: '--', steps: '--', spo2: '--' };
  
  // 移除蓝牙事件监听器，防止重新连接时重复触发
  uni.offBLECharacteristicValueChange();
  uni.offBluetoothDeviceFound();
  
  // 重置监听器注册标志，允许重新注册
  isBLEListenerRegistered = false;
  isDeviceFoundListenerRegistered = false;
  console.log('连接已重置，监听器已清理');
};

// 工具函数：ArrayBuffer转字符串（支持UTF-8中文）
const ab2str = (buf) => {
  const uint8Array = new Uint8Array(buf);
  let result = '';
  let i = 0;
  
  while (i < uint8Array.length) {
    const byte = uint8Array[i];
    
    if (byte < 0x80) {
      // ASCII字符
      result += String.fromCharCode(byte);
      i++;
    } else if (byte < 0xC0) {
      // 无效的延续字节，跳过
      i++;
    } else if (byte < 0xE0) {
      // 2字节序列 (0xC0-0xDF)
      if (i + 1 < uint8Array.length && (uint8Array[i + 1] & 0xC0) === 0x80) {
        const code = ((byte & 0x1F) << 6) | (uint8Array[i + 1] & 0x3F);
        result += String.fromCharCode(code);
        i += 2;
      } else {
        // 无效序列，跳过
        i++;
      }
    } else if (byte < 0xF0) {
      // 3字节序列 (0xE0-0xEF) - 中文常用
      if (i + 2 < uint8Array.length && 
          (uint8Array[i + 1] & 0xC0) === 0x80 && 
          (uint8Array[i + 2] & 0xC0) === 0x80) {
        const code = ((byte & 0x0F) << 12) |
                     ((uint8Array[i + 1] & 0x3F) << 6) |
                     (uint8Array[i + 2] & 0x3F);
        result += String.fromCharCode(code);
        i += 3;
      } else {
        // 无效序列，跳过
        i++;
      }
    } else if (byte < 0xF8) {
      // 4字节序列 (0xF0-0xF7)
      if (i + 3 < uint8Array.length &&
          (uint8Array[i + 1] & 0xC0) === 0x80 &&
          (uint8Array[i + 2] & 0xC0) === 0x80 &&
          (uint8Array[i + 3] & 0xC0) === 0x80) {
        const code = ((byte & 0x07) << 18) |
                     ((uint8Array[i + 1] & 0x3F) << 12) |
                     ((uint8Array[i + 2] & 0x3F) << 6) |
                     (uint8Array[i + 3] & 0x3F);
        result += String.fromCharCode(code);
        i += 4;
      } else {
        // 无效序列，跳过
        i++;
      }
    } else {
      // 5或6字节序列，不支持，跳过
      i++;
    }
  }
  
  return result;
};

// 工具函数：字符串转ArrayBuffer
const str2ab = (str) => {
  const buf = new ArrayBuffer(str.length);
  const bufView = new Uint8Array(buf);
  for (let i = 0; i < str.length; i++) {
    bufView[i] = str.charCodeAt(i);
  }
  return buf;
};

// 接收消息的函数
const receiveEsp32Message = (newMsg) => {
  latestMessage.value = newMsg;
  const now = new Date();
  msgTime.value = `${now.getHours()}:${now.getMinutes().toString().padStart(2, '0')}:${now.getSeconds().toString().padStart(2, '0')}`;
  
  uni.vibrateShort();
  autoCallEmergency(newMsg);
};

const autoCallEmergency = (msg) => {
  const validContacts = contacts.value.filter(c => c.name && c.phone);
  if (validContacts.length === 0) return;
  
  const target = validContacts[0];
  uni.showModal({
    title: '紧急呼叫',
    content: `收到设备消息: "${msg}"，是否立即呼叫 ${target.name}(${target.phone})？`,
    confirmText: '立即呼叫',
    cancelText: '取消',
    success: (res) => {
      if (res.confirm) {
        uni.makePhoneCall({ phoneNumber: target.phone });
      }
    }
  });
};

// 联系人管理函数
const openEditModal = () => {
  tempContacts.value = JSON.parse(JSON.stringify(contacts.value));
  showModal.value = true;
};

const saveSettings = () => {
  contacts.value = JSON.parse(JSON.stringify(tempContacts.value));
  uni.setStorageSync('health_contacts', JSON.stringify(contacts.value));
  showModal.value = false;
  uni.showToast({ title: '设置已更新' });
};

const makeCall = (phone) => {
  if (!phone) return;
  uni.makePhoneCall({ phoneNumber: phone });
};

// 主连接/断开函数
const toggleConnect = () => {
  if (isConnected.value) {
    disconnectDevice();
  } else {
    initBluetooth();
  }
};

// 组件挂载
onMounted(() => {
  const saved = uni.getStorageSync('health_contacts');
  if (saved) {
    contacts.value = JSON.parse(saved);
  }
});

// 组件卸载时清理
onUnmounted(() => {
  if (isConnected.value) {
    disconnectDevice();
  }
  uni.closeBluetoothAdapter();
});
</script>

<style scoped>
/* 保持原有样式不变 */
.container { position: relative; width: 100%; min-height: 100vh; background: linear-gradient(135deg, #e0f7e0 0%, #c8e6c9 100%); }
.content-wrapper { padding: 90rpx 30rpx; padding-bottom: 120rpx; position: relative; z-index: 1; }
.header { text-align: center; margin-bottom: 40rpx; }
.title { font-size: 48rpx; font-weight: bold; color: #2e7d32; display: block; }
.subtitle { font-size: 26rpx; color: #66bb6a; margin-top: 10rpx; display: block; }
.connect-btn { background: #2e7d32; color: white; border-radius: 40rpx; font-size: 26rpx; display: inline-block; padding: 0 40rpx; }
.btn-connected { background: #c62828; }

/* 设备列表样式 */
.device-list-section {
  background: rgba(255, 255, 255, 0.5);
  border-radius: 20rpx;
  padding: 25rpx;
  margin-bottom: 30rpx;
  backdrop-filter: blur(5px);
  border: 1px solid rgba(46, 125, 50, 0.1);
}

/* 扫描状态 */
.scan-status {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 30rpx 0;
  margin-bottom: 20rpx;
}
.scan-animation {
  display: flex;
  gap: 10rpx;
  margin-bottom: 15rpx;
}
.scan-dot {
  width: 16rpx;
  height: 16rpx;
  background: #2e7d32;
  border-radius: 50%;
  animation: scanPulse 1s ease-in-out infinite;
}
.scan-dot:nth-child(2) { animation-delay: 0.2s; }
.scan-dot:nth-child(3) { animation-delay: 0.4s; }
@keyframes scanPulse {
  0%, 100% { opacity: 0.3; transform: scale(0.8); }
  50% { opacity: 1; transform: scale(1.2); }
}
.scan-text {
  font-size: 24rpx;
  color: #666;
}

/* 设备列表头部 */
.device-list-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15rpx;
}
.device-list-title {
  font-size: 28rpx;
  font-weight: bold;
  color: #2e7d32;
}
.device-actions {
  display: flex;
  gap: 15rpx;
}
.action-btn {
  font-size: 24rpx;
  color: #2e7d32;
  background: rgba(46, 125, 50, 0.1);
  padding: 8rpx 20rpx;
  border-radius: 20rpx;
}
.action-btn:active {
  background: rgba(46, 125, 50, 0.2);
}

/* 设备列表 */
.device-list {
  display: flex;
  flex-direction: column;
  gap: 12rpx;
}
.device-item {
  background: rgba(255, 255, 255, 0.9);
  border-radius: 15rpx;
  padding: 20rpx;
  display: flex;
  align-items: center;
  border: 1px solid #e0e0e0;
  transition: all 0.2s ease;
}
.device-item.selected {
  background: #e8f5e9;
  border-color: #2e7d32;
}
.device-item:active {
  background: #f1f8e9;
  transform: scale(0.98);
}

/* 设备图标 */
.device-icon {
  width: 70rpx;
  height: 70rpx;
  background: linear-gradient(135deg, #e0f7e0 0%, #c8e6c9 100%);
  border-radius: 50%;
  display: flex;
  justify-content: center;
  align-items: center;
  margin-right: 20rpx;
}
.icon-bluetooth {
  font-size: 32rpx;
}

/* 设备信息 */
.device-info {
  flex: 1;
  display: flex;
  flex-direction: column;
}
.device-name {
  font-size: 28rpx;
  font-weight: bold;
  color: #333;
  margin-bottom: 6rpx;
}
.device-id {
  font-size: 20rpx;
  color: #999;
}

/* 信号强度 */
.device-rssi {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 10rpx 15rpx;
  border-radius: 10rpx;
  min-width: 80rpx;
}
.device-rssi.signal-strong {
  background: rgba(76, 175, 80, 0.1);
}
.device-rssi.signal-medium {
  background: rgba(255, 152, 0, 0.1);
}
.device-rssi.signal-weak {
  background: rgba(244, 67, 54, 0.1);
}
.rssi-value {
  font-size: 28rpx;
  font-weight: bold;
  color: #333;
}
.signal-strong .rssi-value { color: #4caf50; }
.signal-medium .rssi-value { color: #ff9800; }
.signal-weak .rssi-value { color: #f44336; }
.rssi-unit {
  font-size: 18rpx;
  color: #666;
}

/* 空状态 */
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 40rpx 0;
}
.empty-icon {
  font-size: 60rpx;
  margin-bottom: 20rpx;
}
.empty-text {
  font-size: 28rpx;
  color: #666;
  margin-bottom: 10rpx;
}
.empty-hint {
  font-size: 22rpx;
  color: #999;
}

.main-card {
  background: rgba(255, 255, 255, 0.6); border-radius: 24rpx; padding: 30rpx 20rpx;
  backdrop-filter: blur(5px);
  margin-bottom: 40rpx; box-shadow: 0 8rpx 32rpx rgba(46, 125, 50, 0.1);
}
.data-grid { display: flex; flex-wrap: wrap; justify-content: space-around; }
.data-item { display: flex; flex-direction: column; align-items: center; width: 50%; margin-bottom: 20rpx; }
.data-value { font-size: 50rpx; font-weight: bold; }
.temp-color { color: #ff7043; }
.heart-color { color: #e91e63; }
.steps-color { color: #2196f3; }
.spo2-color { color: #9c27b0; }
.data-label { font-size: 24rpx; color: #666; }

.emergency-section {
  background: rgba(255, 255, 255, 0.4); border-radius: 24rpx; padding: 30rpx;
  margin-bottom: 30rpx; /* 增加下边距给消息模块 */
}
.title-row { display: flex; justify-content: space-between; align-items: center; }
.emergency-title { font-size: 34rpx; font-weight: bold; color: #c62828; }
.edit-tag { font-size: 24rpx; color: #2e7d32; background: rgba(255,255,255,0.8); padding: 4rpx 16rpx; border-radius: 20rpx; }
.emergency-desc { font-size: 22rpx; color: #888; margin-top: 6rpx; display: block; }

.contact-list { margin-top: 20rpx; }
.contact-card {
  background: white; border-radius: 16rpx; padding: 20rpx;
  display: flex; align-items: center; margin-bottom: 15rpx;
}
.contact-avatar { width: 70rpx; height: 70rpx; background: #e8f5e9; border-radius: 50%; display: flex; justify-content: center; align-items: center; margin-right: 20rpx; }
.avatar-text { font-size: 30rpx; color: #2e7d32; font-weight: bold; }
.contact-info { flex: 1; }
.contact-name { font-size: 28rpx; font-weight: bold; }
.contact-phone { font-size: 24rpx; color: #888; }
.call-btn { color: #c62828; font-size: 36rpx; }

/* --- 新增：实时消息模块样式 --- */
.message-section {
  background: rgba(255, 255, 255, 0.5);
  border-radius: 24rpx;
  padding: 30rpx;
  backdrop-filter: blur(5px);
  box-shadow: 0 4rpx 20rpx rgba(0,0,0,0.05);
}
.message-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20rpx;
}
.message-title {
  font-size: 30rpx;
  font-weight: bold;
  color: #333;
}
.message-time {
  font-size: 20rpx;
  color: #999;
}
.message-box {
  background: rgba(255, 255, 255, 0.8);
  border-radius: 16rpx;
  padding: 30rpx;
  min-height: 100rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 1px dashed #ddd;
  transition: all 0.3s ease;
}
.message-box.has-msg {
  border: 1px solid #66bb6a;
  background: #f1f8e9;
}
.message-text {
  font-size: 28rpx;
  color: #666;
  text-align: center;
}
.has-msg .message-text {
  color: #2e7d32;
  font-weight: 500;
}

/* 弹窗样式 (保持不变) */
.modal-mask { position: fixed; top: 0; left: 0; right: 0; bottom: 0; background: rgba(0,0,0,0.5); display: flex; justify-content: center; align-items: center; z-index: 999; }
.modal-content { width: 80%; background: white; border-radius: 30rpx; padding: 40rpx; max-height: 85vh; }
.modal-title { font-size: 32rpx; font-weight: bold; text-align: center; margin-bottom: 30rpx; display: block; }
.modal-scroll { max-height: 50vh; margin-bottom: 30rpx; }
.edit-group { margin-bottom: 25rpx; padding: 15rpx; background: #f9f9f9; border-radius: 15rpx; }
.group-label { font-size: 22rpx; color: #2e7d32; font-weight: bold; margin-bottom: 10rpx; display: block; }
.modal-input { background: #fff; height: 70rpx; border: 1rpx solid #eee; border-radius: 10rpx; padding: 0 20rpx; margin-bottom: 10rpx; font-size: 26rpx; }
.modal-btns { display: flex; gap: 20rpx; }
.modal-btn { flex: 1; font-size: 28rpx; border-radius: 40rpx; }
.confirm { background: #2e7d32; color: white; }
</style>