# contact-node 可靠性增强：LWT + 离线缓存补发

Issue: CoCoCoDeDeDe/home-monitor#6
状态: 已完成（PR #7 已 squash 合并，Issue #6 已关闭）
Phase: 1

任务清单与验收标准以 Issue 为准：https://github.com/CoCoCoDeDeDe/home-monitor/issues/6

## 设计要点

- **LWT 在线状态**：固件 `mqtt.connect(nodeId, user, pass, topicStatus, 1, true, "offline")`，上线发 `online`（retain）；异常掉线 broker 自动广播 `offline`。app 订阅 `contact/+/status` 维护节点注册表，`GET /nodes` 可查。
- **离线缓存**：MQTT 断开期间的状态变化写入 LittleFS `/events.log`（上限 400B，约 10 条，超出丢弃并打印告警）。
- **syncreq/sync 握手补发**（测试中发现并修复的竞态）：
  - 初版设计是"重连成功立即补发"，实测发现竞态：board 重连 ≤5s，而 app 的 paho 重连退避最长 60s，补发时 app 可能还没订阅，除 retain 的最后一条外全部丢失（本该告警的 open 事件就丢了）。
  - 改为握手模式：板子重连后只发 `contact/syncreq`；app 每次（重）连后、及收到 syncreq 时，广播 `contact/sync`；板子收到 sync 才补发（payload 带 `"cached":true`）并清档；30s 没等到 sync 重发 syncreq 兜底。
- **QoS 说明**：PubSubClient 发布只支持 QoS 0。QoS 1 由 LWT（willQoS=1）和缓存补发机制兜住。
- **FLASH 复位窗口**：上电瞬间采样 GPIO0 会进 ROM 下载模式，改为开机后 1.5s 窗口内按住 FLASH ≥300ms 清空 WiFi+MQTT 配置。

## 改动文件

- `firmware/contact-node/src/main.cpp`：LWT、LittleFS 事件缓存、syncreq/sync 握手补发、FLASH 复位窗口检测
- `server/app/mqtt_client.py`：订阅 `contact/+/state` + `contact/+/status` + `contact/syncreq`，连接后广播 `contact/sync`，双回调 on_state(node,state,cached)/on_status(node,status)
- `server/app/main.py`：节点注册表 + `GET /nodes`；节点键名统一 `node.removeprefix("contact-")`（payload 带前缀、topic 不带，否则一个节点两条记录）

## 验证记录（2026-08-01，实体板 contact-6750f8 / COM3）

1. **LWT 掉线检测**：拔 USB 断电，50s 内 `/nodes` 变 `offline`；插回后自动恢复 `online` ✅
2. **断线缓存**：`docker stop server-mosquitto-1`，短接 D1-GND 模拟开门→关门，串口确认 `cached event: open/closed` 写入 LittleFS ✅
3. **握手补发端到端**：`docker start` + 重启 app，串口 `sync requested` → 收到 sync → `cached events flushed`；app 日志收到 `open(cached:true)` → **Server酱告警推送成功（微信收到）** → `closed(cached:true)`；`/nodes` 恢复 online/closed ✅
4. **踩坑记录**：
   - 主机自访 LAN IP（192.168.110.110）不通是 WSL mirrored 网络 hairpin 的正常现象，localhost 可达即可，勿当故障排查。
   - `docker stop` 后板子需 ~90s（keepalive 超时）才检测到断线，此期间事件会发到半开连接上丢失——测试时必须等串口出现 `mqtt connect failed` 重试后再模拟事件。
   - `pio device monitor` 重定向到文件需 `PYTHONUNBUFFERED=1`，否则块缓冲看不到输出；监视器附着会复位板子（DTR）。
   - 杜邦线短接接触不良会导致事件检测不到，串口无输出时先查接线。

## 遗留/后续

- 半开连接窗口（keepalive 检测前 ~90s）内的事件仍会丢失，如需更强保障可考虑缩短 keepalive 或发布失败回退缓存（当前实测主要断线场景都能被缓存覆盖）。
- 板子每次 boot 会把初始状态当事件缓存并走一次 sync 握手补发，无害但多一次 syncreq 往返。
