# 图引擎（#27）实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 用数据流图引擎替换 _on_state 硬编码翻译/告警，IO 输入输出打通，表单即图的投影。

**Architecture:** 服务端新增 graph_engine/blocks/graph_store/graph_service 四个模块；图存 SQLite graphs 表（与 events.db 同文件）；MQTT 状态事件喂图增量求值；/nodes 的显示文案与告警全部来自图输出；看板配置表单 = 节点标准子图（固定块 id in1/sem1/disp1/alm1）的读/写投影。设计文档：`personal/tickets/20260802-node-registry/2026-08-02-graph-engine-design.md`（同目录）。

**Tech Stack:** Python 3.12 / FastAPI / paho-mqtt / SQLite（标准库）/ 原生 JS；测试为零依赖 `python test_graph.py`（requests 打 stub，不用 pytest）。

> **修订（2026-08-02 晚）：全链路布尔化。** Task 1-7 执行时线上词表还是单词（open/closed、triggered/released），随后用户拍板改为布尔（`1`=触发/`0`=释放），已由后续 commit 修正（翻译块按 `str(raw)` 查 map、档案/模板键改 1/0、表单动态行、模板保留历史词键）。下文 Task 1-7 代码块中的 triggered/released 以实际仓库为准；**Task 8 已按布尔协议改写**。

## Global Constraints

- 仓库 `home-monitor/`，工作分支 `feat/node-registry`；commit message 带 `(refs #27)`；home-monitor 仓库 commit 前列改动清单等用户确认
- 固件本计划只改代码不刷机（窗户节点 6750f8 跑 #9 长稳到 8/5）
- 值格式统一：`{"raw": str, "text": str, "level": "ok"|"info"|"warn", "ts": float}`（io_in 入口只带 raw/ts）
- 块接口：类属性 `kind/inputs/outputs` + 静态方法 `evaluate(params, ins, state, now, ctx) -> dict|None`；`state` 为引擎持有的块私有可变 dict；返回 None = 输出无变化截断传播
- 未知原始值透传 `{text: raw原样, level: "info"}`，不告警
- 非法图（未知 kind/端口不存在/成环/缺字段）抛 `GraphError`，API 层转 400
- 本地无 fastapi/paho/requests：测试文件顶部 stub 第三方包；运行 `cd home-monitor/server/app && python test_graph.py`
- 当前工作树已有 PR1 旧方案未提交改动（registry.py、main/alerter/mqtt_client/index.html/nodetypes 改动、design.md 补充）。mqtt_client 的 `+/syncreq` 通用化与 design.md 补充**保留**；registry.py、main/alerter/index.html/nodetypes 的旧方案改动**被本计划取代**（Task 5/6/7 覆盖或还原）

---

### Task 1: graph_store.py — graphs 表持久化

**Files:**
- Create: `home-monitor/server/app/graph_store.py`
- Create: `home-monitor/server/app/test_graph.py`（本任务先建骨架，后续任务持续追加用例）

**Interfaces:**
- Produces: `GraphStore(path)`；`.save(node_id: str, spec: dict) -> None`；`.load(node_id: str) -> dict|None`；`.load_all() -> dict[str, dict]`（node_id→spec）；`.delete(node_id: str) -> None`；`.close() -> None`

- [ ] **Step 1: 写失败测试**

`test_graph.py` 骨架：

```python
"""图引擎测试：零依赖，python test_graph.py 直接跑。requests stub 在 blocks import 前。"""

import sys, types, tempfile, os

for _n in ["requests"]:
    sys.modules[_n] = types.ModuleType(_n)

FAILED = []
def check(name, fn):
    try:
        fn()
        print(f"PASS {name}")
    except AssertionError as e:
        FAILED.append(name)
        print(f"FAIL {name}: {e}")

def test_store_roundtrip():
    from graph_store import GraphStore
    db = tempfile.mktemp(suffix=".db")
    s = GraphStore(db)
    spec = {"blocks": [{"id": "in1", "kind": "io_in", "params": {}}], "wires": []}
    s.save("6750f8", spec)
    assert s.load("6750f8") == spec, "save 后 load 应原样返回"
    assert s.load_all() == {"6750f8": spec}
    s.save("6750f8", {"blocks": [], "wires": []})  # upsert 覆盖
    assert s.load("6750f8") == {"blocks": [], "wires": []}
    s.delete("6750f8")
    assert s.load("6750f8") is None
    s.close(); os.unlink(db)

check("store_roundtrip", test_store_roundtrip)

if __name__ == "__main__":
    sys.exit(1 if FAILED else 0)
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd home-monitor/server/app && python test_graph.py`
Expected: FAIL（ModuleNotFoundError: graph_store）

- [ ] **Step 3: 实现 graph_store.py**

```python
"""图持久化：graphs 表（与 events.db 同文件）。每节点一张小图；node_id 空串 = 全局图（未来联动）。"""

import json
import sqlite3
import threading
import time

DDL = """
CREATE TABLE IF NOT EXISTS graphs (
  graph_id TEXT PRIMARY KEY,  -- 节点图 = node_id；全局图 = "global:<name>"（未来）
  node_id  TEXT NOT NULL DEFAULT '',
  json     TEXT NOT NULL,
  ts       REAL NOT NULL
);
"""


class GraphStore:
    """线程安全：API 在 FastAPI 线程，求值在 paho 线程。"""

    def __init__(self, path: str):
        self._db = sqlite3.connect(path, check_same_thread=False)
        self._db.executescript(DDL)
        self._lock = threading.Lock()

    def save(self, node_id: str, spec: dict) -> None:
        with self._lock:
            self._db.execute(
                "INSERT INTO graphs (graph_id, node_id, json, ts) VALUES (?,?,?,?) "
                "ON CONFLICT(graph_id) DO UPDATE SET json=excluded.json, ts=excluded.ts",
                (node_id, node_id, json.dumps(spec, ensure_ascii=False), time.time()))
            self._db.commit()

    def load(self, node_id: str) -> dict | None:
        with self._lock:
            row = self._db.execute(
                "SELECT json FROM graphs WHERE graph_id = ?", (node_id,)).fetchone()
        return json.loads(row[0]) if row else None

    def load_all(self) -> dict[str, dict]:
        with self._lock:
            rows = self._db.execute("SELECT graph_id, json FROM graphs").fetchall()
        return {r[0]: json.loads(r[1]) for r in rows}

    def delete(self, node_id: str) -> None:
        with self._lock:
            self._db.execute("DELETE FROM graphs WHERE graph_id = ?", (node_id,))
            self._db.commit()

    def close(self) -> None:
        self._db.close()
```

- [ ] **Step 4: 跑测试确认通过**

Run: `python test_graph.py`
Expected: `PASS store_roundtrip`，退出码 0

- [ ] **Step 5: Commit（确认后）**

```bash
git add server/app/graph_store.py server/app/test_graph.py
git commit -m "feat: graphs 表持久化 GraphStore (refs #27)"
```

---

### Task 2: blocks.py — 5 种块 + 注册表；alerter.py 瘦身为 send_sct

**Files:**
- Create: `home-monitor/server/app/blocks.py`
- Modify: `home-monitor/server/app/alerter.py`（整体替换为 send_sct 一个函数）
- Test: `home-monitor/server/app/test_graph.py`

**Interfaces:**
- Consumes: 无（send_sct 被 blocks 与本模块外共用）
- Produces: `BLOCKS: dict[str, type]`；块类类属性 `kind: str`、`inputs: tuple`、`outputs: tuple`、静态 `evaluate(params: dict, ins: dict, state: dict, now: float, ctx: dict) -> dict|None`；`alerter.send_sct(sendkey: str, title: str, desp: str) -> None`（无 key 打日志）；ctx 约定键：`sendkey`/`node`/`alias`/`publish`

- [ ] **Step 1: 追加失败测试**

