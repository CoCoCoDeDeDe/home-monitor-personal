# 图引擎设计：IO 节点自由配置逻辑（Niagara 理念）

Issue: CoCoCoDeDeDe/home-monitor#27
日期: 2026-08-02
状态: 已批准（5/5 节用户逐节确认）

## 背景与目标

#27 最初方案是"语义类型档案"（contact/presence 预设语义，raw_map 翻译）。
头脑风暴后演进为参考 Niagara Framework 的数据流图理念：

- 底层只保留硬件本质（collision + 布尔原始值），不预设任何脱离传感器本质的语义
- 用户直接配置 0/1（布尔原始值，map 键 `"1"`/`"0"`）各对应什么人话、哪个级别
- 架构兼容未来图联动（逻辑门/延时/设备互控），第一版先把 IO 输入输出打通

## 关键决策（用户逐节确认）

1. **数据流图**（非仅语义配置/规则联动）：节点=积木块，连线=数据流
2. **引擎先行**：服务端图引擎 + JSON 图定义 + 看板只读拓扑；拖拽编辑器未来做
3. **表单即图的投影**：语义表单不存配置，只是节点子图的读/写投影，图是唯一真身
4. **第一版块类型**：IO 输入点、IO 输出点、语义翻译块、显示点、告警点（5 种）。
   联动块（逻辑门/延时）未来加，只是注册新块类型，不动架构
5. **图引擎为唯一底座**：_on_state 硬编码翻译/告警删除，显示与告警全部来自图输出

## 架构总览

```
设备 ──MQTT──▶ IO输入点 ──▶ 语义翻译块 ──┬──▶ 显示点 ──▶ 看板 /nodes
                                        └──▶ 告警点 ──▶ Server酱
看板表单 ──▶ 生成/修改该节点的子图（投影）
（未来）IO输出点 ──MQTT──▶ 设备；逻辑/延时块插在任意连线中间
```

- 引擎 `server/app/graph.py`：事件驱动，MQTT 消息 → IO 输入点 → 沿连线传播，
  块输出变化才继续传播（增量，不整图重算）
- 存储：SQLite 新增 `graphs` 表（与 events.db 同文件）：`graph_id, node_id, json, ts`；
  每节点一张小图，未来跨节点联动图同表（node_id 空 = 全局图）
- `nodetypes/*.json` 从规则本体降级为块的预设模板与 schema 词汇表

## 块模型与图 schema

```json
{
  "blocks": [
    { "id": "in1",   "kind": "io_in",     "params": { "topic": "collision/6750f8/state" } },
    { "id": "sem1",  "kind": "translate", "params": { "map": {
        "1": { "text": "门窗打开", "level": "warn" },
        "0": { "text": "门窗关闭", "level": "info" } } } },
    { "id": "disp1", "kind": "display",   "params": { "alias": "平台窗户" } },
    { "id": "alm1",  "kind": "alert",     "params": { "cooldown": 60, "channel": "sct" } }
  ],
  "wires": [
    { "from": "in1.state",  "to": "sem1.in" },
    { "from": "sem1.out",   "to": "disp1.state" },
    { "from": "sem1.event", "to": "alm1.trigger" }
  ]
}
```

- 块 = id + kind + params；kind 注册进块注册表，未知 kind 报错不静默
- 端口寻址 `块id.端口名`；IO 输入点出 `state`；翻译块入 `in`，出 `out`（当前显示值）
  与 `event`（变化沿）；显示点/告警点是汇
- 值为带类型字典 `{"raw","text","level","ts"}`，一种格式从头流到尾
- 求值：wires 拓扑排序缓存；输入变化 → 受影响块按拓扑序重算，输出没变截断传播；
  建图/改图时做环检测拒绝
- 加载/保存全量校验（kind 已注册、端口存在、无环、params 过块 schema），非法图不入库

## 表单即图的投影

- 打开表单：`GET /api/nodes/{id}/config` → 从图提取投影（别名、两状态文案+级别、
  告警冷却）；块不存在给默认值
- 保存表单：`PUT /api/nodes/{id}/config` → 重建该节点标准子图（固定块 id：
  in1/sem1/disp1/alm1），整体替换，热加载生效
