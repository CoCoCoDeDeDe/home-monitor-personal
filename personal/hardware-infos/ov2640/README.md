<!-- markdownlint-disable -->

# OV2640 摄像头模组（排针款，正点原子风格）

淘宝候选商品信息整理，用于 home-monitor #12 摄像头节点。

## 商品参数（商品页）

- 传感器：OV2640（同链接也有 OV5640 SKU，**下单选 OV2640**）
- 接口：DVP 8 位数据 + SCCB（类 IIC）控制接口，**直出无 FIFO**（排针含 VSYNC/HREF/PCLK 直连信号）
- 输出格式：RawRGB、RGB565/555、GRB422、YUV422/420、YCbCr422、JPEG
- 分辨率：UXGA(1600×1200)@15帧、SVGA(800×600)@30帧、CIF(352×288)@60帧，可缩放到任意小尺寸
- 镜头：F2.0、视角 78°、焦距 3.6mm、1/4 英寸传感器
- **滤光片：850nm 感红外滤光片（IR-pass，滤可见光）**
- 电源：3.3V（板载 LDO + 有源晶振，无需外部时钟），功耗 40mA
- IO 电平：2.8V LVTTL，兼容 3.3V
- 板载两颗白光 LED（LED1/LED2），FLASH 引脚控制补光
- 尺寸：24×32mm，工作温度 -30~70℃

## 排针引脚（2.54mm，2×9）

`GND SCL SDA D0 D2 D4 D6 PCLK PWDN / 3V3 VSYNC HREF RST D1 D3 D5 D7 FLASH`

→ 可杜邦线直连 ESP32-S3 N16R8 排针，零焊接

## 适配评估（#12）

- ✓ 排针 2.54mm：免飞线焊接
- ✓ 无 FIFO 直出 DVP：ESP32 DMA 直读可用（下单前跟客服确认"不带FIFO"）
- ✓ 3.3V 供电/IO，与 ESP32-S3 电平兼容
- ⚠️ **850nm IR-pass 滤光片**：白天靠阳光中的红外成分成像，画面为黑白红外风格（无彩色），辨识度可用；夜间需配 850nm 红外补光灯板。板载白光 LED 被 IR-pass 滤掉基本没用
- ⚠️ 同链接 OV5640 是 500 万像素另一型号，勿选错

## info images

- ![desc1](./taobao-desc-1.jpg)
- ![desc2](./taobao-desc-2.jpg)
- ![desc3](./taobao-desc-3.png)
- ![desc4](./taobao-desc-4.png)