```python
def test_translate_block():
    from blocks import BLOCKS
    b = BLOCKS["translate"]
    params = {"map": {"triggered": {"text": "门窗打开", "level": "warn"},
                      "released": {"text": "门窗关闭", "level": "info"}}}
    st = {}
    r1 = b.evaluate(params, {"in": {"raw": "triggered", "ts": 1.0}}, st, 1.0, {})
    assert r1["out"]["text"] == "门窗打开" and r1["out"]["level"] == "warn"
    assert "event" not in r1, "首个值不应产生 event（防开机/retain 误告警）"
    r2 = b.evaluate(params, {"in": {"raw": "released", "ts": 2.0}}, st, 2.0, {})
    assert r2["event"]["text"] == "门窗关闭", "raw 变化应产生 event"
    r3 = b.evaluate(params, {"in": {"raw": "weird", "ts": 3.0}}, st, 3.0, {})
    assert r3["out"]["text"] == "weird" and r3["out"]["level"] == "info", "未知原始值透传"

def test_alert_block_cooldown():
    from blocks import BLOCKS
    sent = []
    import alerter
    orig = alerter.send_sct
    alerter.send_sct = lambda k, t, d: sent.append(t)
    try:
        import blocks
        blocks.send_sct = alerter.send_sct  # monkeypatch 到 blocks 命名空间
        b = BLOCKS["alert"]
        params = {"cooldown": 60, "channel": "sct"}
        st = {}
        ctx = {"sendkey": "", "node": "6750f8", "alias": "平台窗户"}
        warn = {"raw": "triggered", "text": "门窗打开", "level": "warn", "ts": 0}
        b.evaluate(params, {"trigger": warn}, st, 100.0, ctx)
        b.evaluate(params, {"trigger": warn}, st, 120.0, ctx)  # 冷却内
        b.evaluate(params, {"trigger": warn}, st, 200.0, ctx)  # 冷却外
        assert sent == ["门窗打开告警", "门窗打开告警"], f"冷却期只发一次: {sent}"
        info = {"raw": "released", "text": "门窗关闭", "level": "info", "ts": 0}
        b.evaluate(params, {"trigger": info}, st, 300.0, ctx)
        assert len(sent) == 2, "非 warn 不告警"
    finally:
        alerter.send_sct = orig

def test_display_block():
    from blocks import BLOCKS
    b = BLOCKS["display"]
    st = {}
    v = {"raw": "triggered", "text": "门窗打开", "level": "warn", "ts": 1.0}
    r1 = b.evaluate({"alias": "平台窗户"}, {"state": v}, st, 1.0, {})
    assert r1["view"] == {"text": "门窗打开", "level": "warn", "alias": "平台窗户"}
    r2 = b.evaluate({"alias": "平台窗户"}, {"state": v}, st, 2.0, {})
    assert r2 is None, "view 无变化返回 None"

check("translate_block", test_translate_block)
check("alert_block_cooldown", test_alert_block_cooldown)
check("display_block", test_display_block)
```

注意：三个 `check(...)` 放在文件末尾 `if __name__` 之前。

- [ ] **Step 2: 跑测试确认失败**

Run: `python test_graph.py`
Expected: 新用例 FAIL（ModuleNotFoundError: blocks）

- [ ] **Step 3: 实现 alerter.py（整体替换）**

```python
"""Server酱 Turbo 发送（告警块的发送通道；冷却逻辑已搬入 blocks.AlertBlock）。"""

import requests


def send_sct(sendkey: str, title: str, desp: str) -> None:
    """无 key（开发/测试）打日志代替推送；发送失败打日志不重试（告警不阻塞数据流）。"""
    if not sendkey:
        print(f"[alert] (no sendkey) {title}: {desp}", flush=True)
        return
    try:
        resp = requests.post(
            f"https://sctapi.ftqq.com/{sendkey}.send",
            data={"title": title, "desp": desp},
            timeout=10,
        )
        resp.raise_for_status()
        print(f"[alert] 已推送 Server酱: {title}", flush=True)
    except Exception as e:
        print(f"[alert] 推送失败 {title}: {e}", flush=True)
```

- [ ] **Step 4: 实现 blocks.py**

```python
"""图块实现（Issue #27 图引擎）：5 种块 + BLOCKS 注册表。

块接口：类属性 kind/inputs/outputs + 静态 evaluate(params, ins, state, now, ctx)。
state 是引擎持有的块私有 dict（跨求值存活，如冷却时间、上一 raw）。
ctx 由 GraphService 注入：sendkey/node/alias/publish。
值统一 {"raw","text","level","ts"}；io_in 入口只带 raw/ts（engine.feed 构造）。
"""

import json
import time

from alerter import send_sct


class IoInBlock:
    """IO 输入点：MQTT 状态由此进图。不参与 evaluate——引擎 feed 时直接设置其输出。"""

    kind = "io_in"
    inputs = ()
    outputs = ("state",)

    @staticmethod
    def evaluate(params, ins, state, now, ctx):
        return None


class IoOutBlock:
    """IO 输出点：汇，把收到的值 publish 到 params.topic（设备互控地基；固件订阅端未来接）。"""

    kind = "io_out"
    inputs = ("cmd",)
    outputs = ()

    @staticmethod
    def evaluate(params, ins, state, now, ctx):
        v = ins.get("cmd")
        if v is None:
            return None
        publish = ctx.get("publish")
        if publish:
            publish(params["topic"], json.dumps(v, ensure_ascii=False))
        return None


class TranslateBlock:
    """语义翻译块：原始值 → 用户配置文案+级别。out=当前显示值，event=raw 变化沿。"""

    kind = "translate"
    inputs = ("in",)
    outputs = ("out", "event")

    @staticmethod
    def evaluate(params, ins, state, now, ctx):
        v = ins.get("in")
        if v is None:
            return None
        raw = v.get("raw", "")
        m = (params.get("map") or {}).get(raw)
        if m is None:  # 未知原始值透传：永远能显示、不告警
            m = {"text": raw, "level": "info"}
        out = {"raw": raw, "text": m.get("text", raw),
               "level": m.get("level", "info"), "ts": v.get("ts", now)}
        prev = state.get("raw")
        state["raw"] = raw
        result = {"out": out}
        if prev is not None and prev != raw:
            result["event"] = out
        return result


class DisplayBlock:
    """显示点：看板卡片数据源。view 变化才输出（驱动增量传播截断）。"""

    kind = "display"
    inputs = ("state",)
    outputs = ("view",)

    @staticmethod
    def evaluate(params, ins, state, now, ctx):
        v = ins.get("state")
        if v is None:
            return None
        view = {"text": v.get("text"), "level": v.get("level"),
                "alias": params.get("alias", "")}
        if state.get("view") == view:
            return None
        state["view"] = view
        return {"view": view}


class AlertBlock:
    """告警点：warn 级 event → 冷却防抖 → Server酱。首值无 event（翻译块保证）。"""

    kind = "alert"
    inputs = ("trigger",)
    outputs = ()

    @staticmethod
    def evaluate(params, ins, state, now, ctx):
        v = ins.get("trigger")
        if v is None or v.get("level") != "warn":
            return None
        if now - state.get("last", 0) < int(params.get("cooldown", 60)):
            print(f"[alert] 冷却期内跳过: {ctx.get('node')}", flush=True)
            return None
        state["last"] = now
        name = ctx.get("alias") or ctx.get("node", "")
        send_sct(ctx.get("sendkey", ""), f"{v['text']}告警",
                 f"{name}（{ctx.get('node','')}）{v['text']}，"
                 f"时间 {time.strftime('%H:%M:%S')}")
        return None


BLOCKS = {b.kind: b for b in (IoInBlock, IoOutBlock, TranslateBlock, DisplayBlock, AlertBlock)}
```

- [ ] **Step 5: 跑测试确认通过**

Run: `python test_graph.py`
Expected: 4 个 PASS，退出码 0

- [ ] **Step 6: Commit（确认后）**

