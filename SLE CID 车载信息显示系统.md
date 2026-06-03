## 一、项目概述
### 1.1 项目背景
本项目基于华为 HiSilicon WS63E 星闪（SLE）无线模组与 GEC6818 ARM Linux 开发板，设计并实现了一套车载信息显示（CID）系统。系统融合了无线传感、嵌入式 Linux 人机交互、云端数据上传等多项技术，具备完整的从感知到显示、从本地到云端的数据链路。

### 1.2 项目目标
| 目标项 | 说明 |
| --- | --- |
| 传感器数据采集 | 实时采集 SHT30 温湿度及 MQ-135 空气质量数据 |
| 无线组网传输 | 三节点星闪（SLE）无线网络，实现数据与指令双向传输 |
| 车载界面显示 | GEC6818 800×480 触摸屏渲染 CID 仪表盘界面 |
| 触摸交互控制 | 8 个分区触摸热区，控制座椅电机与氛围灯等等 |
| 云端数据上传 | 通过 MQTT 协议将传感器数据上传至 OneNET 物联网平台可远程查看车内温湿度 |
| 实时天气获取 | 通过获取公网IP调用 ShowAPI 获取实时天气并显示 |


---

## 二、系统总体架构
### 2.1 硬件组成
| 节点 | 硬件型号 | 数量 | 角色 |
| --- | --- | --- | --- |
| Board 1 | HiSilicon WS63E | 1 块 | SLE Server，传感器采集 + 指令路由 |
| Board 2 | HiSilicon WS63E | 1 块 | SLE Client，座椅步进电机控制和氛围灯LED控制 |
| Board 3 | HiSilicon WS63E | 1 块 | SLE Client，读取温湿度+ MQ-135 空气质量)采集 |
| 上位机 | GEC6818（ARM Cortex-A53） | 1 块 | Linux 上位机，CID 界面渲染与云端交互 |


**硬件平台**：HiSilicon WS63E × 3 块，GEC6818 × 1 块

### 2.2 系统架构图
```plain
┌──────────────────────────────────────────────────────────┐
│               GEC6818 上位机（Linux ARM）                 │
│  触摸主循环 		 │ UART 传感器接收线程 │ MQTT 上传线程       │
│  HTTP 天气线程  │ 分区恢复线程                             │
│  LCD /dev/fb0 ←→ 触摸屏 /dev/input/event0                 │
└──────────────┬───────────────────────────────────────────┘
               │ UART /dev/ttySAC1（115200 8N1，双向）
               │
┌──────────────▼──────────────┐
│      Board 1  WS63E         │  SLE Notify F5/R5 ──→  Board 2（座椅电机）
│      SLE Server（主节点）    │
│      SHT30 I2C + MQ-135 ADC │  SLE Notify L1/L0 ──→  Board 3（氛围灯）
└─────────────────────────────┘
               │ MQTT TCP
               ▼
        OneNET 云平台（mqtts.heclouds.com）
```

### 2.3 整体数据流和代码流程图
| 环节 | 发送方 | 接收方 | 协议/接口 | 内容 |
| --- | --- | --- | --- | --- |
| 1 | SHT30 / MQ-135 | Board 3 MCU | I2C / ADC | 温湿度、空气质量原始数据 |
| 2 | Board 1 UART1 TX | GEC6818 /dev/ttySAC1 | uart 115200 8N1 | 11 字节传感器帧 |
| 3 | GEC6818 /dev/ttySAC1 TX | Board 1 UART RX | UART 115200 8N1 | 控制指令 F5/R5/L1/L0 |
| 4 | Board 1 | Board 2 | SLE Notify | 座椅电机步进指令 F5/R5 |
| 5 | Board 1 | Board 3 | SLE Notify | 氛围灯指令 L1/L0 |
| 6 | GEC6818 | LCD Framebuffer | /dev/fb0 mmap | BMP 背景 + 文字叠加渲染 |
| 7 | GEC6818 | ShowAPI 服务器 | HTTP GET Socket | 实时天气 JSON |
| 8 | GEC6818 | OneNET MQTT Broker | MQTT 3.1.1 QoS0 | 传感器数据 JSON |


