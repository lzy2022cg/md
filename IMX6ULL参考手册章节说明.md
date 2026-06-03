# IMX6ULL 参考手册章节总览

> NXP i.MX6ULL Applications Processor Reference Manual, Rev. 1, 11/2017  
> 共 60 章，4000+ 页

---

## 系统概述类（Ch 1–9）

| 章节 | 英文名 | 说明 |
|------|--------|------|
| Ch 1 | Introduction | 芯片总体介绍、特性、封装 |
| Ch 2 | Memory Maps | 所有外设的内存地址映射（重要！查寄存器基地址用） |
| Ch 3 | Interrupts and DMA Events | 中断号列表、DMA请求源 |
| Ch 4 | External Signals and Pin Multiplexing | 引脚定义与复用功能说明 |
| Ch 5 | Fusemap | 熔丝位，存储启动方式、安全配置等一次性写入数据 |
| Ch 6 | WEIM – External Memory Interface | 外部并行存储器接口（NOR Flash、SRAM等） |
| Ch 7 | System Debug | 系统调试接口（CoreSight、ETM等） |
| Ch 8 | System Boot | 启动流程、启动引脚配置（BOOT_MODE） |
| Ch 9 | Multimedia | 多媒体子系统概述 |

---

## 时钟与总线类（Ch 10–13）

| 章节 | 英文名 | 说明 |
|------|--------|------|
| Ch 10 | CCM – Clock Control Module | **时钟控制**，所有外设时钟的根源，驱动开发必看 |
| Ch 11 | Analog | PLL、振荡器等模拟电路 |
| Ch 12 | ARM Cortex-A7 Platform | CPU核心架构（缓存、MMU等） |
| Ch 13 | APBH-DMA / Crossbar Switch | APB总线DMA控制器、多主总线矩阵 |

---

## 通用外设类（Ch 14–31）

| 章节 | 英文名 | 说明 |
|------|--------|------|
| Ch 14 | Boot ROM | 内部Boot ROM实现细节 |
| Ch 15 | FlexCAN | **CAN总线控制器**（工业通信） |
| Ch 16 | CSI – Camera Sensor Interface | **摄像头传感器接口**（接OV系列摄像头） |
| Ch 17 | DCP – Data Co-Processor | 数据协处理器（AES/SHA硬件加速） |
| Ch 18 | EIM | 外部接口模块 |
| Ch 19 | ECSPI | **增强型SPI控制器**（4路SPI） |
| Ch 20 | ENET – Ethernet MAC | **以太网控制器**（10/100M，两路） |
| Ch 21 | GPC – General Power Controller | 通用电源控制器 |
| Ch 22 | GPIO | **通用输入输出**（控制LED、按键等） |
| Ch 23 | GPT – General Purpose Timer | **通用定时器**（定时/计数/输入捕获） |
| Ch 24 | I2C | **I²C总线控制器**（4路，接传感器/屏幕） |
| Ch 25 | LCDIF | 基础LCD接口 |
| Ch 26 | MIPI CSI-2 / SPDIF | 数字音频/摄像接口 |
| Ch 27 | SNVS – Secure Non-Volatile Storage | **安全非易失存储**（RTC时间、安全密钥） |
| Ch 28 | SPDIF | Sony/Philips数字音频接口 |
| Ch 29 | SJC – System JTAG Controller | JTAG调试控制器 |
| Ch 30 | TEMPMON | **片上温度传感器** |
| Ch 31 | TSC – Touch Screen Controller | **电阻式触摸屏控制器** |

---

## IO与引脚控制（Ch 32）

| 章节 | 英文名 | 说明 |
|------|--------|------|
| Ch 32 | **IOMUXC** – IO Mux Controller | **IO复用控制器**，共390+个寄存器，配置每个引脚的功能和电气特性，**做驱动必用！** |

---

## 显示与存储类（Ch 33–38）

| 章节 | 英文名 | 说明 |
|------|--------|------|
| Ch 33 | KPP – Keypad Port | 矩阵键盘控制器 |
| Ch 34 | **eLCDIF** – Enhanced LCD Interface | **增强LCD接口**，驱动RGB屏（正点原子屏幕就用这个） |
| Ch 35 | **MMDC** – Multi Mode DDR Controller | **DDR内存控制器**（LPDDR2/DDR3） |
| Ch 36 | MQS – Medium Quality Sound | 中等质量音频输出（GPIO模拟PWM音频） |
| Ch 37 | OCOTP_CTRL – OTP Controller | 片上OTP熔丝控制器（读写MAC地址等） |
| Ch 38 | OCRAM | 片上SRAM控制器（内部快速RAM） |