```bash
git add server/app/blocks.py server/app/alerter.py server/app/test_graph.py
git commit -m "feat: 图块 5 种 + BLOCKS 注册表；alerter 瘦身为 send_sct (refs #27)"
```

---

### Task 3: graph_engine.py — 校验 / 拓扑 / 增量求值

**Files:**
- Create: `home-monitor/server/app/graph_engine.py`
- Test: `home-monitor/server/app/test_graph.py`

**Interfaces:**
- Consumes: `blocks.BLOCKS`（Task 2）
- Produces: `GraphError(ValueError)`；`Graph(spec: dict)`；`.feed(block_id: str, outputs: dict, now: float, ctx: dict) -> set[str]`（本轮产出新输出的块 id）；`.output(block_id: str, port: str)`（当前缓存输出，无则 None）；`.block_params(block_id: str) -> dict|None`

- [ ] **Step 1: 追加失败测试**

```python
def _spec(cycle=False, bad_kind=False, bad_port=False):
    kind2 = "nope" if bad_kind else "translate"
    to_port = "nope" if bad_port else "in"
    wires = [{"from": "in1.state", "to": f"sem1.{to_port}"}]
    if cycle:
        wires.append({"from": "sem1.out", "to": "in1.state"})
    return {"blocks": [
        {"id": "in1", "kind": "io_in", "params": {"topic": "collision/6750f8/state"}},
        {"id": "sem1", "kind": kind2, "params": {"map": {"triggered": {"text": "开", "level": "warn"}}}},
    ], "wires": wires}

def test_graph_validation():
    from graph_engine import Graph, GraphError
    Graph(_spec())  # 合法图不抛
    for bad, why in [(_spec(bad_kind=True), "未知 kind"),
                     (_spec(bad_port=True), "端口不存在"),
                     (_spec(cycle=True), "环")]:
        try:
            Graph(bad)
            raise AssertionError(f"{why} 应抛 GraphError 但没抛")
        except GraphError:
            pass

def test_graph_feed_incremental():
    from graph_engine import Graph
    spec = {"blocks": [
        {"id": "in1", "kind": "io_in", "params": {"topic": "t"}},
        {"id": "sem1", "kind": "translate", "params": {"map": {"triggered": {"text": "开", "level": "warn"},
                                                              "released": {"text": "关", "level": "info"}}}},
        {"id": "disp1", "kind": "display", "params": {"alias": "平台窗户"}},
    ], "wires": [
        {"from": "in1.state", "to": "sem1.in"},
        {"from": "sem1.out", "to": "disp1.state"},
    ]}
    g = Graph(spec)
    produced = g.feed("in1", {"state": {"raw": "triggered", "ts": 1.0}}, 1.0, {})
    assert produced == {"in1", "sem1", "disp1"}, f"首次全链路传播: {produced}"
    assert g.output("disp1", "view") == {"text": "开", "level": "warn", "alias": "平台窗户"}
    produced = g.feed("in1", {"state": {"raw": "triggered", "ts": 2.0}}, 2.0, {})
    assert produced == {"in1", "sem1"}, f"display 输入没变应截断: {produced}"
    produced = g.feed("in1", {"state": {"raw": "released", "ts": 3.0}}, 3.0, {})
    assert "disp1" in produced and g.output("disp1", "view")["text"] == "关"
    assert g.block_params("disp1") == {"alias": "平台窗户"}
    assert g.block_params("nope") is None

check("graph_validation", test_graph_validation)
check("graph_feed_incremental", test_graph_feed_incremental)
```

- [ ] **Step 2: 跑测试确认失败**

Run: `python test_graph.py`
Expected: FAIL（ModuleNotFoundError: graph_engine）

- [ ] **Step 3: 实现 graph_engine.py**

```python
"""图引擎（Issue #27）：图校验 + 拓扑排序 + 事件驱动增量求值。

Graph 只认 spec（{"blocks":[...], "wires":[...]}），不关心持久化与 MQTT。
feed(block_id, outputs) 从某块注入输出并沿连线传播：只有上游产出新输出的块
才重算；块 evaluate 返回 None = 输出无变化 = 截断传播。
"""

from blocks import BLOCKS


class GraphError(ValueError):
    """非法图：未知 kind / 端口不存在 / 引用缺失 / 成环 / 缺字段。"""


def _parse_port(ref: str, what: str) -> tuple[str, str]:
    if "." not in ref:
        raise GraphError(f"{what} 端口引用缺 '.': {ref!r}")
    return tuple(ref.split(".", 1))


class Graph:
    def __init__(self, spec: dict):
        self._blocks = {b["id"]: b for b in spec.get("blocks", [])}
        if len(self._blocks) != len(spec.get("blocks", [])):
            raise GraphError("块 id 重复")
        for bid, b in self._blocks.items():
            kind = b.get("kind")
            if kind not in BLOCKS:
                raise GraphError(f"块 {bid}: 未知 kind {kind!r}")
            if "params" not in b:
                b["params"] = {}
        self._in_wires: dict[str, list[tuple[str, str, str]]] = {
            bid: [] for bid in self._blocks}
        for w in spec.get("wires", []):
            fb, fp = _parse_port(w.get("from", ""), "from")
            tb, tp = _parse_port(w.get("to", ""), "to")
            if fb not in self._blocks or tb not in self._blocks:
                raise GraphError(f"连线引用未知块: {w}")
            if fp not in BLOCKS[self._blocks[fb]["kind"]].outputs:
                raise GraphError(f"块 {fb} 无输出端口 {fp!r}")
            if tp not in BLOCKS[self._blocks[tb]["kind"]].inputs:
                raise GraphError(f"块 {tb} 无输入端口 {tp!r}")
            self._in_wires[tb].append((fb, fp, tp))
        self._topo = self._topo_sort()
        self._outputs: dict[str, dict] = {bid: {} for bid in self._blocks}
        self._state: dict[str, dict] = {bid: {} for bid in self._blocks}

    def _topo_sort(self) -> list[str]:
        """Kahn；有环抛 GraphError。"""
        indeg = {bid: len(ws) for bid, ws in self._in_wires.items()}
        queue = [bid for bid, d in indeg.items() if d == 0]
        order = []
        while queue:
            bid = queue.pop()
            order.append(bid)
            for tid, ws in self._in_wires.items():
                if any(fb == bid for fb, _, _ in ws):
                    indeg[tid] -= 1
                    if indeg[tid] == 0:
                        queue.append(tid)
        if len(order) != len(self._blocks):
            raise GraphError("连线成环")
        return order

    def feed(self, block_id: str, outputs: dict, now: float, ctx: dict) -> set[str]:
        """从 block_id 注入输出并传播。返回本轮产出新输出的块 id 集。"""
        if block_id not in self._blocks:
            raise GraphError(f"feed 未知块 {block_id!r}")
        self._outputs[block_id].update(outputs)
        produced = {block_id}
        for bid in self._topo:
            if bid == block_id or bid in produced:
                continue
            ws = self._in_wires[bid]
            if not any(fb in produced for fb, _, _ in ws):
                continue
            block = BLOCKS[self._blocks[bid]["kind"]]
            ins = {tp: self._outputs[fb].get(fp) for fb, fp, tp in ws}
            try:
                result = block.evaluate(self._blocks[bid]["params"], ins,
                                        self._state[bid], now, ctx)
            except Exception as e:  # 块异常：日志 + 本次传播终止，不扩散
                print(f"[graph] 块 {bid} 求值异常: {e}", flush=True)
                continue
            if result:
                self._outputs[bid].update(result)
                produced.add(bid)
        return produced

    def output(self, block_id: str, port: str):
        return self._outputs.get(block_id, {}).get(port)

    def block_params(self, block_id: str) -> dict | None:
        b = self._blocks.get(block_id)
        return b["params"] if b else None
```

- [ ] **Step 4: 跑测试确认通过**

Run: `python test_graph.py`
Expected: 6 个 PASS，退出码 0