- 默认值策略：无图节点 → 按固件类型档案自动生成默认图（collision 原始文案），
  用户打开表单看到的是真实投影
- contact/presence 降级为模板：只预填翻译块 map，系统不再记得用过哪个模板
- 未来兼容：用户直接改图加的额外块，表单重建时按固定 id 只动标准块，额外块保留

## 与现有代码的整合

改造：
- `main.py _on_state`：只更新节点元信息（在线/rssi/uptime）、写 events 落盘
  （原始事件是历史事实，不归图管）、喂图引擎求值；翻译/告警硬编码删除
- `registry.py` → 被 graphs 表取代（无用户数据，删表不迁移）
- `alerter.py` → 变成告警块内部实现（冷却逻辑搬入），保留 Server酱通道
- `/nodes`：新增 `display` 字段 = 显示点输出（{text,level}），别名来自显示点 params；
  同时附该节点翻译块 map，供看板事件列表把历史原始事件翻译成当前语义文案
  （历史事件文案随当前图走，见「错误处理」）
- `index.html`：渲染直接读 `display`（前端不再懂翻译）；配置表单 = 文案×2 +
  级别×2 + 模板下拉预填
- `nodetypes/`：collision.json 保留（IO 输入点词汇 + HA discovery）；
  contact/presence 改为纯模板文件

不动：mqtt_client（含 +/syncreq 通用化）、events.py、discovery.py、
LWT/健康上报、固件（PR2 才改 contact→collision，合并不刷机）

## 错误处理

- 非法图拒绝入库（400 + 具体错误）；启动时坏图跳过，节点退化为兜底显示，不拖垮服务
- 运行期块异常：捕获+日志+该次传播终止；告警发送失败打日志不重试（与现行为一致）
- 未知原始值：透传 `{text: raw原样, level: "info"}`，永远能显示、不告警
- 固件词表变化：历史事件落原始值照原样显示；实时显示随当前图

## 测试

`server/app/test_graph.py`（stub fastapi/paho/requests，纯逻辑）：
- 引擎：拓扑序求值、增量传播截断、环拒绝、未知 kind 拒绝
- 块：翻译块映射/透传、告警块冷却一次、显示点输出格式
- 投影链路：表单保存 → 图生成 → MQTT 消息进 → /nodes display 正确 + warn 告警
- 回归：无配置节点（默认图）行为与现状一致

## 验收（#27）

看板给节点填别名 + 自定义两句话，保存立即生效；触发传感器，看板/事件/Server酱
文案全是自定义内容；不改固件。

## PR 划分

- PR1（refs #27）：图引擎 + graphs 表 + 5 种块 + 投影 API + 存量切换 + 看板改造 + 测试
- PR2（closes #27）：固件 contact-node → collision-node（合并不刷机，
  等 #9 长稳 8/5 满期后刷机验证）

## 修订（2026-08-02 晚）：全链路布尔化

用户拍板：IoT 设备 ↔ 后端的线上协议与后端存储**只跑布尔值**（`1`=触发/`0`=释放），
文字只在图输出（显示/告警）一层出现。原设计的 triggered/released 词表取消：

- 固件 payload：`{"state":1}` / `{"state":0}`；极性约定 1=传感器被激活
- events 落盘 `state` 为 int；graphs 表 map 键 `"1"`/`"0"`（JSON 键为字符串，
  翻译块按 `str(raw)` 查找，raw 原样透传进 out）
- 表单恒定两行（1/0）；open/closed/triggered/released 成为历史词，
  仅保留在 contact/presence 模板中服务旧事件记录翻译
- 部署时清空 graphs 表（开发期单节点，词表语义已变，重新配置即可）

## 未来方向（不在本期）

- 联动块：逻辑门（AND/OR/NOT）、延时/持续块、比较块
- 设备互控：IO 输出点接通固件 svc 订阅端（架构已备，固件暂无订阅）
- 图编辑器：JSON 编辑 → 拖拽画布；看板只读拓扑展示先行
- 批量配置：表单系统/模板批量套用（解决多窗户重复配置）
- 极性反转：inverted 模板预填即可，无需新机制
