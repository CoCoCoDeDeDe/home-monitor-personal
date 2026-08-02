<!-- markdownlint-disable -->

#

## desc

- Collision switch
- MCU module
- Illuminated switch

## basic params

**work voltage:** 3V-12V
**module size:** 24*14MM
**output type:** Digital output (`DO`) 1 and `1`, with high level voltage close to supply voltage.

## Pin desc

1. VCC - 3V-12V power support
2. GND - connects to the Power Ground (PGND)
3. OUT - High/Low Level Output

## Indicator light desc

- ON - collision
- OFF - no collision

## 实测记录（2026-08-02，ESP8266 NodeMCU @ 3.3V）

- 结构：带翘起杠杆的微动开关，杠杆被压到底时有明显"咔哒"声（机械触点确认感）
- 默认态（杠杆翘起、无碰撞）：指示灯熄灭，OUT 输出低电平
- 触发态（杠杆被压下）：指示灯亮，OUT 输出高电平
- 即 OUT = 高有效（active-high），触发时输出接近 VCC 电压
- 供电用 3.3V 时 OUT 高电平≈3.3V，可直接接 ESP8266 GPIO（不耐 5V，勿用 5V 供电后接 IO）
- 用作门窗传感时：门窗关闭压住杠杆 → OUT 高 → 对应 closed，极性自然吻合
- 指示灯串联在输出侧，触发时亮灯耗电约 1~2mA，电池供电场景可考虑拆除/挑掉限流电阻