- [ ] **Step 5: Commit（确认后）**

```bash
git add server/app/graph_engine.py server/app/test_graph.py
git commit -m "feat: 图引擎校验/拓扑/增量求值 (refs #27)"
```

---

### Task 4: graph_service.py — 默认图 / 表单投影 / 热加载

**Files:**
- Create: `home-monitor/server/app/graph_service.py`
- Test: `home-monitor/server/app/test_graph.py`

**Interfaces:**
- Consumes: `graph_store.GraphStore`（Task 1）、`graph_engine.Graph`/`GraphError`（Task 3）、`blocks`（Task 2）
- Produces: `GraphService(store: GraphStore, profiles: dict, sendkey: str, publish)`；
  `.feed_state(ntype: str, node: str, raw: str, now: float) -> None`（无图自动生成默认图）；
  `.display_of(node: str) -> dict`（{"text","level","alias"}，无则 {}）；
  `.translate_map_of(node: str) -> dict`；
  `.projection(ntype: str, node: str) -> dict`（{"alias","map","cooldown"}，表单读）；
  `.apply_form(ntype: str, node: str, alias: str, map_: dict, cooldown: int) -> dict`（表单写：重建标准子图+落库+热加载，返回 projection）；
  `default_map(profiles: dict, ntype: str) -> dict`

- [ ] **Step 1: 追加失败测试**

```python
def _profiles():
    return {"collision": {"dashboard": {"events": {
        "triggered": {"text": "碰撞触发", "level": "warn"},
        "released": {"text": "碰撞释放", "level": "info"}}}}}

def test_service_default_and_form():
    import tempfile, os
    from graph_store import GraphStore
    from graph_service import GraphService
    db = tempfile.mktemp(suffix=".db")
    svc = GraphService(GraphStore(db), _profiles(), "", None)
    # 默认值策略：无图节点 feed 后自动生成默认图，文案=collision 原始文案
    svc.feed_state("collision", "6750f8", "triggered", 1.0)
    assert svc.display_of("6750f8")["text"] == "碰撞触发"
    # 表单投影默认值
    proj = svc.projection("collision", "6750f8")
    assert proj["alias"] == "" and proj["map"]["triggered"]["text"] == "碰撞触发"
    # 表单写：自定义文案立即生效
    svc.apply_form("collision", "6750f8", "平台窗户",
                   {"triggered": {"text": "门窗打开", "level": "warn"},
                    "released": {"text": "门窗关闭", "level": "info"}}, 60)
    svc.feed_state("collision", "6750f8", "released", 2.0)
    d = svc.display_of("6750f8")
    assert d == {"text": "门窗关闭", "level": "info", "alias": "平台窗户"}, d
    assert svc.translate_map_of("6750f8")["triggered"]["text"] == "门窗打开"
    # 持久化：新实例（模拟重启）图还在
    svc2 = GraphService(GraphStore(db), _profiles(), "", None)
    svc2.feed_state("collision", "6750f8", "triggered", 3.0)
    assert svc2.display_of("6750f8")["text"] == "门窗打开", "重启后图应保留"
    assert svc2.projection("collision", "6750f8")["alias"] == "平台窗户"

def test_service_alert_link():
    import tempfile, os, time
    import blocks
    sent = []
    orig = blocks.send_sct
    blocks.send_sct = lambda k, t, d: sent.append((t, d))
    try:
        from graph_store import GraphStore
        from graph_service import GraphService
        db = tempfile.mktemp(suffix=".db")
        svc = GraphService(GraphStore(db), _profiles(), "", None)
        svc.apply_form("collision", "6750f8", "平台窗户",
                       {"triggered": {"text": "门窗打开", "level": "warn"},
                        "released": {"text": "门窗关闭", "level": "info"}}, 60)
        svc.feed_state("collision", "6750f8", "triggered", 1.0)   # 首值无 event，不告警
        svc.feed_state("collision", "6750f8", "released", 2.0)    # info，不告警
        svc.feed_state("collision", "6750f8", "triggered", 3.0)   # 变化沿 + warn → 告警
        assert len(sent) == 1 and sent[0][0] == "门窗打开告警", sent
        assert "平台窗户" in sent[0][1]
    finally:
        blocks.send_sct = orig

check("service_default_and_form", test_service_default_and_form)
check("service_alert_link", test_service_alert_link)
```

- [ ] **Step 2: 跑测试确认失败**

Run: `python test_graph.py`
Expected: FAIL（ModuleNotFoundError: graph_service）

- [ ] **Step 3: 实现 graph_service.py**

```python
"""图服务（Issue #27）：store+engine+blocks 的装配层，main.py 只与它对。

- 默认值策略：无图节点首次 feed 时按固件类型档案生成默认图（原始文案）
- 表单投影：projection/apply_form 读写节点标准子图（固定块 id in1/sem1/disp1/alm1）
- 图是唯一真身：注册表/语义档案概念不存在于本层
"""

import time

from graph_engine import Graph, GraphError
from graph_store import GraphStore


def default_map(profiles: dict, ntype: str) -> dict:
    """默认翻译表 = 固件类型档案的 dashboard.events（原始词汇文案）。"""
    p = profiles.get(ntype) or {}
    return dict(((p.get("dashboard") or {}).get("events")) or {})


def standard_graph(ntype: str, node: str, alias: str, map_: dict, cooldown: int) -> dict:
    """节点标准子图：io_in → translate → display / alert（固定块 id，表单投影约定）。"""
    return {"blocks": [
        {"id": "in1", "kind": "io_in",
         "params": {"topic": f"{ntype}/{node}/state"}},
        {"id": "sem1", "kind": "translate", "params": {"map": map_}},
        {"id": "disp1", "kind": "display", "params": {"alias": alias}},
        {"id": "alm1", "kind": "alert",
         "params": {"cooldown": int(cooldown), "channel": "sct"}},
    ], "wires": [
        {"from": "in1.state", "to": "sem1.in"},
        {"from": "sem1.out", "to": "disp1.state"},
        {"from": "sem1.event", "to": "alm1.trigger"},
    ]}


class GraphService:
    def __init__(self, store: GraphStore, profiles: dict, sendkey: str, publish):
        self._store = store
        self._profiles = profiles
        self._sendkey = sendkey
        self._publish = publish
        self._graphs: dict[str, Graph] = {}
        self._specs: dict[str, dict] = {}  # 重建用（Graph 内部结构不暴露 spec）
        for nid, spec in store.load_all().items():
            try:
                self._graphs[nid] = Graph(spec)
                self._specs[nid] = spec
            except GraphError as e:  # 坏图跳过，节点退化兜底显示，不拖垮服务
                print(f"[graph] 节点 {nid} 图加载失败跳过: {e}", flush=True)

    def _ensure(self, ntype: str, node: str) -> Graph:
        if node not in self._graphs:
            spec = standard_graph(ntype, node, "", default_map(self._profiles, ntype), 60)
            self._graphs[node] = Graph(spec)
            self._specs[node] = spec  # 默认图不落库：用户保存表单时才持久化
            print(f"[graph] {node}: 生成默认图（{ntype}）", flush=True)
        return self._graphs[node]

    def _ctx(self, node: str) -> dict:
        alias = (self._graphs[node].block_params("disp1") or {}).get("alias", "")
        return {"sendkey": self._sendkey, "node": node, "alias": alias,
                "publish": self._publish}

    def feed_state(self, ntype: str, node: str, raw: str, now: float) -> None:
        g = self._ensure(ntype, node)
        g.feed("in1", {"state": {"raw": raw, "ts": now}}, now, self._ctx(node))

    def display_of(self, node: str) -> dict:
        g = self._graphs.get(node)
        return (g.output("disp1", "view") or {}) if g else {}

    def translate_map_of(self, node: str) -> dict:
        g = self._graphs.get(node)
        return ((g.block_params("sem1") or {}).get("map") or {}) if g else {}

    def projection(self, ntype: str, node: str) -> dict:
        """表单读：从标准块 params 提取；无图按默认值投影（不落库）。"""
        self._ensure(ntype, node)
        g = self._graphs[node]
        return {
            "alias": (g.block_params("disp1") or {}).get("alias", ""),
            "map": (g.block_params("sem1") or {}).get("map")
                   or default_map(self._profiles, ntype),
            "cooldown": (g.block_params("alm1") or {}).get("cooldown", 60),
        }

    def apply_form(self, ntype: str, node: str, alias: str,
                   map_: dict, cooldown: int) -> dict:
        """表单写：重建标准子图 + 落库 + 热加载。GraphError 由 API 层转 400。"""
        spec = standard_graph(ntype, node, alias, map_, cooldown)
        g = Graph(spec)  # 先校验，非法图不入库
        self._store.save(node, spec)
        self._graphs[node] = g
        self._specs[node] = spec
        print(f"[graph] {node}: 表单已应用（alias={alias!r}）", flush=True)
        return self.projection(ntype, node)
```