---

## 电源与定时器类（Ch 39–40）

| 章节 | 英文名 | 说明 |
|------|--------|------|
| Ch 39 | **PMU** – Power Management Unit | **电源管理单元**（LDO、电压调节） |
| Ch 40 | **PWM** – Pulse Width Modulation | **PWM控制器**（调光、电机控制等，8路） |

---

## 多媒体处理类（Ch 41–42）

| 章节 | 英文名 | 说明 |
|------|--------|------|
| Ch 41 | **PXP** – Pixel Pipeline | **像素处理引擎**（图像缩放、色彩转换、旋转，硬件加速） |
| Ch 42 | QuadSPI | **四线SPI接口**（接串行NOR Flash，高速） |

---

## 存储与安全类（Ch 43–44）

| 章节 | 英文名 | 说明 |
|------|--------|------|
| Ch 43 | ROMC – ROM Controller with Patch | ROM控制器（支持补丁机制） |
| Ch 44 | RNGB – Random Number Generator | **硬件随机数生成器**（安全应用） |

---

## 音频与DMA类（Ch 45–46）

| 章节 | 英文名 | 说明 |
|------|--------|------|
| Ch 45 | **SAI** – Synchronous Audio Interface | **同步音频接口**（I²S兼容，接音频编解码芯片） |
| Ch 46 | **SDMA** – Smart DMA Controller | **智能DMA控制器**，有独立小CPU，64通道，Linux驱动较复杂 |

---

## 通信接口类（Ch 47–57）

| 章节 | 英文名 | 说明 |
|------|--------|------|
| Ch 47 | SJC – System JTAG Controller | 系统JTAG安全控制器 |
| Ch 48 | SNVS | 安全非易失存储（含硬件RTC） |
| Ch 49 | ECSPI | 增强型SPI详细寄存器 |
| Ch 50 | TEMPMON | 片上温度监控 |
| Ch 51 | TSC | 电阻触摸屏控制器 |
| Ch 52 | **UART** | **串口控制器**（8路UART，调试/通信最常用） |
| Ch 53 | **USB** | **USB控制器**（2路USB OTG，支持Host/Device） |
| Ch 54–57 | 其他外设 | 各类补充外设寄存器定义 |

---

## 存储卡与系统类（Ch 58–60）

| 章节 | 英文名 | 说明 |
|------|--------|------|
| Ch 58 | **uSDHC** – Ultra Secured Digital Host Controller | **SD/MMC/eMMC控制器**（2路，接SD卡/eMMC，Linux启动盘） |
| Ch 59 | **WDOG** – Watchdog Timer | **看门狗定时器**（防系统死机，嵌入式必备） |
| Ch 60 | XTALOSC – Crystal Oscillator | **晶振控制器**（24MHz主晶振、32kHz低速晶振） |

---

## 嵌入式Linux开发最常用章节速查

| 优先级 | 章节 | 用途 |
|--------|------|------|
| ★★★ | Ch 2 — Memory Maps | 查所有外设的寄存器基地址 |
| ★★★ | Ch 10 — CCM 时钟 | 使能外设时钟，写驱动第一步 |
| ★★★ | Ch 32 — IOMUXC | 配置引脚复用功能，驱动必须 |
| ★★★ | Ch 22 — GPIO | 控制I/O引脚（LED、按键） |
| ★★★ | Ch 52 — UART | 串口调试输出 |
| ★★ | Ch 34 — eLCDIF | LCD屏幕驱动 |
| ★★ | Ch 20 — ENET | 以太网驱动 |
| ★★ | Ch 58 — uSDHC | SD卡/eMMC存储 |
| ★★ | Ch 59 — WDOG | 看门狗 |
| ★★ | Ch 40 — PWM | PWM输出控制 |
| ★ | Ch 24 — I2C | I²C外设（传感器、触摸屏） |
| ★ | Ch 19 — ECSPI | SPI外设（Flash、显示屏） |
| ★ | Ch 46 — SDMA | 高性能DMA数据传输 |
| ★ | Ch 16 — CSI | 摄像头接口 |
| ★ | Ch 45 — SAI | 音频编解码 |