══════════════════ ① 数据上行：环境监测 ══════════════════

 [SHT30] 温湿度  	┐ I2C(GPIO13/14)

                 	   	├──────────────────────► [Board 3]  每2秒采集

 [MQ-135] 空气质量 ┘ ADC(GPIO9)+DO(GPIO10)  		  │ 打包8字节

                                      					    		          ▼

                                  				星闪 SLE Write ──► [Board 1]（主节点/Server）

                    		                                        						│ 重打包成11字节帧

                                		                            						▼

       			                                     UART1 115200 8N1 ──► [GEC6818]

                                         												                │

                                          	             ┌────────────────────┤	

                                         					           	     ▼                    		▼

                                          	屏幕显示                 MQTT上传

                                      	/dev/fb0叠加渲染       手拼MQTT包(QoS0)

                                      	温湿度/空气质量       ──► [OneNET 云平台]



══════════════════ ② 指令下行：座椅电机 ══════════════════



 [GEC6818] 触摸"座椅上调/下调"

      │ 每60ms发 F5 / R5

      ▼

  UART1 ──► [Board 1] ─角度×2048/360换算步数─► ── 星闪 SLE Notify ──► [Board 2] 收到 Notify

                                                                             							   │

                                                                      ▼

                                                           				       [GPIO11~14]

──►[ULN2003]──►驱动[28BYJ-48 电机]

══════════════════ ③ 指令下行：氛围灯 ══════════════════



 [GEC6818] 触摸"氛围灯"开关(300ms防抖)

      │ 发 L1(开) / L0(关)

      ▼

 ── UART1 ──► [Board 1] ──── 星闪 SLE Notify ──► [Board 3]（传感器+氛围灯节点）

                                                  					    │

                                               						 [GPIO]──► [氛围灯 LED 开/关]



══════════════════ ④ GEC6818 独立任务：实时天气 ══════════════════



 [GEC6818] 天气线程(每10分钟)

      │ ① HTTP取公网IP  ② 带IP请求 ShowAPI

      ▼

   HTTP GET(Socket) ──► [ShowAPI 服务器] ─返回JSON─► 解析"城市 天气 风向" ──► 屏幕天气区