- [ ] **Step 4: 跑测试确认通过**

Run: `python test_graph.py`
Expected: 8 个 PASS，退出码 0

- [ ] **Step 5: Commit（确认后）**

```bash
git add server/app/graph_service.py server/app/test_graph.py
git commit -m "feat: GraphService 默认图/表单投影/热加载 (refs #27)"
```

---

### Task 5: main.py 整合 — 硬编码翻译/告警切换为图引擎

**Files:**
- Modify: `home-monitor/server/app/main.py`（整体替换，含撤销旧方案 registry 改动）
- Delete: `home-monitor/server/app/registry.py`（旧方案产物，graphs 表取代）
- Test: `home-monitor/server/app/test_graph.py`

**Interfaces:**
- Consumes: `graph_service.GraphService`、`graph_store.GraphStore`、`graph_engine.GraphError`、`events.EventStore`
- Produces: `main._on_state(ntype, node, state, cached, retained, gsvc, store, discover) -> None`；`GET /api/nodes/{node}/config -> {"alias","map","cooldown"}`；`PUT /api/nodes/{node}/config`（body 同 GET）-> 同 GET；`/nodes` 条目新增 `display: dict`、`map: dict` 字段；`app.state.gsvc`

- [ ] **Step 1: 追加失败测试**（stub 第三方包后直调 main._on_state 全链路）

```python
def test_main_on_state_link():
    import tempfile, os
    for name in ["fastapi", "pydantic", "paho", "paho.mqtt", "paho.mqtt.client"]:
        sys.modules[name] = types.ModuleType(name)
    class _F:
        def __init__(self, *a, **k): pass
        def get(self, *a, **k): return lambda f: f
        def put(self, *a, **k): return lambda f: f
    sys.modules["fastapi"].FastAPI = _F
    sys.modules["fastapi"].HTTPException = Exception
    fr = types.ModuleType("fastapi.responses")
    fr.FileResponse = lambda *a, **k: None
    sys.modules["fastapi.responses"] = fr
    sys.modules["pydantic"].BaseModel = object
    sys.modules["paho.mqtt.client"].Client = _F
    sys.modules["paho.mqtt.client"].CallbackAPIVersion = types.SimpleNamespace(VERSION2=2)
    import events
    from graph_store import GraphStore
    from graph_service import GraphService
    import main
    db = tempfile.mktemp(suffix=".db")
    store = events.EventStore(db)
    gsvc = GraphService(GraphStore(db), _profiles(), "", None)
    main._on_state("collision", "6750f8", "triggered", False, False,
                   gsvc, store, lambda t, n: None)
    assert main.nodes["6750f8"]["state"] == "triggered"
    assert gsvc.display_of("6750f8")["text"] == "碰撞触发", "图引擎应产出 display"
    ev = store.query(limit=1)[0]
    assert ev["payload"]["state"] == "triggered", "事件落原始值"
    # retained 重放：不记事件、不喂图
    main._on_state("collision", "6750f8", "released", False, True,
                   gsvc, store, lambda t, n: None)
    assert gsvc.display_of("6750f8")["text"] == "碰撞触发", "retained 不应喂图"
    store.close(); os.unlink(db)

check("main_on_state_link", test_main_on_state_link)
```

- [ ] **Step 2: 跑测试确认失败**

Run: `python test_graph.py`
Expected: FAIL（main.py 里 `_on_state` 签名还是旧的 / registry 残留）

- [ ] **Step 3: 整体替换 main.py**

```python
"""home-monitor 服务端：MQTT 消费 / 流转发 / 检测 / 告警 / Web。

功能规划见 docs/design.md。当前能力：
- 节点状态事件 → 事件落盘 + 图引擎求值（翻译/显示/告警是图输出，Issue #27）
- 节点在线状态跟踪（LWT）+ 健康指标（rssi/uptime），GET /nodes 查看
- 事件落盘 SQLite（events.py），GET /api/events 查历史（重启不丢）
- 看板配置表单 = 节点标准子图的投影（GET/PUT /api/nodes/{id}/config）
"""

import os
import time
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException
from fastapi.responses import FileResponse
from pydantic import BaseModel

from discovery import discovery_messages
from events import EventStore
from graph_engine import GraphError
from graph_service import GraphService
from graph_store import GraphStore
from mqtt_client import start_mqtt
from profiles import load_profiles

# 节点状态表：{node: {"type","status","state","rssi","uptime","ts"}}（引擎外元信息）
nodes: dict[str, dict] = {}
# 物模型类型档案（lifespan 启动时加载；模板文件也在其中）
PROFILES: dict[str, dict] = {}


def _ensure_node(ntype: str, node: str, discover) -> dict:
    """状态表建条目；新节点且类型有档案时触发 HA discovery 代发。"""
    if node not in nodes:
        entry = nodes.setdefault(node, {})
        entry["type"] = ntype
        discover(ntype, node)
        return entry
    return nodes[node]


def _on_state(ntype: str, node: str, state: str, cached: bool, retained: bool,
              gsvc: GraphService, store: EventStore, discover) -> None:
    entry = _ensure_node(ntype, node, discover)
    entry["state"] = state
    entry["ts"] = time.time()
    if retained:
        return  # retain 重放只更新状态表，不记事件、不喂图、不告警
    store.record(ntype, node, "state", {"state": state, "cached": cached})
    gsvc.feed_state(ntype, node, state, entry["ts"])  # 翻译/显示/告警全在图里


def _on_status(ntype: str, node: str, status: str, retained: bool,
               store: EventStore, discover) -> None:
    entry = _ensure_node(ntype, node, discover)
    entry["status"] = status
    entry["ts"] = time.time()
    if not retained:
        store.record(ntype, node, "status", {"status": status})


def _on_health(ntype: str, node: str, rssi, uptime, discover) -> None:
    entry = _ensure_node(ntype, node, discover)
    entry["rssi"] = rssi
    entry["uptime"] = uptime
    entry["ts"] = time.time()


@asynccontextmanager
async def lifespan(app: FastAPI):
    db_path = os.getenv("EVENTS_DB", "/data/events.db")
    store = EventStore(db_path)
    PROFILES.update(load_profiles(os.getenv("PROFILES_DIR", "/srv/nodetypes")))

    def discover(ntype: str, node: str) -> None:
        """新节点出现：按档案代发 HA discovery config（retain，幂等）。"""
        profile = PROFILES.get(ntype)
        if not profile:
            return
        for topic, payload in discovery_messages(ntype, node, profile):
            client.publish(topic, payload, retain=True)
        print(f"[discovery] {ntype}/{node} config 已代发", flush=True)

    # publish 闭包延迟绑定 client（io_out 块用；client 在下方 start_mqtt 才赋值）
    def publish(topic: str, payload: str):
        client.publish(topic, payload)

    gsvc = GraphService(GraphStore(db_path), PROFILES,
                        os.getenv("SCT_SENDKEY", ""), publish)

    client = start_mqtt(
        host=os.getenv("MQTT_HOST", "localhost"),
        port=int(os.getenv("MQTT_PORT", "1883")),
        user=os.getenv("MQTT_USER", ""),
        password=os.getenv("MQTT_PASS", ""),
        on_state=lambda t, n, s, c, r: _on_state(t, n, s, c, r, gsvc, store, discover),
        on_status=lambda t, n, s, r: _on_status(t, n, s, r, store, discover),
        on_health=lambda t, n, r, u: _on_health(t, n, r, u, discover),
    )
    app.state.mqtt = client
    app.state.events = store
    app.state.gsvc = gsvc
    yield
    client.loop_stop()
    client.disconnect()
    store.close()


app = FastAPI(title="home-monitor", lifespan=lifespan)

_STATIC = os.path.join(os.path.dirname(__file__), "static")


@app.get("/")
def dashboard():
    """监控看板：图输出驱动的单页（无框架，5s 轮询 /nodes + /api/events）。"""
    return FileResponse(os.path.join(_STATIC, "index.html"))


@app.get("/health")
def health() -> dict:
    return {"status": "ok", "mqtt_connected": app.state.mqtt.is_connected()}


@app.get("/nodes")
def list_nodes() -> dict:
    """节点状态表 + 图输出：display=显示点输出（{text,level,alias}），
    map=翻译表（看板事件列表把历史原始事件翻译成当前语义文案）。"""
    return {nid: {**entry,
                  "display": app.state.gsvc.display_of(nid),
                  "map": app.state.gsvc.translate_map_of(nid)}
            for nid, entry in nodes.items()}


class NodeForm(BaseModel):
    alias: str = ""
    map: dict = {}
    cooldown: int = 60


def _ntype_of(node: str) -> str:
    """节点固件类型：状态表里有就用；未知节点按 collision（当前唯一固件类型）。"""
    return nodes.get(node, {}).get("type", "collision")


@app.get("/api/nodes/{node}/config")
def get_node_config(node: str) -> dict:
    """表单投影读（Issue #27）：alias/map/cooldown，无图节点按默认值投影。"""
    return app.state.gsvc.projection(_ntype_of(node), node)


@app.put("/api/nodes/{node}/config")
def set_node_config(node: str, form: NodeForm) -> dict:
    """表单投影写：重建节点标准子图，热加载生效，无需改固件。"""
    try:
        return app.state.gsvc.apply_form(_ntype_of(node), node,
                                         form.alias.strip(), form.map, form.cooldown)
    except GraphError as e:
        raise HTTPException(400, f"非法图: {e}")


@app.get("/api/events")
def list_events(limit: int = 50, node: str | None = None) -> list[dict]:
    """事件时间线（SQLite 落盘，重启不丢）：按时间倒序，可按节点过滤。"""
    return app.state.events.query(limit=min(limit, 200), node=node)


@app.get("/api/profiles")
def list_profiles() -> dict:
    """类型档案与模板（collision=IO 词汇+HA discovery；contact/presence=表单预填模板）。"""
    return PROFILES
```

