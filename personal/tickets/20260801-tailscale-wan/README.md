# Tailscale 组网实现外网访问（无公网 IP）

Issue: CoCoCoDeDeDe/home-monitor#8
状态: 未开工
Phase: 3（外网访问）

任务清单与验收标准以 Issue 为准：https://github.com/CoCoCoDeDeDe/home-monitor/issues/8

## 决策备忘（2026-08-01）

- **宽带无公网 IP（CGNAT）**，DDNS + 端口映射路线作废
- 需求重理：板子↔broker 都在 LAN，MQTT 无需公网；告警走 Server酱出站已解决；真正需要外网的只有"手机看实时视频/状态"
- 方案对比后选 **Tailscale 组网**（免费、WireGuard 加密、零端口开放）；备选 Cloudflare Tunnel（浏览器免客户端）/ frp+VPS（付费）暂不采用
- 原 MQTT TLS + broker 8883 公网暴露**取消**；LAN 内 MQTT TLS 降级为可选加固，需要时另开 ticket
- 限制：看视频的设备都要装 Tailscale 客户端（家人共享场景再评估 Cloudflare Tunnel）

## 过程记录

（待开工后补充）
