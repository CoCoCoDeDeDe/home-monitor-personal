# 20260802-node-registry

Issue: CoCoCoDeDeDe/home-monitor#27

节点注册表：别名 + 语义类型绑定，固件退化为原始二值上报（raw state + device_class 模式）。

## 进度

- 8/2 开工。~~设计以 issue #27 正文为准~~ → 头脑风暴后演进为图引擎方案，
  设计定稿见 `2026-08-02-graph-engine-design.md`（Niagara 理念：表单即图的投影，
  引擎为唯一底座，第一版 5 种块打通 IO 输入输出）；当晚再修订为**全链路布尔化**
- 分层：固件上报硬件本质类型（collision，原始布尔 1/0）→ 图引擎求值（翻译块
  参数=用户配置的 1/0 含义）→ 显示点/告警点输出
- 默认值策略：无图节点自动生成默认图（collision 原始文案），兼容现状
- 8/2 **收官**：PR #28（图引擎+布尔化，refs）与 PR #29（固件 collision-node，
  closes）均已 merge，issue #27 关闭。固件 8/5 长稳满期后刷机验证
- 遗留：部署时清过 graphs 表；节点 6750f8 已按布尔模型重新配置
- 后续补充：类型系统 + 通道分离决策已写入 design.md 物模型节（bool/enum/number/
  string 小数据走类型系统；直播/点云/录像走 MQTT 指针 + HTTP 内容）
- 8/6 **固件刷机 + 端到端验证完成**：collision-node 刷入 6750f8（COM6），布尔上报/
  mDNS/翻译/告警全链路实测通过；清理了旧 contact 固件留下的 MQTT retain 残留
  （/nodes type 卡在 contact 的原因：type 是进程内"首次见到"记录，重启 app 即刷新）
- 8/6 **bug 修复 #32**：保存配置/重启后卡片回退显示原始值 → display 缓存空洞时
  用当前 state 按翻译表现算兜底（PR #33，closes #32）。排查中一度误判 map 乱码，
  实为 Git Bash python 管道 GBK 解码假象，DB/服务端数据始终正常
- 计划 2 个 PR：
  - PR1（refs #27）：服务端注册表 + 配置 API + 语义翻译 + collision/presence 档案 + syncreq 通用化
  - PR2（closes #27）：固件 contact-node → collision-node 改名 + 状态词改 triggered/released（合并不刷机，等 #9 长稳满期后刷）
- 边界：开发期服务端重启会在 #9 长稳日志混入离线/上线事件，每日检查时标注为已知重启。
