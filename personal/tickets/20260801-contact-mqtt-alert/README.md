# contact-node MQTT 上报 + app 消费 + Server酱告警

Issue: CoCoCoDeDeDe/home-monitor#3
状态: 进行中
Phase: 1

任务清单与验收标准以 Issue 为准：https://github.com/CoCoCoDeDeDe/home-monitor/issues/3

## 过程记录

- 架构决策（Broker/后端/告警/前端/配网）：Mosquitto + FastAPI + Server酱 Turbo + 服务端渲染 + WiFiManager，已录入 design.md（PR #4）
- 固件：WiFiManager 自定义参数同时配置 MQTT host/port/user/pass；节点 ID 用 `contact-<chipid>` 保证唯一；发布带 retain，app 重启可拿最新状态；上线即上报当前状态
- app：paho-mqtt 后台线程（loop_start）不阻塞 FastAPI；断线自动重连 1-60s 递增；Alerter 按节点冷却
- 端到端模拟验证通过：开门→告警、冷却期跳过、关门不告警、/health 显示 mqtt_connected
- **实板端到端验证通过**：ESP8266(NodeMCU, contact-6750f8) 短接 D1-GND → 微信收到"门窗打开告警"
- 踩坑（详见 docs/deployment.md）：
  - Docker Desktop 端口发布只到 localhost，局域网设备需 mirrored 网络 + 防火墙规则 + Hyper-V 入站 Allow 三层放行；portproxy 方案无效；主机自访 LAN IP 不通是 hairpin 正常现象
  - WiFiManager 自定义参数不落盘 → 固件加 LittleFS 持久化；FLASH 键复位改开机后 1.5s 窗口检测（上电瞬间按住 GPIO0 会进下载模式）
  - CH340 驱动需手动装；串口监视进程残留会占用 COM 口导致烧录失败