![](https://cdn.nlark.com/yuque/0/2026/svg/67294588/1780222227552-c401d7c5-546f-42e1-a43d-5945c4fca313.svg)

---

## 三、硬件设计
### 3.1 WS63E 三节点连接关系
| 连接关系 | 接口说明 |
| --- | --- |
| Board 1 UART1_TX → GEC6818 /dev/ttySAC1 RX | 传感器数据上报（115200 8N1） |
| GEC6818 /dev/ttySAC1 TX → Board 1 UART1_RX | 电机/灯光控制指令下发 |
| Board 1 ↔ Board 2 | 星闪 SLE 无线（2.4 GHz），电机指令通道 |
| Board 1 ↔ Board 3 | 星闪 SLE 无线（2.4 GHz），灯光指令通道 |


> Board 1 固定本地 MAC 地址为 `11:22:33:44:55:66`，Board 2 和 Board 3 通过扫描精准识别并连接 Board 1。
>

### 3.2 传感器接入
| 传感器 | 接口 | 采集内容 |
| --- | --- | --- |
| SHT30 | I2C | 温度（℃）、湿度（%） |
| MQ-135 | ADC | 空气质量电压（mV）、数字输出（正常/超标） |


### 3.3 接线逻辑图
(1) Board 1(主节点) ↔ GEC6818 串口

![](https://cdn.nlark.com/yuque/0/2026/svg/67294588/1780219701935-558e7f36-6ec1-41f6-8953-c1be4c71cee7.svg)

(2) Board 2(电机节点) → ULN2003 → 28BYJ-48 步进电机

![](https://cdn.nlark.com/yuque/0/2026/svg/67294588/1780219652716-c2853d5e-70be-46ff-9fe2-e39374a4cb82.svg)

(3) Board 3(传感器节点) ↔ SHT30 / MQ-135

![](https://cdn.nlark.com/yuque/0/2026/svg/67294588/1780219466201-e07ac4bb-10cc-443f-9141-d610d91a604b.svg)

(4) 系统总接线图

![](https://cdn.nlark.com/yuque/0/2026/svg/67294588/1780219748117-d1057410-2262-4810-841f-e895854c3c78.svg)

---

## 四、软件实现
### 4.1 星闪(SLE)无线组网
本系统采用 1 Server + 2 Client 的 SLE 星型拓扑,三块板子全靠星闪无线通信,Server 为中心,两个 Client 独立接入、互不直连,数据均经 Server 中转。

● Board 1(Server,主节点):注册 SSAP 服务并持续广播,同时维护 Board 2、Board 3 两个连接(以 conn_id 区分)。向电机节点 Notify 下发指令,接收传感器节点 Write 上报的数据。

● Board 2(Client,电机节点):扫描连接 Board 1,订阅 Notify,解析电机指令 [方向+步数],驱动 28BYJ-48 步进电机。

● Board 3(Client,传感器节点):扫描连接 Board 1,用 Write 主动上报——每 2 秒读取 SHT30 温湿度和 MQ-135 空气质量,打包发给 Server。

数据双向流动:指令源在主节点 → Notify 下发(电机);数据源在从节点 → Write 上报(传感器)。数据在哪端产生,就由哪端发起,正好演示星闪两个相反方向的数据交换。

### 4.2 UART 传感器帧协议
Board 1 与 GEC6818 之间定义了 **11 字节自定义二进制帧**：

| 字节 | 内容 |
| --- | --- |
| [0][1] | 帧头 0xAA 0x55 |
| [2][3] | 温度[高字节][低字节] 温度 × 10（int16，有符号） |
| [4][5] | 湿度[高字节][低字节] 湿度 × 10（uint16） |
| [6][7] | 空气质量[高字节][低字节] MQ-135 电压 mV（uint16） |
| [8] | MQ-135 数字输出（0=正常，1=超标） |
| [9] | XOR 校验（[2]⊕…⊕[8]）进行异或运算 |
| [10] | 帧尾 0x0D |


### 4.3 LCD 界面渲染
+ 通过 `/dev/fb0` mmap 直接操作显存，800×480 分辨率，32 位 BGRA 色深
+ 加载 BMP 背景图并保存副本（`g_bg_image`），文字叠加时先从副本恢复背景，再渲染透明文字，避免色块遮挡
+ 使用自定义 TrueType 渲染库（FreeType 风格）支持中文显示

### 4.4 触摸屏 8 分区交互
屏幕划分为 8 个功能热区：

┌───────────────────────────┬───────────────────────────┐

│  🕐 时间                                                   │  ☁ 天气                                                  │

│  ZONE_TIME                                            │  ZONE_WEATHER                                    │

│  显示当前系统时间                                    │  显示合肥实时天气                                    │

├───────────────┬───────────┴───────┬───────────────────┤

│  🌡 温度                      │  💧 湿度                               │  ▤ 空气质量                          │

│  ZONE_TEMP              │  ZONE_HUMI                       │  ZONE_AIR                           │

│  显示SHT30温度          │  显示SHT30湿度                    │ 显示MQ-135空气质量          │

├───────────────┼───────────────────┼───────────────────┤

│  ⬆ 座椅上调               │  ⬇ 座椅下调                        │  💡 氛围灯                           │

│  ZONE_SEAT_UP         │  ZONE_SEAT_DN                  │  ZONE_LIGHT                      │

│  持续按压每                 │  持续按压每                           │  氛围灯开关                          │

│  60ms重发 F5              │  60ms重发 R5                       │  (300ms防抖)                       │

└───────────────┴───────────────────┴───────────────────┘



| 分区 | 功能 |
| --- | --- |
| ZONE_TIME | 显示当前系统时间 |
| ZONE_WEATHER | 显示合肥实时天气 |
| ZONE_TEMP | 显示 SHT30 温度 |
| ZONE_HUMI | 显示 SHT30 湿度 |
| ZONE_AIR | 显示 MQ-135 空气质量 |
| ZONE_SEAT_UP | 座椅上调（持续按压每 60ms 重发 F5） |
| ZONE_SEAT_DN | 座椅下调（持续按压每 60ms 重发 R5） |
| ZONE_LIGHT | 氛围灯开关（300ms 防抖） |


### 4.5 天气获取
GEC6818 端通过 POSIX TCP Socket 手工构建 HTTP GET 请求(不依赖任何第三方 HTTP 库),分两步获取实时天气:

取公网 IP:先请求公网 IP 查询接口,拿到本机当前的公网 IP 地址;

按 IP 定位查天气:再连接 ShowAPI(ali-weather.showapi.com),调用 /ip-to-weather?ip=<公网IP> 接口,服务端根据 IP 自动定位所在城市并返回该地天气 JSON。

程序从返回 JSON 中解析出城市名、天气描述、风向,拼成显示字符串(如"合肥 晴 东北风")。

### <font style="color:rgb(51, 51, 51);">4.5 天气获取</font>
<font style="color:rgb(51, 51, 51);">通过 POSIX TCP Socket 手工构建 HTTP GET 请求，调用 ShowAPI（</font>`<font style="color:rgb(51, 51, 51);background-color:rgb(243, 244, 244);">ali-weather.showapi.com</font>`<font style="color:rgb(51, 51, 51);">）获取实时天气，每 10 分钟更新一次，无需第三方 HTTP 库。</font>

| 数据流 | 含义 | 单位 |
| --- | --- | --- |
| `temp_value` | SHT30 温度 | ℃ |
| `humidity_value` | SHT30 湿度 | % |
| `historical_record` | MQ-135 电压 | mV |


采用断线重连机制，每 55 秒发送 PINGREQ 心跳保活。

### 4.7 多线程架构
GEC6818 上位机采用多线程并发设计：

| 线程 | 职责 | 唤醒周期 |
| --- | --- | --- |
| 主线程 | 触摸事件处理，发送控制指令 | select() 50ms 轮询 |
| show_sensor_thread | UART 接收传感器帧，更新全局变量 | 阻塞 read()，2s 超时 |
| show_weather_thread | HTTP 拉取天气 | sleep(600)，10 分钟 |
| mqtt_upload_thread | MQTT 上传传感器数据 | sleep(60)，约 60 秒 |
| zone_restore_thread | 信息分区 5 秒倒计时，到期恢复背景 | usleep(100000)，100ms |


共享资源通过 pthread 互斥锁保护（lcd_mutex、g_sensor_mutex、g_weather_mutex 等），保证并发安全。

时刻            谁在动           干了什么                                                                  锁的状态  
─────────────────────────────────────────────────────────────────────  
T=0ms       [系统]             main 启动，铺背景图，创建 4 个子线程        			 lcd_mutex 空闲  
                  [主线程]  	⚡ select() 等触摸，50ms 超时  
		  [sensor]  	💤 阻塞在 read() 等串口数据（最多2s）  
		  [weather] 	⚡ 立刻拉一次天气… 🔒g_weather_mutex 写入       	🔓 g_weather_mutex  
      		  [mqtt]  		 💤 sleep(55)  
		  [zone]    	⚡ 扫一遍5个分区，没到期 → usleep(100ms)      

T=50ms     [主线程]  	⚡ select 超时，没人摸屏 → 再 select(50ms)  
T=100ms   [zone]    	⚡ 醒，扫描分区都没到期 → 再睡 100ms  
T=100ms   [主线程]  	⚡ select 超时 → 再睡  
…（主线程每50ms、zone每100ms，各自空转，互不打扰）…

T=2000ms [sensor]  	⚡ 串口收到一帧传感器数据！  
		  [sensor]  	 解析出 温度/湿度/空气质量  
		  [sensor]  	🔒 g_sensor_mutex → 更新全局变量 → 				🔓g_sensor_mutex  
         	  [sensor] 	🔒 lcd_mutex → 把数值画到屏幕 → 				🔓lcd_mutex  
         	  [sensor]  	 又回去阻塞 read() 等下一帧（2s）              

─────────── 这时用户用手指点了一下"温度"区 ───────────  
T=2300ms [主线程]  	⚡ select 返回！读到触摸"按下"事件  
					 [主线程]  判断落在 ZONE_TEMP 分区  
T=2350ms [主线程]  读到"抬起"事件 → 要显示温度  
		  [主线程]  	🔒 g_sensor_mutex 读温度值 → 					🔓g_sensor_mutex  
        	  [主线程]  	🔒 lcd_mutex 在温度区叠加显示"温度 25.3℃" 		🔓lcd_mutex  
         	  [主线程]  	🔒 g_zone_mutex 记下"这个区5秒后恢复" →   		🔓g_zone_mutex

T=2400ms [zone]           ⚡ 醒，扫描发现 ZONE_TEMP 有了"到期时间"  
       					 [zone]    但还没到（还差5秒）→ 继续每100ms盯着  
					 …（zone 每100ms 检查一次，直到 T=7350ms）…

T=7350ms [zone]    	⚡ 到期了！🔒 lcd_mutex → 把温度区擦回背景               🔓lcd_mutex  
                   			 屏幕恢复原样

   …（一直这样跑，直到 T=55s mqtt 醒来上传、T=600s weather 再拉天气）…

---

## 五、关键技术亮点五、关键技术亮点
| 技术点 | 实现方式 |
| --- | --- |
| SLE 一对二组网 | 主板同时连着两块从板，靠"连接编号（conn_id）"区分谁是谁，命令只发给该发的那一块 |
| 没用现成的 HTTP 库 | 取天气时没装别人的库，自己用最基础的网络接口（Socket）拼出请求、再读回数据 |
| 没用现成的 MQTT 库 | 往云平台（OneNET）传数据时，也是自己按 MQTT 协议格式一个字节一个字节拼出来的 |
| 串口数据防出错 | 串口帧加了帧头帧尾和"异或校验"，收到后核对一遍，数据传错了能发现、丢掉 |
| 文字叠加在背景图上 | 屏幕先铺一张 BMP 背景图，再把文字画上去；遇到全黑（透明）的像素就跳过不画，看起来像贴上去的 |
| 兼容老系统 | GEC6818 系统比较老，把一个网络设置的步骤挪到连接成功之后再做，避免老系统报错连不上 |


---

## 六、项目成果
+ 实现了 WS63E 三节点星闪无线组网，Board 1 作为主节点同时连接两个从节点，指令路由稳定
+ GEC6818 成功渲染 CID 仪表盘界面，触摸交互响应正常，8 个功能分区全部实现
+ 传感器数据经 UART 实时上报，温湿度与空气质量数据正确解析并显示
+ 座椅步进电机（Board 2）与氛围灯（Board 3）均可通过触摸屏远程控制
+ 实时天气（ShowAPI）与 MQTT 云端上传（OneNET）功能正常运行

---

## 七、总结与展望
本项目完整实现了从传感器采集、星闪无线传输、嵌入式 Linux 显示交互到云端数据上传的全链路车载信息系统，覆盖了嵌入式 C、Linux 系统编程、无线通信协议、网络编程等多个技术领域。

**后续可拓展方向：**

+ 扩展更多 SLE Client 节点（如车窗控制、空调控制）
+ 接入华为云 IoTDA，替换 OneNET 平台
+ 使用 Qt 或 LVGL 替换自制 Framebuffer 渲染，提升 UI 表现力
+ 为 SLE 连接添加应用层加密，提升安全性

