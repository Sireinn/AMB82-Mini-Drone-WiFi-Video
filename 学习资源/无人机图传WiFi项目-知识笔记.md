---
title: 无人机 Wi‑Fi 图传模块：从摄像头到地面端
date: 2026-10-29
tags:
  - 大创项目
  - 无人机
  - WiFi图传
  - AMB82-Mini
  - RTSP
  - 嵌入式
status: 第三版（Obsidian 增强）
规范: "[[无人机图传项目-Obsidian笔记规范]]"
---

# 无人机 Wi‑Fi 图传模块：从摄像头到地面端

这篇笔记从一帧图像出发，说明它怎样从无人机上的摄像头到地面电脑上。

项目当前还没有给出明确的延迟、距离、分辨率和功耗指标。因此第一阶段的目标是把一个能运行、能测量、能复现的最小闭环跑通，再根据实测结果调整码率、频段、协议和硬件。

> [!abstract] 这篇笔记的主线
> 摄像头产生像素，ISP 整理图像，编码器压缩数据，RTP 携带媒体，Wi‑Fi 负责传输，RTSP 负责让播放器建立会话，VLC 或 OpenCV 最后完成解码和显示。飞控控制信息沿另一条 TTL UART 链路传输。

## 本页导航

[[#1. 先看完整链路|完整链路]] · [[#2. 摄像头、ISP 和编码器|摄像头与编码]] · [[#3. Wi‑Fi 只负责把两端接入同一网络|Wi‑Fi]] · [[#4. RTSP 如何把视频交给播放器|RTSP]] · [[#5. RTP、UDP 和 TCP 各自负责什么|传输协议]] · [[#7. 串口链路要和视频链路分开|UART]] · [[#8. 先跑通最小闭环|最小实验]] · [[#9. 根据日志定位故障|故障定位]]

## 1. 先看完整链路

摄像头输出原始像素，AMB82-Mini 的 ISP 对图像做曝光、色彩、降噪、裁剪和缩放等处理，硬件编码器再把连续图像压缩成 H.264、H.265 或 JPEG 码流。码流经过 RTP 封装后，由 Wi‑Fi 送到地面电脑，最后交给 VLC 或 OpenCV 接收、解码和显示。

~~~mermaid
flowchart LR
    CAM[摄像头<br/>原始像素] --> ISP[ISP<br/>曝光/色彩/降噪/缩放]
    ISP --> ENC[硬件编码器<br/>H.264/H.265/JPEG]
    ENC --> RTP[RTP 媒体包]
    RTP --> WIFI[Wi‑Fi<br/>STA 或 AP]
    WIFI --> PC[地面电脑]
    PC --> DEC[VLC/OpenCV<br/>接收/解码/显示]

    FC[飞控或机载控制器] <-->|TTL UART<br/>状态与控制帧| UART[AMB82-Mini 串口任务]
    UART --> ENC
~~~

图中还有一条控制链路。飞控或其他机载控制器通过 TTL UART 与 AMB82-Mini 交换状态、心跳和简短命令。视频链路数据量大，重点是持续传输和控制延迟；串口数据量小，重点是帧边界、校验和及时响应。两条链路不能共用一个会互相堵塞的任务和缓冲区。

这也是“Wi‑Fi 图传”容易被说混的地方：Wi‑Fi 只是无线连接方式，编码器负责压缩视频，RTSP 负责建立和控制会话，RTP 负责承载媒体数据，播放器负责解码和显示。

> [!important] 先分清两条链路
> 视频面追求持续传输和可接受的实时性，控制面追求帧边界、校验和及时响应。调试时不要用“视频能播放”代替“串口协议已经可靠”，也不要让视频任务独占串口任务的缓冲区。

## 2. 摄像头、ISP 和编码器

摄像头传感器把光照转换为连续的像素数据。传感器型号会影响分辨率、输出格式、时钟、初始化寄存器和驱动配置，所以它不是一个“插上就都一样”的输入设备。没有正确初始化的摄像头，后面的编码器和 RTSP 服务即使启动，也可能没有有效视频帧。

在 AMB82-Mini 上，传感器驱动需要按照实际型号读取行、帧和时序，并匹配相应的 Camera Options。社区 WebRTC 实验曾出现“信令已经连接但没有视频包”的情况，最后发现 SDK 配置的传感器与板上实际的 JXF37 不一致。这个例子说明，网络连接正常但没有画面时，应先看摄像头初始化日志，再检查 ISP 和编码参数。

**ISP（Image Signal Processor，图像信号处理器**位于传感器和编码器之间。它把原始像素整理成适合后续处理的图像帧，常见工作包括曝光、白平衡、色彩、降噪、锐化、裁剪和缩放。第一轮实验最好先使用官方 StreamRTSP -> VideoOnly 示例，不要同时修改 ISP 参数；先确认摄像头能稳定出图，再单独改变分辨率或裁剪设置，这样才知道问题来自硬件初始化还是参数调整。

**编码器**的任务是减少需要通过无线链路搬运的数据量。JPEG/MJPEG 通常把每张图像独立压缩，调试直观，但连续视频的带宽较大；H.264 利用帧内和帧间的相似内容压缩，兼容性较好，适合作为第一版实时图传；H.265 的压缩效率可能更高，但解码兼容性和调试复杂度也更高。

例如，同一摄像头从 720p/30fps 改为 1080p/30fps，单帧包含的细节会增加；提高帧率会增加单位时间需要编码和传输的帧数；提高码率通常能减少压缩损失，却会让无线链路更容易拥塞；关键帧间隔过长时，丢包后的画面恢复可能变慢。因此测试记录应写出分辨率、帧率、编码格式、GOP 和码率，不能只写“画面清晰”或“延迟较低”。

> [!tip] JPEG、MJPEG、H.264 和视频参数
> - **JPEG**：面向单张图片的压缩格式。一张图压缩一次，适合抓拍或逐帧调试。
> - **MJPEG**：Motion JPEG，即连续的 JPEG 图片。每一帧都独立编码，解码和定位问题直观，但相同画质下通常比 H.264 占用更多码率。
> - **H.264**：视频编码格式。除了帧内压缩，还会利用相邻帧的相似内容，只保存变化部分，通常更省带宽；代价是编码、解码和丢包恢复更复杂。
> - **分辨率**：一帧图像的像素数量，例如 1280×720。分辨率越高，单帧细节越多，待压缩的数据量通常越大。
> - **帧数 / 帧率（fps）**：单位时间内传输的画面帧数。30 fps 表示每秒 30 帧；帧率越高，运动越连贯，但每秒需要编码和传输的帧也越多。
> - **码率（bitrate）**：单位时间输出的视频数据量，常用 bit/s 或 Mbit/s 表示。码率太低会出现马赛克和细节丢失，太高则可能挤占无线链路、造成卡顿。
> - **GOP（Group of Pictures）**：关键帧（I 帧）与后续预测帧组成的一组画面。比如 30 fps、GOP=30，大约每 1 秒出现一个关键帧；GOP 越长通常越省码率，但丢包后要等更久才能恢复完整画面。
>
> **它们的关系可以这样看：** 分辨率决定每帧有多少像素，帧率决定每秒有多少帧；编码格式和 GOP 决定这些帧如何利用重复信息进行压缩；最终形成的码率决定每秒需要通过 Wi‑Fi 搬运多少数据。因此，提高分辨率或帧率通常会推高码率，改用更高效的编码格式可以压低码率，而码率必须落在无线链路和解码器能够稳定处理的范围内。

## 3. Wi‑Fi 只负责把两端接入同一网络

Wi‑Fi 是基于 IEEE 802.11 系列标准的无线局域网连接方式。在这个项目里，它为 RTSP 控制消息和 RTP 媒体数据提供传输路径，但它不规定视频怎样编码，也不负责建立 RTSP 会话。

第一轮实验建议使用 STA（Station）模式，让 AMB82-Mini 和 PC 都连接同一个路由器或手机热点。这样可以先减少地址配置、连接管理和 AP 隔离等变量。等视频链路稳定后，再测试 AP（Access Point）模式，让模块自己建立无线网络，PC 直接连接它。

2.4 GHz 通常覆盖更远、穿透障碍能力较好，但非重叠信道较少，也更容易受到蓝牙、微波炉和其他设备干扰。5 GHz 通常有更高吞吐量和更多信道，但覆盖距离和穿透能力往往较弱。测试距离时应同时记录模式、频段、信道、距离、遮挡、天线方向、RSSI、码率和卡顿次数，否则“10 米能用”无法判断究竟是距离、信道还是码率造成的。

Wi‑Fi 信号强也不等于视频一定稳定。播放器缓存、编码参数、丢包恢复方式和地面端解码能力都会影响最终画面。

## 4. RTSP 如何把视频交给播放器

RTSP（Real-Time Streaming Protocol，实时流控制协议）是应用层的实时音视频会话控制协议。它负责告诉播放器有哪些媒体流、怎样协商编码和端口、何时开始播放以及何时结束会话。

播放器打开一个 RTSP 地址时，通常先向 AMB82-Mini 请求媒体描述（DESCRIBE），获得包含编码和媒体信息的 SDP；随后通过 SETUP 协商传输方式和端口；发送 PLAY 后，板卡才开始持续发送媒体。播放结束时，播放器用 TEARDOWN 释放会话。RTSP 负责“把会话建立起来并控制它”，真正携带视频的通常是 RTP。

SETUP 可以理解为“给这条媒体流安排送货地址”。播放器会针对 SDP 中列出的每条媒体轨道发送 SETUP 请求，在 `Transport` 字段中提出使用 UDP 还是 TCP，以及接收 RTP/RTCP 的端口（使用 TCP 时也可以协商把 RTP 嵌入 RTSP 连接）。模块确认可用后返回会话编号和实际发送端口；之后的 PLAY 会沿用这组参数，视频 RTP 包才会持续发往对应端口。也就是说，SETUP 负责确定“怎么传、传到哪里”，不负责开始播放，也不改变视频编码格式。



~~~mermaid
sequenceDiagram
    participant PC as 地面端播放器
    participant CAM as AMB82-Mini
    participant NET as Wi‑Fi 网络

    PC->>CAM: DESCRIBE
    CAM-->>PC: SDP：编码、媒体和端口信息
    PC->>CAM: SETUP
    CAM-->>PC: 会话参数
    PC->>CAM: PLAY
    loop 持续播放
        CAM->>NET: H.264 RTP 媒体包
        NET-->>PC: RTP 包、序号、时间戳
        PC->>PC: 缓冲、解码、显示
    end
    PC->>CAM: TEARDOWN
~~~

具体实验可以直接使用官方 Arduino 示例。打开 File -> Examples -> AmebaMultimedia -> StreamRTSP，选择 VideoOnly，填入 Wi‑Fi 名称和密码并上传。串口监视器会给出板卡 IP、RTSP 端口和摄像头初始化结果。假设日志显示：

~~~text
IP: 192.168.1.36
RTSP port: 554
~~~

那么 PC 与开发板处于同一网络时，可以在 VLC 中打开：

~~~text
rtsp://192.168.1.36:554
~~~

> [!example]- 一次可复现的 RTSP 连接
> 板卡串口输出 IP 为 192.168.1.36、端口为 554，PC 与板卡连接同一网络后，在 VLC 中打开 rtsp://192.168.1.36:554。如果 VLC 建立连接但没有画面，先回到摄像头初始化、ISP 和编码器日志；如果画面出现后卡顿，再检查 RSSI、丢包、码率和播放器缓存。

如果日志没有 IP，应先检查 SSID、密码、频段和连接状态；有 IP 但端口无法打开，应检查网段、防火墙和 AP 隔离；VLC 能连接但没有画面，应回到摄像头、ISP 和编码器日志；有画面但持续卡顿，则记录 RSSI、丢包、码率和播放器缓存后再调参数。

官方示例通常使用 554 端口，双路流示例可能使用 554/555，但实际端口必须以串口日志为准。RTSP 也不能单独保证低延迟、稳定画面或远距离通信，最终表现还取决于编码、RTP、Wi‑Fi 和播放器。

## 5. RTP、UDP 和 TCP 各自负责什么

RTP（Real-time Transport Protocol，实时传输协议）用于承载实时音视频媒体数据。它通常为每个媒体包附加序号、时间戳和载荷类型，使接收端知道数据属于哪一路媒体、是否出现丢包或乱序，以及应该按什么节奏播放。

假设连续收到序号为 1001、1002、1004 的 RTP 包，接收端可以判断 1003 可能丢失。实时播放器通常会继续播放后续画面，而不是无限等待丢失的数据；具体表现取决于编码结构和播放器的错误恢复能力。RTP 提供的是媒体传输所需的顺序和时间信息

UDP 是 **User Datagram Protocol（用户数据报协议）**：U 是 User（用户），D 是 Datagram（数据报），P 是 Protocol（协议）。TCP 是 **Transmission Control Protocol（传输控制协议）**：T 是 Transmission（传输），C 是 Control（控制），P 是 Protocol（协议）。两者都位于传输层，但工作方式不同。UDP 通常不确认、不等待重传，应用层需要自己面对丢包和乱序，因此延迟较低、适合实时视频；TCP 会确认、按序交付并自动重传，数据更完整，但丢包时可能等待重传而产生队头阻塞和延迟。实时图传通常优先 UDP 保持画面及时，网络限制或需要穿过只允许单连接的环境时，也可以使用 TCP 承载 RTP。

因此，实时图传中常让 RTP 媒体使用 UDP，使少量丢包不会让画面一直等待；RTSP 控制连接通常使用 TCP，使会话建立、参数协商和结束请求可靠到达。在某些网络环境中，RTP 也可以交错到 RTSP 的 TCP 连接中。所以不能简单地说“UDP 一定低延迟”或“TCP 一定不能传视频”，实际结果还取决于丢包率、缓存、编码结构和网络配置。

## 6. AMB82-Mini 机载计算节点

AMB82-Mini 是 Realtek AmebaPro2 系列开发板，核心 SoC 为 RTL8735B。它集成 Wi‑Fi、BLE、摄像头接口和音视频编解码能力，并提供 UART、SPI、I2C、PWM、ADC 和 GPIO 等接口。在本项目中，它不是一个只做串口到 Wi‑Fi 转换的透明网卡，而是同时承担摄像头采集、图像处理、视频编码、无线连接、RTSP/RTP 输出和串口任务的机载节点。

官方入门资料列出的常用串口引脚包括 Serial1 的 PA2/PA3、Serial2 的 PD16/PD15、Serial3 的 PE2/PE1，日志口为 PF4/PF3。实际接入飞控前，应以手中板卡版本、原理图和丝印为准，避开下载口和日志口，否则可能出现程序无法下载或调试日志干扰飞控数据。

供电也属于系统的一部分。Wi‑Fi 发射、摄像头启动和编码器同时工作时会产生瞬时负载，无人机电源还可能受到电机和电调噪声影响。接入飞行器前应确认输入电压、稳压方式、峰值电流、地线、天线和摄像头供电。

## 7. 串口链路

TTL UART 是使用逻辑电平进行点对点串行通信的接口，通常使用 TX、RX 和 GND。它适合传输工作模式、启动状态、信号质量、时间戳、抓图命令、心跳和简短控制帧，不适合传输完整视频。

TX 与 RX 交叉连接，并共用 GND。双方还必须约定波特率、数据位、校验位和停止位。为了让接收端判断一条消息是否完整、是否损坏，可以先采用下面的帧格式：

~~~text
帧头(2B) | 版本(1B) | 类型(1B) | 序号(1B) | 长度(1B) | 载荷(NB) | CRC16(2B)
~~~

联调时先做 USB-TTL 回环：确认模块和转换器都是 3.3 V TTL，模块 TX 接转换器 RX，模块 RX 接转换器 TX，两者共用 GND，设置相同波特率，再发送带序号和 CRC 的测试帧。只有接收端能够正确识别帧头、长度和 CRC 后，才把同一帧格式接入飞控。

3.3 V TTL UART 与 RS‑232 的电压标准不同，不能直接混接。看到串口有数据也不等于协议正确，还要检查电平、波特率、帧边界、校验和任务调度。视频任务和串口任务应使用不同的缓冲队列，视频卡顿时，串口仍应能够接收停止命令或发送心跳。

> [!warning] 接线前先确认电平
> 3.3 V TTL UART、5 V 逻辑电平和 RS‑232 不是同一种接口。TX、RX 交叉连接只是接线关系，不能代替电平兼容性检查。

## 8. 先跑通最小闭环

第一阶段的目标是：单块 AMB82-Mini 连接摄像头，通过 Wi‑Fi 输出 RTSP 视频，PC 端能够持续播放，同时 TTL UART 能收发一条带 CRC 的控制帧。

建议按这个顺序做：

1. 确认板卡、摄像头、天线、USB 线和 3.3 V TTL 转 USB 模块；
2. 安装 Arduino IDE 和官方 AmebaPro2 开发板包；
3. 先上传 Blink，确认下载模式和串口日志正常；
4. 运行 StreamRTSP -> VideoOnly；
5. 让 AMB82-Mini 和 PC 连接同一个网络；
6. 从串口日志记录 IP、RTSP 端口和摄像头初始化结果；
7. 在 VLC 中打开 rtsp://<IP>:<port>；
8. 连续播放 10 分钟，记录分辨率、帧率、码率、卡顿和断线；
9. 单独做 UART TX/RX 回环；
10. 视频和串口都稳定后，再测试 AP 模式、不同频段和不同码率。

测试记录至少包括：

| 类别 | 记录内容 |
|---|---|
| 视频 | 分辨率、帧率、编码格式、GOP、码率 |
| 网络 | STA/AP、2.4/5 GHz、信道、距离、遮挡、RSSI |
| 实时性 | 首帧时间、平均延迟、最大延迟、卡顿次数 |
| 可靠性 | 连续播放时长、断线次数、重连时间、丢帧情况 |
| 串口 | 电平、波特率、帧格式、CRC、心跳周期 |
| 硬件 | 输入电压、电流、天线、摄像头型号、工作温度 |

“低延迟”需要测量。可以在摄像头前放一个毫秒显示器或周期闪烁的 LED，同时观察 VLC 窗口中的同一事件，记录拍摄时刻与显示时刻的差值。这个数值是摄像头到显示器的端到端估计，不等于单独的网络往返时间。

> [!tip] 建议的实验节奏
> 先固定网络模式、频段、分辨率和码率，只验证链路是否能持续播放；再一次只改变一个参数。这样得到的卡顿、延迟和重连结果才有比较意义。

## 9. 根据日志定位故障

~~~mermaid
flowchart TD
    A[无法播放] --> B{串口有摄像头初始化错误?}
    B -- 是 --> C[核对传感器型号、排线、供电和 Camera Options]
    B -- 否 --> D{能否看到 Wi‑Fi IP?}
    D -- 否 --> E[检查 SSID、密码、频段、天线和日志]
    D -- 是 --> F{PC 能否连通 IP/端口?}
    F -- 否 --> G[检查同网段、防火墙、端口和 AP 隔离]
    F -- 是 --> H{VLC 有画面但卡顿?}
    H -- 是 --> I[降低码率/分辨率，检查信道、RSSI 和播放器缓存]
    H -- 否 --> J[记录当前参数，开始 UART 与距离测试]
~~~

排查时先判断问题属于哪一层：供电、摄像头和 ISP 属于硬件输入；IP、端口、网段和防火墙属于网络可达性；分辨率、帧率、码率、GOP、丢包和缓存属于媒体参数。不要同时更换摄像头、SDK、协议和网络模式，否则无法判断哪一项改变产生了结果。

## 10. 后续路线

第一阶段使用官方 AmebaPro2 Arduino SDK，先完成 Blink、串口和 StreamRTSP -> VideoOnly 的最小闭环。只有当项目需要精确控制任务、缓冲、码率、断线重连或自定义协议时，再转到 FreeRTOS/MMF。

ameba_stream_project 可以用来参考 Python/OpenCV 地面端、多板管理和按帧保存；AMB82-MINI-Vision-Hub 可以用来参考 AP 模式和视频节点与控制节点的分层；本地 WebRTC 实验可以用来了解更低延迟的方向，但它需要信令、ICE、DTLS 和 FreeRTOS SDK 修改，不适合作为第一版交付路线。

项目组还需要明确第一版采用 720p 还是 1080p、允许的端到端延迟、地面端是否只用 VLC、飞控 UART 的电平和协议、模块接入哪一路电源，以及丢包时优先保持实时性还是优先保证画面完整。

第一阶段的验收标准是：**单块 AMB82-Mini 能稳定采集摄像头画面，通过 Wi‑Fi 输出 RTSP，PC 端持续播放，同时 TTL UART 能收发带 CRC 的控制帧，所有参数和测试条件有记录。**

> [!success] 第一阶段交付判断
> 能播放只是视频面打通的证据。只有当视频连续播放、测试条件有记录、UART 控制帧能够校验并且两条任务互不阻塞时，才算完成最小闭环。

## 资料来源

### 官方硬件与 SDK

- [AMB82-Mini Getting Started](https://github.com/Ameba-AIoT/ameba-arduino-doc/blob/main/source/ameba_pro2/amb82-mini/Getting_Started/Getting%20Started%20with%20Ameba.rst)
- [AmebaPro2 Arduino SDK](https://github.com/Ameba-AIoT/ameba-arduino-pro2)
- [官方 RTSP Streaming 教程](https://www.amebaiot.com.cn/cn/amebapro2-arduino-video-rtsp/)
- [AmebaPro2 FreeRTOS SDK](https://github.com/Ameba-AIoT/ameba-rtos-pro2)
- [AmebaPro2 MMF 文档](https://github.com/Ameba-AIoT/ameba-rtos-pro2-doc/blob/master/source/application_note/06_MMF.rst)
- [AmebaPro2 硬件设计指南](https://ameba-doc-rtos-pro2-sdk.readthedocs-hosted.com/en/latest/hw_design/)
- [Camera Module Bring Up Guide](https://www.amebaiot.com.cn/en/camera-module-bring-up-guide/)

### 协议与标准

- [IETF RFC 7826：RTSP 2.0](https://www.rfc-editor.org/rfc/rfc7826.html)
- [IETF RFC 3550：RTP](https://datatracker.ietf.org/doc/rfc3550/)
- [ITU-T H.264 Recommendation](https://www.itu.int/ITU-T/recommendations/rec.aspx?rec=H.264)

### 无线与串口背景

- [Microsoft：Wi‑Fi 频段与家庭网络布局](https://support.microsoft.com/en-us/windows/experience/connectivity-networking/wi-fi-and-your-home-layout)
- [Ubuntu：UART 控制台与 3.3 V TTL 注意事项](https://ubuntu.com/hardware/docs/boards/how-to/special_hardware/uart-console/)
- [DJI Onboard SDK：UART 硬件设置](https://developer.dji.com/onboard-sdk/documentation/development-workflow/hardware-setup.html)

### 工程参考

- [ameba_stream_project](https://github.com/Dennis40816/ameba_stream_project)
- [AMB82-MINI-Vision-Hub](https://github.com/NNavoda/AMB82-MINI-Vision-Hub)
- [AMB82 Mini 本地 WebRTC 实验](https://forum.amebaiot.com/t/working-poc-amb82-mini-local-webrtc-h-264-streaming-to-firefox-without-aws/5168)
