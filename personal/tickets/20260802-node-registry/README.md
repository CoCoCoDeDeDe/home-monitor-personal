# 20260802-node-registry

Issue: CoCoCoDeDeDe/home-monitor#27

节点注册表：别名 + 语义类型绑定，固件退化为原始二值上报（raw state + device_class 模式）。

## 进度

- 8/2 开工。~~设计以 issue #27 正文为准~~ → 头脑风暴后演进为图引擎方案，
  设计定稿见 `2026-08-02-graph-engine-design.md`（Niagara 理念：表单即图的投影，
  引擎为唯一底座，第一版 5 种块打通 IO 输入输出）
- 分层：固件上报硬件本质类型（collision，triggered/released 原始词汇）→ 图引擎
  求值（翻译块参数=用户配置的 0/1 含义）→ 显示点/告警点输出
- 默认值策略：无图节点自动生成默认图（collision 原始文案），兼容现状
- 计划 2 个 PR：
  - PR1（refs #27）：服务端注册表 + 配置 API + 语义翻译 + collision/presence 档案 + syncreq 通用化
  - PR2（closes #27）：固件 contact-node → collision-node 改名 + 状态词改 triggered/released（合并不刷机，等 #9 长稳满期后刷）
- 边界：开发期服务端重启会在 #9 长稳日志混入离线/上线事件，每日检查时标注为已知重启。
