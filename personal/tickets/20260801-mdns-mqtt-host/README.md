# 固件：MQTT host 支持 mDNS 主机名（免 IP 重配）

Issue: CoCoCoDeDeDe/home-monitor#16
状态: 已完成（PR #17 已合并，Issue #16 已关闭）
Phase: 1

任务清单与验收标准以 Issue 为准：https://github.com/CoCoCoDeDeDe/home-monitor/issues/16

## 设计要点

- 固件新增最小 mDNS A 查询（~90 行手写组播 224.0.0.251:5353 DNS 报文）：**core 3.1.2 的 LEAmDNS 只有 queryService，没有主机名查询 API**（queryHost 是 ESP32 的，别被文档误导）
- QNAME 必须带 `local` 后缀 label（首版漏了导致查询格式错误无应答）
- 应答校验：匹配 answer 的 NAME 与查询名（组播应答常不带 question 段，不能靠 question 匹配）；首版只取第一条 A 记录，抢到了共享 WiFi 上别的设备的记录（解析出陌生 IP）——不过滤会连错服务器
- 组播无线投递无 ACK 不可靠：查询重发 3 次分段等待
- 解析失败回退原样 host（交系统 DNS）；连接持续失败 >30s 自动重新解析

## 服务器侧要求（Windows）

- WLAN 网络配置改**专用（Private）**：公用配置下 Windows 不应答外部 mDNS 查询（本机自测能通是因为没过防火墙，注意这个测试陷阱）
- 防火墙规则 `home-monitor-mdns-5353`（UDP 5353 入站 Any）
- 两者都已写入 docs/deployment.md

## 验证记录（2026-08-01）

- 串口：`mdns LAPTOP-20260408.local -> 192.168.110.110` → `mqtt connected` ✅
- 开门事件 → app 消费 → Server酱告警 ✅，关门恢复 ✅
- 按键时序坑：RST 之前按住 FLASH 会进 ROM 下载模式（先 RST 松手再在 1.5s 窗口内按 FLASH），已写进部署手册
- 遗留：换网络环境免重配的完整验收要等实际带回家验证

## 过程记录

排障顺序：Public 配置挡外部查询（防火墙规则没用）→ 改 Private 后解析出**错误的** IP（无校验抢到别人的 mDNS 记录）→ 加 NAME 校验 + local 后缀修复 → 解析正确、链路全通。
