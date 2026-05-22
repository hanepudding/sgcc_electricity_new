# ⚡️ 国家电网电力获取

Fork of [ARC-MX/sgcc_electricity_new](https://github.com/ARC-MX/sgcc_electricity_new)（Apache 2.0），主要区别：

- **LLM 端点可配置**：上游硬编码火山引擎豆包，本 fork 支持任意 OpenAI 兼容 API（Gemini、OpenAI 等）
- **CloakBrowser 反检测**：上游已切回普通 Selenium，本 fork 保留 CloakBrowser（Chromium 源码级反检测）
- **CI 使用 GHCR**：镜像托管在 GitHub Container Registry，不依赖阿里云/Docker Hub

[![Docker Image CI](https://github.com/hanepudding/sgcc_electricity_new/actions/workflows/docker-image.yml/badge.svg)](https://github.com/hanepudding/sgcc_electricity_new/actions/workflows/docker-image.yml)

<p align="center">
<img src="assets/image-20230730135540291.png" alt="mini-graph-card" width="400">
<img src="assets/image-20240514.jpg" alt="mini-graph-card" width="400">
</p>

## 简介

将国网电费、用电量数据接入 Home Assistant，实时追踪家庭用电情况，可选每日用电量存储到数据库。

提供以下传感器实体：

| 实体 entity_id                          | 说明                                               |
| -------------------------------------- | -------------------------------------------------- |
| sensor.last_electricity_usage_xxxx     | 最近一天用电量，单位 KWH                           |
| sensor.electricity_charge_balance_xxxx | 预付费显示电费余额，反之显示上月应交电费，单位元   |
| sensor.yearly_electricity_usage_xxxx   | 今年总用电量，单位 KWH                             |
| sensor.yearly_electricity_charge_xxxx  | 今年总用电费，单位元                               |
| sensor.month_electricity_usage_xxxx    | 最近一个月用电量，单位 KWH                         |
| sensor.month_electricity_charge_xxxx   | 上月总用电费，单位元                               |
| sensor.month_valley_usage_xxxx         | 当月谷时用电量，单位 KWH                           |
| sensor.month_flat_usage_xxxx           | 当月平时用电量，单位 KWH                           |
| sensor.month_peak_usage_xxxx           | 当月峰时用电量，单位 KWH                           |
| sensor.month_tip_usage_xxxx            | 当月尖时用电量，单位 KWH                           |
| sensor.prepay_balance_xxxx             | 预付费余额/应交金额（后付费账户），单位元          |

可选：近 7/30 天每日用电量写入 SQLite 数据库（表名 `daily{userid}`）。

## 适用范围

适用于除南方电网覆盖省份（广东、广西、云南、贵州、海南）外的用户。

支持架构：`linux/amd64`。

## 实现流程

通过 Selenium + **CloakBrowser 反检测浏览器**自动登录国家电网官网获取数据。登录时的腾讯点击/滑动验证码通过**大模型（LLM）视觉识别**解算，支持任意 OpenAI 兼容 API（Google Gemini、OpenAI 等）。获取数据后通过 Home Assistant [REST API](https://developers.home-assistant.io/docs/api/rest/) 更新传感器状态。

---

# 安装与部署

## 0）获取大模型 API Key

验证码解算需要一个支持视觉的大模型 API。上游项目推荐使用（并硬编码）**火山引擎豆包**模型。

任何 OpenAI 兼容 API 均可，配置 `LLM_API_KEY`、`LLM_BASE_URL`、`LLM_MODEL` 三个环境变量即可。

**Gemini 示例：**
```bash
LLM_API_KEY="AIzaSy..."
LLM_BASE_URL="https://generativelanguage.googleapis.com/v1beta/openai"
LLM_MODEL="gemini-2.5-flash"
```

**火山引擎豆包示例：**
```bash
LLM_API_KEY="ark-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
LLM_BASE_URL="https://ark.cn-beijing.volces.com/api/v3"
LLM_MODEL="doubao-seed-2-0-pro-260215"
```

## 1）注册国家电网账户

注册并绑定电表：[https://www.95598.cn/osgweb/login](https://www.95598.cn/osgweb/login)

## 2）获取 HA 长期访问令牌

1. 打开 Home Assistant → 左下角点击你的用户名
2. 滚动到页面底部「长期访问令牌」
3. 点击「创建令牌」→ 起个名字 → 复制生成的 token

## 3）部署

### 方式 A：Home Assistant Add-on（推荐）

1. HA 前端 → 设置 → Add-on 商店 → 右上角 ⋮ → 仓库 → 添加：
   ```
   https://github.com/hanepudding/sgcc_electricity_new
   ```
2. 刷新页面，找到「SGCC Electricity」→ 安装
3. 在 Add-on 配置页填写参数（等同于 `.env` 中的配置项）→ 启动

### 方式 B：Docker Compose

```bash
git clone https://github.com/hanepudding/sgcc_electricity_new.git
cd sgcc_electricity_new
cp example.env .env
vim .env  # 填写配置，参考 example.env 中的注释
docker-compose up -d
```

查看日志：`docker-compose logs sgcc_electricity_app`

更新：

```bash
docker-compose down
docker-compose pull
git pull origin master
docker-compose up -d
```

## 4）HA 配置

在 `configuration.yaml` 中添加以下模板传感器（将 `_xxxx` 替换为日志中显示的用户 ID 后缀）：

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

配置完成后重启 HA。

## 5）HA 数据展示

结合 [mini-graph-card](https://github.com/kalkih/mini-graph-card) 和 [mushroom](https://github.com/piitaya/lovelace-mushroom) 实现美化效果：

<img src="assets/Ha-mini-card.jpg" alt="Ha-mini-card.jpg" style="zoom: 50%;" />

<details>
<summary>Lovelace 卡片 YAML</summary>

```yaml
type: vertical-stack
cards:
  - type: custom:mini-graph-card
    entities:
      - entity: sensor.last_electricity_usage_xxxx
        name: 国网每日用电量
        aggregate_func: first
        show_state: true
        show_points: true
        icon: mdi:lightning-bolt-outline
      - entity: sensor.electricity_charge_balance_xxxx
        name: 电费余额
        aggregate_func: first
        show_state: true
        show_points: true
        color: "#e74c3c"
        icon: mdi:cash
        y_axis: secondary
    group_by: date
    hour24: true
    hours_to_show: 240
    lower_bound: 0
    upper_bound: 10
    lower_bound_secondary: 0
    upper_bound_secondary: 120
    show:
      icon: false
  - type: horizontal-stack
    cards:
      - graph: none
        type: sensor
        entity: sensor.month_electricity_charge_xxxx
        detail: 1
        name: 上月电费
        icon: ""
        unit: 元
      - graph: none
        type: sensor
        entity: sensor.month_electricity_usage_xxxx
        detail: 1
        name: 上月用电量
        unit: 度
        icon: mdi:lightning-bolt-outline
  - type: horizontal-stack
    cards:
      - animate: true
        entities:
          - entity: sensor.yearly_electricity_usage_xxxx
            name: 今年总用电量
            aggregate_func: first
            show_state: true
            show_points: true
        group_by: date
        hour24: true
        hours_to_show: 240
        type: custom:mini-graph-card
      - animate: true
        entities:
          - entity: sensor.yearly_electricity_charge_xxxx
            name: 今年总用电费用
            aggregate_func: first
            show_state: true
            show_points: true
        group_by: date
        hour24: true
        hours_to_show: 240
        type: custom:mini-graph-card
```

</details>

## 6）余额不足通知

在 `.env` 中设置 `PUSH_TYPE=PUSHPLUS` 和 `BALANCE` 阈值。使用 [pushplus](https://www.pushplus.plus/) 推送，token 获取参考[教程](https://cloud.tencent.com/developer/article/2139538)。

---

## 致谢

> 基于 [ARC-MX/sgcc_electricity_new](https://github.com/ARC-MX/sgcc_electricity_new)（Apache 2.0）fork。
> 原始项目：[louisslee/sgcc_electricity](https://github.com/louisslee/sgcc_electricity)。
> 感谢 ARC-MX 及所有贡献者的工作。
