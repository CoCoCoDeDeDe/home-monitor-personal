# Web 监控看板：HA Discovery 兼容物模型 + 通用节点渲染

Issue: CoCoCoDeDeDe/home-monitor#19
状态: 未开工
Phase: 1

任务清单与验收标准以 Issue 为准：https://github.com/CoCoCoDeDeDe/home-monitor/issues/19

## 决策备忘（2026-08-01 讨论结论）

- 用户核心诉求：**尽量通用化、自定义化，方便未来接入各种 node**
- 设计演进：类型卡片映射 → JSON 档案数据驱动 → 物模型（属性/事件/服务思想）→ 最终选定 **HA MQTT Discovery 词汇做基底**（用户拍板，要求保证之后能接真 HA）
- 候选否决：自研轻量 TSL（无生态）、W3C WoT TD（MQTT 支持弱、工具链重）、阿里/小米 TSL（平台锁定）
- 关键设计：**discovery config 由服务端代发（retain）而非固件发**——HA 不关心来源，固件保持零改动轻量，接真 HA 时 publisher 直接复用
- 前端：无框架单页，档案驱动通用渲染 + 未知类型兜底卡片；不引前端构建链
- 附带收益：事件落盘（现在事件在内存，重启即丢）

## 过程记录

（待开工后补充）