同时删除旧方案文件：`git rm -f server/app/registry.py`（该文件未被跟踪则 `rm server/app/registry.py`）。

行为变更备忘（写进 commit message / PR 描述）：
- 冷却时间从 env `ALERT_COOLDOWN_SECONDS` 改为图 params（表单可配，默认 60s），env 废弃
- contact.json 去掉 ha 段后，现存 contact 类型节点不再重发 HA discovery；broker 里 retain 的旧 config 仍有效，固件刷为 collision 后由 collision.json 接管

- [ ] **Step 4: 跑测试确认通过**

Run: `python test_graph.py`
Expected: 9 个 PASS，退出码 0

- [ ] **Step 5: Commit（确认后）**

```bash
git add server/app/main.py server/app/test_graph.py
git rm -f server/app/registry.py 2>/dev/null || rm server/app/registry.py
git commit -m "feat: _on_state 切换图引擎；配置 API 改表单投影；冷却时间入图 params (refs #27)"
```

---

### Task 6: index.html — 图输出渲染 + 新配置表单

**Files:**
- Modify: `home-monitor/server/app/static/index.html`（整体替换）

**Interfaces:**
- Consumes: `/nodes`（条目含 `display{text,level,alias}`、`map`、`type/status/state/rssi/uptime`）、`/api/events`、`/api/profiles`（含 `template` 的条目是预填模板）、`GET/PUT /api/nodes/{id}/config`（Task 5）
- Produces: 前端不再做 raw→语义翻译；事件列表用 `map` 翻译；配置表单 PUT `{alias, map, cooldown}`

- [ ] **Step 1: 整体替换 index.html**

