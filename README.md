<div align="center">

# 🫀 HealthWatch

**把 ESP32 采集到的健康数据，变成清晰、及时、可行动的移动端信息。**

一款基于 **ESP32 BLE + uni-app + Vue 3** 的移动健康监测应用。  
支持体温、心率、步数与血氧实时展示，提供趋势图、历史记录、数据导出和紧急联系人呼叫能力。

[![Vue 3](https://img.shields.io/badge/Vue-3-42b883?style=flat-square&logo=vuedotjs&logoColor=white)](https://vuejs.org/)
[![uni-app](https://img.shields.io/badge/uni--app-Mobile-2b9939?style=flat-square)](https://uniapp.dcloud.net.cn/)
[![ESP32](https://img.shields.io/badge/ESP32-BLE-e7352c?style=flat-square&logo=espressif&logoColor=white)](https://www.espressif.com/)
[![Android](https://img.shields.io/badge/Android-Supported-3ddc84?style=flat-square&logo=android&logoColor=white)](https://developer.android.com/)
[![Status](https://img.shields.io/badge/status-prototype-4f86f7?style=flat-square)](#项目状态)

[功能亮点](#功能亮点) · [界面预览](#界面预览) · [快速开始](#快速开始) · [设备协议](docs/DEVICE_PROTOCOL.md) · [技术架构](docs/ARCHITECTURE.md)

</div>

## 项目简介

HealthWatch 面向可穿戴设备、个人健康监护与传感器实验场景。应用通过 BLE 扫描并连接名为 `ESP32_BLE` 的设备，接收健康数据后在移动端完成解析、展示和短期趋势分析；当设备发送紧急消息时，应用会振动提醒，并在用户确认后拨打预设联系人。

> 本项目是软硬件联调原型，不是医疗器械，也不能替代专业诊断或紧急医疗服务。

## 功能亮点

| 模块 | 能力 |
| --- | --- |
| 📡 BLE 连接 | 扫描 ESP32 设备、展示信号强度、建立连接、订阅特征值通知 |
| 🫀 实时监测 | 同屏展示体温、心率、今日步数与血氧四类指标 |
| 📈 趋势分析 | Canvas 动态折线图、自适应坐标轴、最大值/最小值/平均值统计 |
| 🕘 历史记录 | 每类指标保留最近 50 个采样点，可单独查看与清除 |
| 📋 数据导出 | 将当前指标的时间序列整理为 CSV 格式并复制到剪贴板 |
| ☎️ 紧急呼叫 | 本地维护紧急联系人；收到设备消息后振动并请求确认拨号 |
| 🔐 本地优先 | 联系人和最近一次读数保存在设备本地，不依赖远程账号或云服务 |

## 界面预览

<table>
  <tr>
    <td align="center"><strong>应用界面</strong></td>
    <td align="center"><strong>设备与实时指标</strong></td>
    <td align="center"><strong>趋势与历史记录</strong></td>
  </tr>
  <tr>
    <td><img src="https://github.com/user-attachments/assets/47933d50-a19c-4d27-b3f2-8176393a7226" width="250" alt="HealthWatch 应用界面"></td>
    <td><img src="https://github.com/user-attachments/assets/1fbe43cc-2575-4b32-8ab9-e515679d79d6" width="250" alt="HealthWatch 设备与实时指标界面"></td>
    <td><img src="https://github.com/user-attachments/assets/e4611550-c8bd-4628-8f63-3ffc7acb44d2" width="250" alt="HealthWatch 数据趋势界面"></td>
  </tr>
</table>

## 工作流程

```text
ESP32 传感器
    │  BLE Notify
    ▼
HealthWatch 连接与数据解析
    ├── 实时指标卡片
    ├── 趋势图与统计
    ├── 最近 50 条内存记录
    └── 紧急消息 → 振动提醒 → 用户确认 → 拨号
```

更完整的模块关系见 [技术架构](docs/ARCHITECTURE.md)。

## 数据协议

应用支持两种健康数据格式：

```json
{"temp":36.5,"heart":75,"steps":1234,"spo2":98}
```

```text
75,1234,36.5,98
心率,步数,体温,血氧
```

默认设备名、Service UUID 和 Characteristic UUID 见 [设备接入协议](docs/DEVICE_PROTOCOL.md)。实际硬件 UUID 不同时，需要同步修改 `pages/home/home.vue` 中的 `ESP32_CONFIG`。

## 快速开始

### 环境要求

- [HBuilderX](https://www.dcloud.io/hbuilderx.html)，建议使用包含 App 开发能力的版本
- 支持 BLE 的 Android 真机
- ESP32 或 ESP32-S3 设备
- 已开启手机蓝牙和定位权限

### 运行应用

1. 克隆仓库：

   ```bash
   git clone https://github.com/Pigpig22/HealthWatch.git
   ```

2. 使用 HBuilderX 打开项目目录。
3. 根据 [设备接入协议](docs/DEVICE_PROTOCOL.md) 配置 ESP32 的设备名与 UUID。
4. 选择“运行 → 运行到手机或模拟器 → Android App 基座”。
5. 在真机上允许蓝牙、附近设备、定位和电话权限。
6. 打开应用，扫描并选择 `ESP32_BLE` 设备。

> BLE、拨号等能力依赖原生运行环境，普通浏览器预览无法完整验证。

## 项目结构

```text
HealthWatch/
├── App.vue                      # 应用入口与全局生命周期
├── main.js                      # Vue / uni-app 启动入口
├── manifest.json                # App 模块、权限与平台配置
├── pages.json                   # 页面路由和底部导航
├── pages/
│   ├── home/home.vue            # BLE 连接、实时指标、联系人与紧急呼叫
│   └── data/data.vue            # 趋势图、统计、历史记录与导出
├── static/                      # Logo 与导航图标
├── docs/
│   ├── ARCHITECTURE.md          # 技术架构与数据流
│   └── DEVICE_PROTOCOL.md       # ESP32 BLE 接入协议
└── CONTRIBUTING.md              # 贡献指南
```

## 技术栈

- **Vue 3 Composition API**：页面状态与业务逻辑
- **uni-app**：跨端页面、原生 BLE、存储、振动和拨号 API
- **Canvas 2D**：实时折线图和自适应坐标轴
- **ESP32 BLE / NimBLE**：传感数据和紧急消息传输
- **HBuilderX**：真机调试与 Android 打包

## 数据与隐私

- 紧急联系人通过 `uni.setStorageSync` 保存在本机。
- 最近一次健康读数保存在本机，用于页面间同步。
- 趋势记录当前仅保存在运行内存中，每项最多 50 条；退出或刷新后不保证保留。
- 导出功能会生成 CSV 文本并复制到系统剪贴板，不会自动上传服务器。
- 仓库代码未实现云端账号、远程数据库或健康数据上传。

请只授予实际需要的系统权限，并避免在公开演示时使用真实联系人号码或敏感健康数据。

## 项目状态

当前版本是可运行的软硬件联调原型，重点验证 BLE 数据链路、移动端可视化和紧急消息闭环。后续可扩展：

- 数据持久化与按日期查询
- 异常阈值和分级告警
- 后台持续采集与断线重连
- 数据文件分享或云端同步
- 自动化测试与正式安装包发布

## 参与贡献

欢迎提交 Bug、功能建议或改进方案。开始前请阅读 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 免责声明

本项目仅用于学习、原型验证和工程实践。显示的数据可能受到传感器精度、佩戴方式、通信质量和软件状态影响，不应直接用于医疗诊断、治疗决策或替代当地紧急救援服务。
