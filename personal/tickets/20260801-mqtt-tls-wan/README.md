# MQTT TLS 加密 + broker 外网安全暴露

Issue: CoCoCoDeDeDe/home-monitor#8
状态: 未开工
Phase: 3（外网访问）

任务清单与验收标准以 Issue 为准：https://github.com/CoCoCoDeDeDe/home-monitor/issues/8

## 决策备忘（2026-08-01 讨论结论）

- TLS 与外网访问绑定做，不单独提前做（纯 LAN 场景 TLS 收益有限）
- 单向 TLS（客户端验服务器 + 用户名密码）优先，双向 TLS 后续视需要
- DDNS 域名是证书 CN/SAN 的前提，先定 DDNS 方案再签证书
- ESP8266 侧关键点：BearSSL WiFiClientSecure、必须先 NTP 校时、握手约 20KB 内存需实测

## 过程记录

（待开工后补充）
