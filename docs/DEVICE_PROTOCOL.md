# ESP32 BLE 设备接入协议

HealthWatch 默认连接名为 `ESP32_BLE` 的设备，并订阅两个 Notify 特征值。硬件端配置不一致时，请同时修改 App 的 `ESP32_CONFIG`。

## 默认标识

| 项目 | 默认值 |
| --- | --- |
| 设备名过滤 | `ESP32_BLE` |
| Service UUID | `0000FF00-0000-1000-8000-00805f9b34fb` |
| 健康数据 Characteristic | `0000FF01-0000-1000-8000-00805f9b34fb` |
| 消息 Characteristic | `0000FF02-0000-1000-8000-00805f9b34fb` |

ESP32 使用 NimBLE 时，16 位 UUID 通常会扩展为上表所示的 Bluetooth Base UUID。

## 健康数据

健康数据特征值应通过 Notify 发送 UTF-8 文本。支持以下两种格式。

### JSON

```json
{"temp":36.5,"heart":75,"steps":1234,"spo2":98}
```

字段：

| 字段 | 类型 | 含义 | 示例 |
| --- | --- | --- | --- |
| `temp` | Number | 体温，摄氏度 | `36.5` |
| `heart` | Integer | 心率，bpm | `75` |
| `steps` | Integer | 今日步数 | `1234` |
| `spo2` | Integer | 血氧饱和度，百分比 | `98` |

### 逗号分隔

```text
75,1234,36.5,98
```

顺序固定为：心率、步数、体温、血氧。前三项必需，血氧可省略。

## 紧急消息

消息特征值可通过 Notify 发送 UTF-8 文本，例如：

```text
检测到跌倒，请确认安全状态
```

App 收到消息后会振动并显示呼叫确认框。消息内容应简短，不要包含二进制数据。

## 联调检查表

- 手机能扫描到 `ESP32_BLE`；
- Service 与 Characteristic UUID 完全一致；
- 两个特征值均支持 Notify；
- 数据使用 UTF-8 编码；
- JSON 数值字段不是带单位的字符串；
- 采样频率适中，避免高频通知阻塞界面；
- 断线后 ESP32 能重新广播；
- 使用测试号码验证紧急呼叫流程。

## 修改配置

在 `pages/home/home.vue` 中查找：

```js
const ESP32_CONFIG = {
  SERVICE_UUID: '0000FF00-0000-1000-8000-00805f9b34fb',
  CHARACTERISTIC_UUID: '0000FF01-0000-1000-8000-00805f9b34fb',
  CHARACTERISTIC2_UUID: '0000FF02-0000-1000-8000-00805f9b34fb',
  DEVICE_NAME_FILTER: 'ESP32_BLE'
}
```

修改后需要重新运行或打包 App。
