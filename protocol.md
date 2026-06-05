# MQTT 通信协议

## 连接信息

| 参数 | 默认值 |
|------|--------|
| 服务器 | 47.98.170.180 |
| 端口 | 8081 |
| 用户名 | dzdx_emqx |
| 密码 | Jp4!sQ7$ |
| 设备 ID | PCT_100_23A |
| Client ID | {device_id}_dev |
| Keep Alive | 30s |
| 上报间隔 | 事件触发（状态变化时立即上报） |

## Topic 定义

### 上行：状态上报

```
Topic:  chemctrl/{device_id}/status
方向:  设备 → 服务器
触发:  状态变化事件（温度变化 > 0.5℃ / 光照变化 > 20 lux / 按键 / 继电器 / 模式改变）
QoS:   0
```

#### Payload 示例

```json
{
  "temperature": 28.5,
  "light": 185,
  "mode": "auto",
  "key1_lock": true,
  "relay3": false,
  "relay4": true,
  "temp_threshold": 31.0,
  "light_threshold": 200
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `temperature` | float | 当前温度 (℃)，精度 0.1 |
| `light` | int | 当前光照 (lux) |
| `mode` | string | `"auto"` 自动模式 / `"manual"` 手动模式 |
| `key1_lock` | bool | KEY1 总闸状态 (`true`=开启) |
| `relay3` | bool | 灯光继电器 (`true`=ON) |
| `relay4` | bool | 风扇继电器 (`true`=ON) |
| `temp_threshold` | float | 风扇开启温度阈值 (℃) |
| `light_threshold` | int | 灯光开启光照阈值 (lux) |

### 下行：指令控制

```
Topic:  chemctrl/{device_id}/command
方向:  服务器 → 设备
QoS:   0
```

> 设备也订阅了自己的 status topic，用于指令响应确认。无 `cmd` 字段的消息（如自己的 status 回显）会被忽略。

#### 1. 查询状态

```json
{
  "cmd": "get_status"
}
```

设备收到后立即上报一次完整状态。

#### 2. 控制继电器

```json
{
  "cmd": "set_relay",
  "relay": 3,
  "value": true
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `relay` | int | `3` = 灯光，`4` = 风扇 |
| `value` | bool | `true` = 开启，`false` = 关闭 |

> ⚠️ KEY1 关闭（总闸断开）时，`set_relay` 指令被忽略。

#### 3. 切换模式

```json
{
  "cmd": "set_mode",
  "mode": "auto"
}
```

| mode 值 | 说明 |
|---------|------|
| `"auto"` | 切换到自动模式（传感器自动控制继电器） |
| `"manual"` | 切换到手动模式（通过 set_relay 或按键控制） |

#### 4. 设置阈值

```json
{
  "cmd": "set_threshold",
  "temp": 32.0,
  "light": 250
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `temp` | float | 风扇开启温度阈值 (℃)，关闭阈值自动设为 temp - 1℃ |
| `light` | int | 灯光开启光照阈值 (lux) |

> 兼容字段别名：`temperature` / `temp_threshold` 均等同于 `temp`；`lux` / `light_threshold` 均等同于 `light`。
> 阈值修改后自动保存到 Flash，断电不丢失。

#### 5. 远程重启

```json
{
  "cmd": "reboot"
}
```

设备延时 100ms 后调用 `ESP.restart()`。

## 自动控制逻辑

### 自动模式 (auto)

```
光照 ADC > light_on_th  → 灯光继电器 ON  (开灯)
光照 ADC < light_off_th → 灯光继电器 OFF (关灯)

温度 > fan_on_temp  → 风扇继电器 ON  (开风扇)
温度 < fan_off_temp → 风扇继电器 OFF (关风扇)
```

- 光照使用**滞回控制**（开/关阈值不同，避免抖动）
- 温度使用 **1℃ 回差**（fan_on_temp - fan_off_temp = 1℃）
- 灯光和风扇独立控制，互不影响

### 手动模式 (manual)

- 传感器自动控制**停止**
- 继电器状态由 `set_relay` MQTT 指令或 KEY2 短按手动切换
- KEY2 短按循环切换 4 种组合：00 → 01 → 10 → 11 → 00…

## 状态变化触发上报

以下任一条件满足时，设备立即上报状态：

| 触发条件 | 阈值 |
|----------|------|
| 温度变化 | > 0.5 ℃ |
| 光照变化 | > 20 lux |
| KEY1 状态改变 | 任意变化 |
| 灯光继电器状态改变 | 任意变化 |
| 风扇继电器状态改变 | 任意变化 |
| 自动/手动模式切换 | 任意变化 |
| 收到 `get_status` 指令 | 立即触发 |
| 收到 `set_threshold` 指令 | 保存后触发 |
| `set_relay` 执行成功 | 立即触发 |
