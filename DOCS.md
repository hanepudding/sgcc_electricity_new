# SGCC Electricity Add-on

定时抓取国家电网电费余额和用电量数据，推送到 Home Assistant 传感器。

## 配置

安装后在「配置」标签页填写以下必填项：

| 参数 | 说明 |
|------|------|
| PHONE_NUMBER | 国网登录手机号 |
| PASSWORD | 国网登录密码 |
| HASS_TOKEN | HA 长期访问令牌 |
| LLM_API_KEY | 大模型 API Key（需支持图片输入） |
| LLM_BASE_URL | 大模型 API 地址 |
| LLM_MODEL | 模型名称 |

点击「显示未使用的可选配置选项」可配置 IGNORE_USER_ID 等可选参数。

填写完成后点击「保存」，返回「信息」标签页点击「启动」。

## 查看运行状态

启动后点击「日志」标签页查看运行日志。首次启动会立即执行一次数据获取，之后每 12 小时自动执行。

## 完整文档

详细说明见 [README](https://github.com/hanepudding/sgcc_electricity_new/blob/master/README.md)。
