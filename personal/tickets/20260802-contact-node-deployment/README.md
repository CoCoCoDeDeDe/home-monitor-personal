# contact-node 上门部署

Issue: CoCoCoDeDeDe/home-monitor#9

## 目标

把 contact-node 从桌面实验（PC USB 供电、杜邦线短接模拟）变成装在真实门窗上的设备：碰撞传感器实体安装、供电确认、3 天长稳验证，产出部署 checklist。

## 现场条件

- 部署位置：面向平台的窗户（第一优先）
- 传感器：碰撞传感器模组（三线 VCC/GND/OUT，碰撞时 OUT 高电平，资料见 `personal/hardware-infos/collision-sensor-module/`）
- 供电：优先插座；无插座用充电宝过渡（ESP8266 常连 ~70mA，10000mAh 约 5~7 天）
- 服务端：家中笔记本 Docker（LAN 192.168.5.14），看板 http://192.168.5.14:8000
- 节点：NodeMCU COM3，节点 ID contact-6750f8，MQTT host LAPTOP-20260408.local

## 步骤

1. [ ] 桌面接线验证：VCC→3V3（红）、GND→GND（黑）、OUT→D1（黄），手动压开关确认看板事件极性（压住=关门/HIGH，松开=开门/LOW）
2. [ ] 确定安装压点与走线，3M 胶固定模组与板子
3. [ ] 供电方案确认（插座 or 充电宝），无插座位置记录为 #11 输入
4. [ ] 3 天长稳观察：掉线、rssi、误报/漏报，每日记录
5. [ ] 产出 docs/deployment.md checklist（PR refs #9），关闭 issue

## 过程记录
