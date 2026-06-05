# PCT-100-23 智能环境监控系统

基于 ESP32-C3 的环境监测与自动控制系统，集成光照/温度传感器、双路继电器输出、OLED 显示屏、RGB 状态指示灯，支持 WiFi 无线连接与 MQTT 远程测控。

## 功能概览

| 模块 | 说明 |
|------|------|
| 🌡️ 温度监测 | DS18B20 数字温度传感器，自动控制风扇继电器 |
| 💡 光照监测 | 模拟量光敏传感器，自动控制灯光继电器 |
| 📺 OLED 显示 | SSD1306 128×64 I²C 屏，中文界面，实时展示所有状态 |
| 💡 RGB 指示灯 | WS2812 集成 LED，6 种颜色/闪烁模式直观反馈系统状态 |
| 📶 WiFi | 2.4GHz STA 模式，串口选网，密码/凭据自动保存 Flash |
| 🔗 MQTT | 远程状态上报 + 指令控制，支持 JSON 格式双向通信 |
| 🔘 双按键 | KEY1 总闸开关，KEY2 模式切换 / 手动四态循环 |

## 快速开始

### 1. 硬件连接

参见 [硬件连接说明](./hardware.md)。

### 2. 编译上传

使用 Arduino IDE 或 PlatformIO，需安装以下库：

- **U8g2** (olikraus/U8g2) — OLED 显示屏
- **Adafruit NeoPixel** — WS2812 RGB LED
- **OneWire** + **DallasTemperature** — DS18B20 温度传感器
- **PubSubClient** — MQTT 客户端
- **ArduinoJson** — JSON 解析
- **Preferences** — ESP32 Flash 存储（内置）
- **WiFi** — ESP32 WiFi（内置）

开发板选择 `ESP32C3 Dev Module`。

### 3. 上电启动

1. 上电后 RGB LED 快速闪烁彩虹色（红绿蓝各 200ms × 2 轮），系统自检
2. OLED 显示当前所有状态
3. 默认自动连接上次保存的 WiFi（首次使用默认 SSID）
4. WiFi 连接成功后自动连接 MQTT 服务器

## 按键操作

| 按键 | GPIO | 操作 | 功能 |
|------|------|------|------|
| KEY1 | 20 | 按下（低电平） | **总闸关闭**：强制关闭所有继电器，进入自动模式 |
| KEY1 | 20 | 松开（高电平） | **总闸开启**：恢复自动/手动模式运行 |
| KEY2 | 21 | 短按（< 0.8s） | **手动模式**下循环切换 4 个继电器状态 |
| KEY2 | 21 | 长按（≥ 2s） | **切换自动/手动模式** |

### 继电器四态（手动模式）

| 状态 | 灯光 (继电器1) | 风扇 (继电器2) |
|------|:---:|:---:|
| 0 (00) | OFF | OFF |
| 1 (01) | OFF | ON  |
| 2 (10) | ON  | OFF |
| 3 (11) | ON  | ON  |

## RGB LED 状态说明

| 颜色/效果 | 含义 |
|-----------|------|
| 🌈 彩虹快闪（开机 2 轮） | 系统自检 |
| 🔴🔵🟢 慢循环（1s/色） | 正常运行 |
| 🔴 红闪（200ms） | WiFi 未连接 |
| 🔴🟢🔵🔘 交替（500ms） | MQTT 未连接 |
| 🔴🟢🔵 交替（300ms） | 光照越界告警 |
| 🔴🟢🔵 交替（500ms） | 温度越界告警 |
| 🔴🟢🔵 快闪（100ms） | 严重告警（光照+温度同时越界） |

## 串口命令

波特率 **115200**，每条命令以换行结尾。

### WiFi 命令

| 命令 | 功能 |
|------|------|
| `s` | 扫描可用网络并列出 |
| `编号` | 选择扫描列表中的网络 |
| `密码` | 在提示后输入密码（明文） |

> 密码和 SSID 自动保存到 Flash，下次开机自动连接。

### MQTT 命令

| 命令 | 功能 |
|------|------|
| `mqtt show` | 显示当前 MQTT 配置 |
| `mqtt ip <IP>` | 设置 MQTT 服务器 IP |
| `mqtt port <端口>` | 设置 MQTT 端口 |
| `mqtt user <用户名>` | 设置 MQTT 用户名 |
| `mqtt pass <密码>` | 设置 MQTT 密码 |
| `mqtt id <设备ID>` | 设置设备 ID |
| `mqtt save` | 保存所有配置到 Flash |
| `mqtt connect` | 立即连接/重连 MQTT |
| `mqtt status` | 立即上报一次状态 |

## MQTT 协议

详见 [MQTT 通信协议](./protocol.md)。

### Topic 格式

| 方向 | Topic | 说明 |
|------|-------|------|
| 上行（上报） | `chemctrl/{device_id}/status` | 设备 → 服务器，状态上报 |
| 下行（指令） | `chemctrl/{device_id}/command` | 服务器 → 设备，远程控制 |

默认 `device_id` = `PCT_100_23A`

## 项目结构

```
PCT2/
├── PCT2.ino          # 主程序 (setup/loop)
├── version.h         # 设备型号定义
├── key.h / key.cpp   # 按键初始化
├── relay.h / relay.cpp   # 双路继电器控制
├── exti.h / exti.cpp     # 按键扫描 + 自动/手动模式逻辑
├── light.h / light.cpp   # 光照采集 + 自动控制
├── temp.h / temp.cpp     # 温度采集 (DS18B20) + 自动控制
├── display.h / display.cpp   # OLED 显示 (U8g2)
├── rgb_led.h / rgb_led.cpp   # RGB LED 状态指示 (WS2812)
├── wifi_manager.h / wifi_manager.cpp  # WiFi 管理
└── mqtt_manager.h / mqtt_manager.cpp  # MQTT 管理
```

## 默认参数

| 参数 | 默认值 |
|------|--------|
| 风扇开启温度 | 31.0 ℃ |
| 风扇关闭温度 | 30.0 ℃ |
| 光照阈值 | 200 lux |
| 设备 ID | PCT_100_23A |
| MQTT 服务器 | 47.98.170.180:8081 |

## 许可证

本项目为课程设计作品，保留所有权利。
