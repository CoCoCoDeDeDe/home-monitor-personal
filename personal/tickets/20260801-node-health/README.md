# 固件：节点健康上报（RSSI/uptime）

Issue: CoCoCoDeDeDe/home-monitor#10
状态: 已完成（待 PR 合并关闭 Issue）
Phase: 1

任务清单与验收标准以 Issue 为准：https://github.com/CoCoCoDeDeDe/home-monitor/issues/10

## 实现摘要

- 固件：每 60s 发布 `contact/<id>/health {"node","rssi","uptime"}`，非 retain（健康状态只看实时），编译期常量 `HEALTH_INTERVAL_MS` 控制周期
- app：订阅 `contact/+/health`，on_health(node, rssi, uptime) 回调写入节点注册表，`GET /nodes` 输出新增 rssi/uptime 字段
- 断线则 health 停更，配合 LWT 的 status=offline 可区分弱信号与离线

## 验证记录（2026-08-01）

- `/nodes` 返回 `{"state":"closed","status":"online","rssi":-47,"uptime":60}`，下一周期 uptime:120 持续更新 ✅
- 当前摆位信号 -47dBm（良好），为 #9 上门部署的摆位评估提供基准参考
- 无踩坑，一次通过