```html
<!DOCTYPE html>
<html lang="zh">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>home-monitor 看板</title>
<style>
  * { box-sizing: border-box; margin: 0; }
  body { font-family: system-ui, sans-serif; background: #0f1420; color: #dde3ee;
         max-width: 720px; margin: 0 auto; padding: 12px; }
  h1 { font-size: 18px; padding: 8px 0; }
  h2 { font-size: 14px; color: #8a93a8; padding: 16px 0 8px; }
  .grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 10px; }
  .card { background: #1a2130; border-radius: 10px; padding: 12px; border: 1px solid #2a3348; }
  .card .label { font-size: 12px; color: #8a93a8; }
  .card .primary { font-size: 22px; font-weight: 600; margin: 6px 0; }
  .card .meta { font-size: 12px; color: #aab3c8; line-height: 1.7; }
  .badge { display: inline-block; font-size: 11px; padding: 1px 8px; border-radius: 8px; margin-left: 6px; }
  .ok    { color: #4ade80; } .badge.ok    { background: #14532d; color: #4ade80; }
  .warn  { color: #fbbf24; } .badge.warn  { background: #553a0f; color: #fbbf24; }
  .info  { color: #60a5fa; } .badge.info  { background: #1e3a5f; color: #60a5fa; }
  .off   { color: #6b7280; } .badge.off   { background: #374151; color: #9ca3af; }
  .gear { float: right; background: none; border: none; color: #8a93a8; cursor: pointer; font-size: 13px; }
  .gear:hover { color: #dde3ee; }
  #cfg { background: #1a2130; border: 1px solid #2a3348; border-radius: 10px; padding: 12px; margin-bottom: 10px; font-size: 13px; line-height: 2; }
  #cfg input, #cfg select { background: #0f1420; color: #dde3ee; border: 1px solid #2a3348; border-radius: 6px; padding: 4px 8px; margin: 0 6px 0 0; }
  #cfg button { background: #1e3a5f; color: #60a5fa; border: none; border-radius: 6px; padding: 5px 12px; cursor: pointer; margin-right: 6px; }
  #cfg .hint { color: #8a93a8; font-size: 11px; line-height: 1.5; }
  .evt { background: #1a2130; border-radius: 8px; padding: 8px 10px; margin-bottom: 6px;
         font-size: 13px; display: flex; gap: 8px; align-items: baseline; }
  .evt .time { color: #8a93a8; font-size: 11px; white-space: nowrap; }
  .evt .node { color: #aab3c8; font-size: 11px; }
  pre { font-size: 11px; color: #aab3c8; white-space: pre-wrap; word-break: break-all; }
</style>
</head>
<body>
<h1>home-monitor 看板</h1>
<div id="cfg" style="display:none"></div>
<h2>节点</h2>
<div class="grid" id="nodes"></div>
<h2>事件</h2>
<div id="events"></div>

<script>
const $ = (id) => document.getElementById(id);
const esc = (s) => String(s).replace(/[&<>"]/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c]));

function fmtDuration(s) {
  if (s == null) return '-';
  s = Math.floor(s);
  const d = Math.floor(s / 86400), h = Math.floor(s % 86400 / 3600), m = Math.floor(s % 3600 / 60);
  return d ? `${d}天${h}时` : h ? `${h}时${m}分` : m ? `${m}分${s % 60}秒` : `${s}秒`;
}
function fmtTime(ts) {
  const d = new Date(ts * 1000);
  return `${d.getMonth()+1}/${d.getDate()} ${String(d.getHours()).padStart(2,'0')}:${String(d.getMinutes()).padStart(2,'0')}:${String(d.getSeconds()).padStart(2,'0')}`;
}

// 卡片渲染：显示文案全部来自图输出（display），前端不懂翻译
function nodeCard(node, id) {
  const online = node.status === 'online';
  const badge = `<span class="badge ${online ? 'ok' : 'off'}">${online ? '在线' : '离线'}</span>`;
  const gear = `<button class="gear" onclick="openCfg('${esc(id)}')" title="配置节点">⚙</button>`;
  const d = node.display || {};
  const title = d.alias ? `${esc(d.alias)} / ${esc(id)}` : `${esc(node.type || '?')} / ${esc(id)}`;
  const primary = d.text
    ? `<div class="primary ${d.level || 'info'}">${esc(d.text)}</div>`
    : `<div class="primary info">${esc(node.state ?? '-')}</div>`;  // 图尚未求值时兜底原始值
  let fields = '';
  if (node.rssi != null) fields += `<div>信号: ${esc(node.rssi)}dBm</div>`;
  if (node.uptime != null) fields += `<div>在线: ${esc(fmtDuration(node.uptime))}</div>`;
  return `<div class="card"><div class="label">${title} ${badge}${gear}</div>
    ${primary}<div class="meta">${fields}</div></div>`;
}

// 事件列表：历史事件落的是原始值，用节点当前翻译表（map）翻译成当前语义文案
function eventText(e, nodeOf) {
  if (e.kind === 'state') {
    const raw = e.payload && e.payload.state;
    const m = ((nodeOf(e.node) || {}).map || {})[raw] || { text: raw, level: 'info' };
    return `<span class="${m.level}">${esc(m.text)}</span>` +
      (e.payload.cached ? ' <span class="badge info">补发</span>' : '');
  }
  if (e.kind === 'status') {
    return e.payload.status === 'online'
      ? '<span class="ok">节点上线</span>' : '<span class="off">节点离线</span>';
  }
  return `<span class="info">${esc(e.kind)}</span> <pre style="display:inline">${esc(JSON.stringify(e.payload))}</pre>`;
}

// ---- 节点配置（Issue #27）：表单 = 节点子图的投影 ----
// 用户直接定义每个原始值（triggered/released）对应的文案与告警级别；
// 模板下拉只是预填，保存后与模板无关联。
let cfgNode = null;
let cfgMap = {};
const RAW_STATES = ['triggered', 'released'];

async function openCfg(id) {
  cfgNode = id;
  const cfg = await fetch(`/api/nodes/${encodeURIComponent(id)}/config`).then(r => r.json());
  cfgMap = cfg.map || {};
  const tplOpts = ['<option value="">模板预填…</option>'].concat(
    Object.entries(lastProfiles).filter(([, p]) => p.template)
      .map(([t, p]) => `<option value="${esc(t)}">${esc(p.label || t)}</option>`));
  const rows = RAW_STATES.map(raw => {
    const m = cfgMap[raw] || { text: '', level: 'info' };
    const lv = (v) => `<select id="cfgLv_${raw}">
      ${['info', 'ok', 'warn'].map(l => `<option value="${l}" ${m.level === l ? 'selected' : ''}>${l}</option>`).join('')}</select>`;
    return `<div>${esc(raw)}（${raw === 'triggered' ? '触发' : '释放'}时）:
      文案 <input id="cfgTx_${raw}" size="12" value="${esc(m.text)}"> 级别 ${lv()}</div>`;
  }).join('');
  $('cfg').style.display = 'block';
  $('cfg').innerHTML = `<b>配置节点 ${esc(id)}</b><br>
    别名 <input id="cfgAlias" size="16" placeholder="如：平台窗户" value="${esc(cfg.alias || '')}">
    告警冷却 <input id="cfgCd" size="4" type="number" min="0" value="${esc(cfg.cooldown ?? 60)}"> 秒
    <select id="cfgTpl" onchange="applyTpl()">${tplOpts.join('')}</select>
    ${rows}
    <div><button onclick="saveCfg()">保存</button><button onclick="$('cfg').style.display='none'">取消</button></div>
    <div class="hint">你在定义这个传感器 0/1 各是什么意思：文案=显示与告警用语，warn 级别才推送告警。
    模板只是预填，可随意改。保存后立即生效，无需改固件。</div>`;
}
function applyTpl() {
  const t = $('cfgTpl').value;
  const tpl = lastProfiles[t] && lastProfiles[t].template;
  if (!tpl) return;
  for (const raw of RAW_STATES) {
    const m = (tpl.map || {})[raw];
    if (m) { $('cfgTx_' + raw).value = m.text || ''; $('cfgLv_' + raw).value = m.level || 'info'; }
  }
}
async function saveCfg() {
  const map = {};
  for (const raw of RAW_STATES) {
    map[raw] = { text: $('cfgTx_' + raw).value, level: $('cfgLv_' + raw).value };
  }
  const body = { alias: $('cfgAlias').value, map, cooldown: Number($('cfgCd').value) || 60 };
  const r = await fetch(`/api/nodes/${encodeURIComponent(cfgNode)}/config`, {
    method: 'PUT', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(body) });
  if (!r.ok) { alert('保存失败: ' + await r.text()); return; }
  $('cfg').style.display = 'none';
  refresh();
}

let lastProfiles = {};
async function refresh() {
  try {
    const [profiles, nodes, events] = await Promise.all([
      fetch('/api/profiles').then(r => r.json()),
      fetch('/nodes').then(r => r.json()),
      fetch('/api/events?limit=30').then(r => r.json()),
    ]);
    lastProfiles = profiles;
    const nodeOf = (nid) => nodes[nid];
    $('nodes').innerHTML = Object.entries(nodes)
      .map(([id, n]) => nodeCard(n, id)).join('') || '<div class="card">无节点</div>';
    $('events').innerHTML = events
      .map(e => `<div class="evt"><span class="time">${fmtTime(e.ts)}</span><span class="node">${esc(((nodeOf(e.node) || {}).display || {}).alias || e.node)}</span>${eventText(e, nodeOf)}</div>`)
      .join('') || '<div class="evt">无事件</div>';
  } catch (err) {
    console.error(err);
  }
}
refresh();
setInterval(refresh, 5000);
</script>
</body>
</html>
```

- [ ] **Step 2: 语法冒烟**

Run: `cd home-monitor/server/app && python -c "import pathlib; h=pathlib.Path('static/index.html').read_text(encoding='utf-8'); assert 'openCfg' in h and 'display' in h; print('HTML-OK')"`
Expected: `HTML-OK`（浏览器端验证在 Task 7 docker 部署后由用户做）

- [ ] **Step 3: Commit（确认后）**

```bash
git add server/app/static/index.html
git commit -m "feat: 看板渲染读图输出 display；配置表单改为 0/1 文案+级别投影 (refs #27)"
```

---

### Task 7: nodetypes 模板化 + 全量验证 + PR1

**Files:**
- Modify: `home-monitor/server/app/nodetypes/contact.json`（整体替换为模板文件）
- Modify: `home-monitor/server/app/nodetypes/presence.json`（整体替换为模板文件）
- 不动: `home-monitor/server/app/nodetypes/collision.json`（IO 词汇 + HA discovery + 默认文案来源）

**Interfaces:**
- Consumes: `profiles.load_profiles` 按 `type` 索引不变；`graph_service.default_map` 读 `profiles[ntype].dashboard.events`
- Produces: 模板文件 schema `{"type","label","template":{"map":{...}}}`（无 ha/dashboard/raw_map）

- [ ] **Step 1: contact.json 整体替换**

