# Web 监控看板：HA Discovery 兼容物模型 + 通用节点渲染

Issue: CoCoCoDeDeDe/home-monitor#19
状态: 已完成（PR #21/#22/#23 全部合并，Issue #19 已关闭）
Phase: 1

任务清单与验收标准以 Issue 为准：https://github.com/CoCoCoDeDeDe/home-monitor/issues/19

## 决策备忘（2026-08-01 讨论结论）

- 用户核心诉求：**尽量通用化、自定义化，方便未来接入各种 node**
- 设计演进：类型卡片映射 → JSON 档案数据驱动 → 物模型（属性/事件/服务思想）→ 最终选定 **HA MQTT Discovery 词汇做基底**（用户拍板，要求保证之后能接真 HA）
- 候选否决：自研轻量 TSL（无生态）、W3C WoT TD（MQTT 支持弱、工具链重）、阿里/小米 TSL（平台锁定）
- 关键设计：**discovery config 由服务端代发（retain）而非固件发**——HA 不关心来源，固件保持零改动轻量，接真 HA 时 publisher 直接复用
- 前端：无框架单页，档案驱动通用渲染 + 未知类型兜底卡片；不引前端构建链
- 附带收益：事件落盘（现在事件在内存，重启即丢）
- **事件存储选 SQLite**：单文件零服务（Python 内置 sqlite3，挂 volume 即可）；事件量小（每天数百条）+ 单写者 + 结构化数据，PostgreSQL/MongoDB 的优势用不上只有运维成本；标准 SQL 保留向 PostgreSQL 平迁的路径；图片类二进制不进库只存路径

## 过程记录

### PR1（2026-08-01）：后端 SQLite 事件落盘 + API

- `events.py`：EventStore（SQLite 单文件，mqtt 回调线程与 FastAPI 线程并发需加锁），表 events(ts/type/node/kind/payload JSON)
- 回调签名加物模型类型前缀：`on_state/on_status/on_health(ntype, node, ...)`，订阅改 `<type>/<id>/*` 通配（新节点类型自动被消费）
- **踩坑**：app 重启时 broker 重放 retain 消息被当成新事件重复记录、甚至重复告警——回调加 `retained` 标志（paho `msg.retain`），重放只更新注册表不落盘不告警
- compose 挂 `./data:/data`，EVENTS_DB=/data/events.db；server/data/ 入 .gitignore
- 验证：容器重建后历史保留；短接开门事件 id 6/7 落盘 + Server酱告警正常；retain 修复后重建不再新增重复事件
- 顺带完成 #16 现场验收：带回家换 WiFi，板子配网保持 mDNS 主机名不动即恢复连接（家里 WiFi 需设"专用"网络配置）
- 剩余：PR2 物模型档案 + discovery 代发；PR3 前端看板页

### PR2（2026-08-01）：物模型档案 + HA discovery 代发

- `nodetypes/contact.json`：ha 段（binary_sensor door + rssi/uptime 传感器）+ dashboard 段（渲染映射/事件文案）
- `profiles.py` 档案加载；`discovery.py` 按档案生成 HA config；新节点出现时服务端代发（retain 幂等），固件零改动
- 验证：`mosquitto_sub -t 'homeassistant/#'` 收到 3 条 retain config（binary_sensor/rssi/uptime），字段符合 HA MQTT Discovery 格式；`GET /api/profiles` 正常
- 设计点：registry 新条目触发代发（_ensure_node），app 重启经 retain 重建注册表时自然重发；broker 有持久化，config 本身也 retain，双保险
- 剩余：PR3 前端看板页（closes #19）

### PR3（2026-08-01）：前端看板页

- `static/index.html`：无框架单页，5s 轮询 /nodes + /api/events + /api/profiles；档案驱动节点卡片（primary 状态映射/字段格式化/级别配色）+ 事件时间线（文案/补发标记）+ 未知类型兜底卡片
- `GET /` 返回看板页（FileResponse）
- 踩坑：模板字符串收尾反引号写成单引号 → `Unexpected end of input` 页面只剩骨架；用 `node --check` 提取脚本定位。教训：写完 JS 先过一遍语法检查再部署
- 验证：localhost/LAN/Tailscale 三端可访问；卡片（门状态/信号/在线时长）、事件时间线渲染正确（用户截图确认）
