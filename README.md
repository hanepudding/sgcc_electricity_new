# ⚡ 国家电网电力获取

[![CI](https://github.com/hanepudding/sgcc_electricity_new/actions/workflows/docker-image.yml/badge.svg)](https://github.com/hanepudding/sgcc_electricity_new/actions/workflows/docker-image.yml)

定时用无头浏览器登录 [95598.cn](https://www.95598.cn/osgweb/login)，抓取电费余额和用电量，推送到 Home Assistant 传感器。验证码由大模型视觉识别自动解算。

```
95598.cn ──[CloakBrowser]──→ 抓取数据 ──[REST API]──→ Home Assistant 传感器
              ↑                           ↑
        LLM 验证码识别              本地缓存防掉数据
```

每 12 小时抓取一次（±10 分钟随机偏移），中间每 5 分钟从缓存重推，HA 重启也不丢数据。

> Fork of [ARC-MX/sgcc_electricity_new](https://github.com/ARC-MX/sgcc_electricity_new)（Apache 2.0）。区别：LLM 端点可配置、保留 CloakBrowser 反检测、GHCR 镜像。如果你（或你的 LLM）想深入了解项目架构，可参照 [README-LLM.md](README-LLM.md)。

---

## TL;DR — Home Assistant Add-on

1. **添加仓库**：HA → 设置 → Add-on → 右上角 ⋮ → 仓库 → 粘贴：
   ```
   https://github.com/hanepudding/sgcc_electricity_new
   ```
2. **安装**：刷新 → 找到「SGCC Electricity」→ 安装
3. **配置并启动**：填写 PHONE_NUMBER、PASSWORD、HASS_TOKEN、LLM 三项 → 启动

查看日志确认运行正常即可。传感器会自动出现在 HA 中。

---

## 你需要准备

| 项目 | 说明 |
|------|------|
| **国网账号** | [95598.cn](https://www.95598.cn/osgweb/login) 注册并绑定电表 |
| **LLM API Key** | 任意支持**图片输入**的 OpenAI 兼容 API，详见下方 |
| **HA 长期访问令牌** | HA → 左下角用户名 → 页面底部 → 创建令牌 |

### LLM 配置示例

验证码识别需要大模型**支持图片输入（Vision）**。纯文本模型（如 DeepSeek V4 Flash）不可用。

**Gemini（推荐）：**
```
LLM_API_KEY=AIzaSy...
LLM_BASE_URL=https://generativelanguage.googleapis.com/v1beta/openai
LLM_MODEL=gemini-2.5-flash
```

**火山引擎豆包：**
```
LLM_API_KEY=ark-xxxxxxxx
LLM_BASE_URL=https://ark.cn-beijing.volces.com/api/v3
LLM_MODEL=doubao-seed-2-0-pro-260215
```

---

## 传感器列表

`XXXX` = 户号后四位，自动创建。

| 实体 | 说明 |
|------|------|
| `sensor.last_electricity_usage_XXXX` | 最近一天用电量 (kWh) |
| `sensor.electricity_charge_balance_XXXX` | 电费余额 / 上月应交电费 (CNY) |
| `sensor.yearly_electricity_usage_XXXX` | 今年总用电量 (kWh) |
| `sensor.yearly_electricity_charge_XXXX` | 今年总电费 (CNY) |
| `sensor.month_electricity_usage_XXXX` | 上月用电量 (kWh) |
| `sensor.month_electricity_charge_XXXX` | 上月电费 (CNY) |
| `sensor.month_valley_usage_XXXX` | 当月谷时 (kWh) |
| `sensor.month_flat_usage_XXXX` | 当月平时 (kWh) |
| `sensor.month_peak_usage_XXXX` | 当月峰时 (kWh) |
| `sensor.month_tip_usage_XXXX` | 当月尖时 (kWh) |
| `sensor.prepay_balance_XXXX` | 预付费余额 / 应交金额 (CNY) |

可选：历史数据存入 SQLite / MySQL（设置 `DB_TYPE`）。

---

## Docker Compose 部署（Alternative）

```bash
git clone https://github.com/hanepudding/sgcc_electricity_new.git
cd sgcc_electricity_new
cp example.env .env   # 编辑填写配置
docker compose up -d
docker compose logs -f sgcc_electricity_app
```

更新：`docker compose pull && docker compose up -d`

---

## HA 可选配置

### 模板传感器

REST API 创建的传感器已包含 `device_class`、`state_class` 等属性，一般无需额外配置。如需在能源面板中使用，可添加 trigger-based 模板传感器：

<details>
<summary>configuration.yaml 模板（点击展开）</summary>

将 `xxxx` 替换为户号后四位。

```yaml
template:
  - trigger:
      - platform: event
        event_type: state_changed
        event_data:
          entity_id: sensor.electricity_charge_balance_xxxx
    sensor:
      - name: electricity_charge_balance_xxxx
        unique_id: electricity_charge_balance_xxxx
        state: "{{ states('sensor.electricity_charge_balance_xxxx') }}"
        state_class: measurement
        unit_of_measurement: "CNY"
        device_class: monetary

  - trigger:
      - platform: event
        event_type: state_changed
        event_data:
          entity_id: sensor.last_electricity_usage_xxxx
    sensor:
      - name: last_electricity_usage_xxxx
        unique_id: last_electricity_usage_xxxx
        state: "{{ states('sensor.last_electricity_usage_xxxx') }}"
        state_class: measurement
        unit_of_measurement: "kWh"
        device_class: energy

  - trigger:
      - platform: event
        event_type: state_changed
        event_data:
          entity_id: sensor.yearly_electricity_usage_xxxx
    sensor:
      - name: yearly_electricity_usage_xxxx
        unique_id: yearly_electricity_usage_xxxx
        state: "{{ states('sensor.yearly_electricity_usage_xxxx') }}"
        state_class: total_increasing
        unit_of_measurement: "kWh"
        device_class: energy

  - trigger:
      - platform: event
        event_type: state_changed
        event_data:
          entity_id: sensor.yearly_electricity_charge_xxxx
    sensor:
      - name: yearly_electricity_charge_xxxx
        unique_id: yearly_electricity_charge_xxxx
        state: "{{ states('sensor.yearly_electricity_charge_xxxx') }}"
        state_class: total_increasing
        unit_of_measurement: "CNY"
        device_class: monetary

  - trigger:
      - platform: event
        event_type: state_changed
        event_data:
          entity_id: sensor.month_electricity_usage_xxxx
    sensor:
      - name: month_electricity_usage_xxxx
        unique_id: month_electricity_usage_xxxx
        state: "{{ states('sensor.month_electricity_usage_xxxx') }}"
        state_class: measurement
        unit_of_measurement: "kWh"
        device_class: energy

  - trigger:
      - platform: event
        event_type: state_changed
        event_data:
          entity_id: sensor.month_electricity_charge_xxxx
    sensor:
      - name: month_electricity_charge_xxxx
        unique_id: month_electricity_charge_xxxx
        state: "{{ states('sensor.month_electricity_charge_xxxx') }}"
        state_class: measurement
        unit_of_measurement: "CNY"
        device_class: monetary

  - trigger:
      - platform: event
        event_type: state_changed
        event_data:
          entity_id: sensor.month_valley_usage_xxxx
    sensor:
      - name: month_valley_usage_xxxx
        unique_id: month_valley_usage_xxxx
        state: "{{ states('sensor.month_valley_usage_xxxx') }}"
        state_class: measurement
        unit_of_measurement: "kWh"
        device_class: energy

  - trigger:
      - platform: event
        event_type: state_changed
        event_data:
          entity_id: sensor.month_flat_usage_xxxx
    sensor:
      - name: month_flat_usage_xxxx
        unique_id: month_flat_usage_xxxx
        state: "{{ states('sensor.month_flat_usage_xxxx') }}"
        state_class: measurement
        unit_of_measurement: "kWh"
        device_class: energy

  - trigger:
      - platform: event
        event_type: state_changed
        event_data:
          entity_id: sensor.month_peak_usage_xxxx
    sensor:
      - name: month_peak_usage_xxxx
        unique_id: month_peak_usage_xxxx
        state: "{{ states('sensor.month_peak_usage_xxxx') }}"
        state_class: measurement
        unit_of_measurement: "kWh"
        device_class: energy

  - trigger:
      - platform: event
        event_type: state_changed
        event_data:
          entity_id: sensor.month_tip_usage_xxxx
    sensor:
      - name: month_tip_usage_xxxx
        unique_id: month_tip_usage_xxxx
        state: "{{ states('sensor.month_tip_usage_xxxx') }}"
        state_class: measurement
        unit_of_measurement: "kWh"
        device_class: energy

  - trigger:
      - platform: event
        event_type: state_changed
        event_data:
          entity_id: sensor.prepay_balance_xxxx
    sensor:
      - name: prepay_balance_xxxx
        unique_id: prepay_balance_xxxx
        state: "{{ states('sensor.prepay_balance_xxxx') }}"
        state_class: measurement
        unit_of_measurement: "CNY"
        device_class: monetary
```

</details>

### 数据看板

<p align="center">
<img src="assets/image-20230730135540291.png" alt="效果图" width="400">
<img src="assets/image-20240514.jpg" alt="效果图" width="400">
</p>

<details>
<summary>Lovelace 卡片 YAML（需安装 mini-graph-card）</summary>

```yaml
type: vertical-stack
cards:
  - type: custom:mini-graph-card
    entities:
      - entity: sensor.last_electricity_usage_xxxx
        name: 每日用电量
        show_state: true
        show_points: true
      - entity: sensor.electricity_charge_balance_xxxx
        name: 电费余额
        show_state: true
        color: "#e74c3c"
        y_axis: secondary
    group_by: date
    hours_to_show: 240
  - type: horizontal-stack
    cards:
      - type: sensor
        entity: sensor.month_electricity_charge_xxxx
        name: 上月电费
        unit: 元
      - type: sensor
        entity: sensor.month_electricity_usage_xxxx
        name: 上月用电量
        unit: 度
```

</details>

---

## 余额不足通知

设置 `PUSH_TYPE=PUSHPLUS` + `BALANCE=5.0`（阈值），使用 [pushplus](https://www.pushplus.plus/) 推送。也支持 `URLPUSH` 自定义 webhook。

## 适用范围

除南方电网覆盖省份（广东、广西、云南、贵州、海南）外的用户。支持架构：`linux/amd64`, `linux/arm64`。

---

## 致谢

基于 [ARC-MX/sgcc_electricity_new](https://github.com/ARC-MX/sgcc_electricity_new)（Apache 2.0）。原始项目：[louisslee/sgcc_electricity](https://github.com/louisslee/sgcc_electricity)。