```json
{
  "type": "contact",
  "label": "门窗",
  "comment": "表单预填模板（Issue #27 图引擎）：只是帮用户预填翻译块 map，与系统无持续关联。原 ha/dashboard 职责已由 collision.json（硬件档案）与图输出接管。",
  "template": {
    "map": {
      "triggered": { "text": "门窗打开", "level": "warn" },
      "released":  { "text": "门窗关闭", "level": "info" }
    }
  }
}
```

- [ ] **Step 2: presence.json 整体替换**

```json
{
  "type": "presence",
  "label": "在位",
  "comment": "表单预填模板（Issue #27 图引擎）：物体在位/移开场景。",
  "template": {
    "map": {
      "triggered": { "text": "物体在位", "level": "info" },
      "released":  { "text": "物体移开", "level": "warn" }
    }
  }
}
```

- [ ] **Step 3: 全量验证**

```bash
cd home-monitor/server/app
python test_graph.py            # 9 个 PASS
python -m py_compile main.py graph_engine.py graph_service.py graph_store.py blocks.py alerter.py mqtt_client.py events.py profiles.py discovery.py
python -c "import json,glob; [json.load(open(f,encoding='utf-8')) for f in glob.glob('nodetypes/*.json')]; print('JSON-OK')"
```

Expected: 全过

- [ ] **Step 4: Commit（确认后）**

```bash
git add server/app/nodetypes/contact.json server/app/nodetypes/presence.json
git commit -m "feat: contact/presence 降级为表单预填模板 (refs #27)"
```

- [ ] **Step 5: docker 重建 + 用户界面验证**

```powershell
cd D:\home\code\iot\home-monitor\codespace\home-monitor\server
docker compose up -d --build
```

用户验证清单（浏览器 `Ctrl+F5`）：
1. 卡片显示「碰撞触发/碰撞释放」默认文案（默认图生效），点 ⚙ 开表单
2. 填别名「平台窗户」+ 模板下拉选「门窗」预填 → 保存 → 卡片/事件文案变「门窗打开/关闭」
3. 拨传感器：事件新增「门窗打开」，Server酱收「门窗打开告警：平台窗户（6750f8）…」（冷却 60s）
4. 手改文案（如改成「窗户开啦」）保存 → 再触发 → 全是自定义文案（#27 验收点）
5. 回归：/api/events 历史（旧 open/closed）显示原样原文；mqtt_connected=true

- [ ] **Step 6: 发 PR1**

```bash
export HTTPS_PROXY=http://127.0.0.1:17890 HTTP_PROXY=http://127.0.0.1:17890
"C:/Program Files/GitHub CLI/gh.exe" pr create --base main --title "feat: 图引擎（Niagara 理念）替换硬编码翻译/告警 (refs #27)" --body "设计：personal 仓库 tickets/20260802-node-registry/2026-08-02-graph-engine-design.md
- graph_engine/blocks/graph_store/graph_service 四模块；图存 graphs 表
- IO输入点→语义翻译块→显示点/告警点；IO输出点备好（设备互控地基）
- 表单即图的投影：用户直接配 0/1 文案+级别；contact/presence 降级为预填模板
- 冷却时间从 env ALERT_COOLDOWN_SECONDS 改为图 params（表单可配）
- 测试：server/app/test_graph.py 9 用例全过（零依赖）"
"C:/Program Files/GitHub CLI/gh.exe" pr merge --squash
git checkout main && git pull
```

---

### Task 8: PR2 — 固件 contact-node → collision-node（合并不刷机）

**Files:**
- Rename: `home-monitor/firmware/contact-node/` → `home-monitor/firmware/collision-node/`
- Modify: `home-monitor/firmware/collision-node/src/main.cpp`
- Modify: `home-monitor/docs/design.md`（目录树一处引用）

- [ ] **Step 1: 改名目录并替换语义**

```bash
cd home-monitor
git mv firmware/contact-node firmware/collision-node
```

`main.cpp` 全部替换点（逐处 Edit）：
- 头注释 `contact-node：ESP8266/ESP-01 + 干簧管 门窗传感器节点` → `collision-node：ESP8266/ESP-01 + 碰撞/接触传感器节点（布尔二值上报器）`；补两行：`线上协议为原始布尔 {"state":1|0}（1=触发/0=释放），语义（门窗/在位…）由服务端图引擎配置（Issue #27），固件不猜部署语义`
- `TOPIC_SYNC`/`TOPIC_SYNCREQ`：`"contact/sync"` → `"collision/sync"`，`"contact/syncreq"` → `"collision/syncreq"`（注释同步改）
- `nodeId`：`snprintf(nodeId, sizeof(nodeId), "contact-%06x", ESP.getChipId())` → `"collision-%06x"`；三个 topic 的 `nodeId + 8` → `nodeId + 10`（"collision-" 10 字符）
- topic 前缀：`"contact/%s/state"`→`"collision/%s/state"`，status/health 同理
- 状态值布尔化：`publishState(const char* state, ...)` → `publishState(int state, ...)`，payload `{\"node\":\"%s\",\"state\":%d%s}`；`cacheEvent/flushEvents/eventsPending` 缓存行存 `"1"`/`"0"`，`flushEvents` 里 `s == "open" || s == "closed"` → `s == "1" || s == "0"` 且 `publishState(s.toInt(), true)`
- 电平→布尔：`lastStableState == HIGH ? "closed" : "open"` → `== HIGH ? 0 : 1`（HIGH=释放=0，LOW=触发=1）；`mqttConnect` 里 `publishState(digitalRead(PIN_REED) == HIGH ? "closed" : "open")` → `publishState(digitalRead(PIN_REED) == HIGH ? 0 : 1)`
- 配网热点：`wm.autoConnect("contact-node-setup")` → `"collision-node-setup"`；WiFiManager 注释里的热点名同步
- 其余注释中的 `contact/` topic 引用全局替换为 `collision/`
- `design.md` 目录树 `└── contact-node/` → `└── collision-node/`

- [ ] **Step 2: 编译验证（不刷机）**

```bash
cd home-monitor/firmware/collision-node
PLATFORMIO_CORE_DIR=C:/.platformio C:/.platformio/penv/Scripts/pio.exe run
```

Expected: SUCCESS（无编译错误）

- [ ] **Step 3: Commit + PR2（确认后）**

```bash
git add firmware docs/design.md
git commit -m "feat: contact-node 改名 collision-node，状态值布尔化 1/0 (closes #27)"
"C:/Program Files/GitHub CLI/gh.exe" pr create --base main --title "feat: 固件 contact→collision 布尔二值上报 (closes #27)" --body "固件退化为诚实布尔上报器：topic/类型改 collision，状态值 open/closed→{\"state\":1|0}（1=触发/0=释放）。合并不刷机——窗户节点 6750f8 跑 #9 长稳到 8/5，满期后刷机验证。"
"C:/Program Files/GitHub CLI/gh.exe" pr merge --squash
git checkout main && git pull
```

- [ ] **Step 4: 刷机验证（8/5 之后，本计划外跟踪）**

刷 collision-node 到 6750f8 → 配网（MQTT host 不变）→ 触发传感器：看板/事件/Server酱 文案与配置一致（contact.json 模板兼容旧词逻辑已无，历史 open/closed 事件按未知值透传显示原样，符合设计）。

---

## 自查记录

- Spec 覆盖：引擎/5 块/表单投影/默认值策略/整合点/错误处理/测试/验收 → Task 1-8 全覆盖；io_out 块有实现无连线（第一版无固件订阅端，符合设计"块先备好"）
- 命名一致性：`GraphService.feed_state/display_of/translate_map_of/projection/apply_form`、`Graph.feed/output/block_params`、`GraphStore.save/load/load_all/delete`、`send_sct`、`BLOCKS` 全篇一致
- 旧方案清理：registry.py 删除（Task 5）；main/alerter/index.html/nodetypes 旧改动被整体替换覆盖；mqtt_client 通用化与 design.md 补充保留
