# Tailscale 组网实现外网访问（无公网 IP）

Issue: CoCoCoDeDeDe/home-monitor#8
状态: 已完成（PR #15 已合并，Issue #8 已关闭）
Phase: 3（外网访问）

任务清单与验收标准以 Issue 为准：https://github.com/CoCoCoDeDeDe/home-monitor/issues/8

## 决策备忘（2026-08-01）

- **宽带无公网 IP（CGNAT）**，DDNS + 端口映射路线作废
- 需求重理：板子↔broker 都在 LAN，MQTT 无需公网；告警走 Server酱出站已解决；真正需要外网的只有"手机看实时视频/状态"
- 方案对比后选 **Tailscale 组网**（免费、WireGuard 加密、零端口开放）；备选 Cloudflare Tunnel（浏览器免客户端）/ frp+VPS（付费）暂不采用
- 原 MQTT TLS + broker 8883 公网暴露**取消**；LAN 内 MQTT TLS 降级为可选加固，需要时另开 ticket
- 限制：看视频的设备都要装 Tailscale 客户端（家人共享场景再评估 Cloudflare Tunnel）

## 过程记录

### 2026-08-01 实施完成

- Windows 服务端：`winget install Tailscale.Tailscale` → `tailscale up` 浏览器登录（GitHub 流程以外也可，个人免费版 100 设备）
- 手机：装 App 同账号登录，关 WiFi 走移动网络访问 `http://<tailscale-ip>:8000/nodes` 与 `/health`，**内容与延迟均正常** ✅
- mirrored 网络下 Docker 发布端口对 Tailscale 接口天然可达，**没有新增防火墙规则**
- 主机自访 Tailscale IP 超时——与 LAN IP 相同的 hairpin 现象，外网设备验证为准
- 部署手册已补 Tailscale 章节（docs/deployment.md，公开仓库不含真实 IP/tailnet 名）
- 待办提醒：#13 迁移旧笔记本时 Tailscale 节点随服务一起迁，届时本机节点可保留作开发用途
