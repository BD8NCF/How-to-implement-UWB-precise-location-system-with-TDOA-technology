This is a series of articles that will introduce you to how to implement a UWB precise location system using TDOA technology.

**IMPORTANT NOTE:**

>**Q: Is it necessary to build this location system?**
>
> A: This article is not for beginners. It requires a basic knowledge of electronic technology and software programming.
> **Q: Are your hardware/software open source?**
> 
> A: It is not open source. In the article, I will introduce how to implement the UWB location system and tell you how to overcome the difficulties, but I will not directly give you the Gerber file of the PCB to make a board, nor will I give you the source code of the software, nor will I give you the compiled firmware. I'm not going to give you any direct results, I'm just telling you how.
> 
> **Q: I am personally very interested in UWB location. Can I make a location system?**
>
> A: If you have a strong hardware/software background and a lot of time, you can certainly do it. This article is just for you to read!
> 
> **Q: I am a commercial company and I want to develop the UWB location system into a commercial product.**
> 
> A: Of course. This article is also written for you. If you want to build the entire system from zero, after reading my article, you only need to draw the circuit and make the board; conceive the software structure and then coding. In this way, I will mention all the difficulties in the article and introduce the solutions. You don't need to hire people to do algorithm research. If you want to save trouble and time, you can directly purchase our circuit diagram (AD engineering files), purchase our software source code, and then quickly enter the production process. (Website: https://uwbhome.top )

# Tag firmware design

In the previous article, I introduced the hardware design of the location anchor and tags, and the firmware design of the anchor (including the algorithm principle of clock synchronization). According to the normal roadmap, the next step should be the firmware design of the tag. But the tag's firmware is really simple and there isn't much content to write. After thinking about it, I decided to write an article for it.

The most important function of the tag is to regularly send TDOA location data packets, and other functions are additional. This function is too simple to implement. Many of the sample codes provided by decawave can send uwb data packets, but the format of the data packets needs to comply with our definition.

The format of the UWB TDOA location data packet we define is as follows:

```C
// Tag测距帧
// 广播帧, Tag 发出的测距消息, 各个 Anchor 根据收到这个帧的时间差来应用 TDOA 测距
typedef struct ieee154_broadcast_tag_range_frame {
	uint8_t frameCtrl[2];			//  0, 2: frame control bytes 00-01: 0x01 (Frame Type 0x01=date), 0xC8 (0xC0=src extended address 64 bits, 0x08=dest address 16 bits)
	uint8_t seq8;				//  2, 1: sequence_number 02
	uint8_t panID[2];			//  3, 2: PAN ID 03-04
	uint8_t destAddr[2];			//  5, 2: 0xFFFF
	uint8_t sourceAddr[8];			//  7, 8: 64Bits EUI地址
	uint8_t	messageType;			// 15, 1: Message Type = RTLS_MSG_TYPE_TAG_RANGE
	uint8_t	seq64[8];			// 16, 8: Tag 发出的测距消息序号, 比 seq8 有更在的最大值
	uint8_t powerVoltage[2];		// 24, 2: 电源电压*1000
	uint8_t batteryVoltage[2];		// 26, 2: 电池电压*1000
	uint8_t lighteness[2];			// 28, 2: 亮度, 直接 ADC 转换过来的值
	uint8_t switchStatus;			// 30, 1: 开关状态
	int16_t imu_aacx;			// 31, 2:
	int16_t imu_aacy;			// 33, 2:
	int16_t imu_aacz;			// 35, 2:
	int32_t imu_roll;			// 37, 4:
	int32_t imu_pitch;			// 41, 4:
	int32_t imu_yaw;			// 45, 4:
	int16_t imu_temp;			// 49, 2:
	uint8_t fcs[2];				// 51, 2
} BROADCAST_TAG_RANGE_MESSAGE;
```

Later we found that this data packet was too big and there were many fields that were unnecessary, so we changed it to a smaller format:

```C
// Tag测距帧, 最小测距帧
// 广播帧, Tag 发出的测距消息, 各个 Anchor 根据收到这个帧的时间差来应用 TDOA 测距
typedef struct ieee154_broadcast_tag_range_min_frame_v2 {
	uint8_t frameCtrl[2];			//  0, 2: frame control bytes 00-01: 0x01 (Frame Type 0x01=date), 0xC8 (0xC0=src extended address 64 bits, 0x08=dest address 16 bits)
	uint8_t seq8;				//  2, 1: sequence_number 02
	uint8_t panID[2];			//  3, 2: PAN ID 03-04
	uint8_t destAddr[2];			//  5, 2: 0xFFFF
	uint8_t sourceAddr[8];			//  7, 8: 64Bits EUI地址
	uint8_t	messageType;			// 15, 1: Message Type = RTLS_MSG_TYPE_TAG_MIN_RANGE_V2
	uint8_t	seq32_3[3];			// 16, 3: Tag 发出的测距消息序号的高3字节, 与 seq8 组合为 seq32, 比 seq8 有更在的最大值
	uint8_t switchStatus;			// 19, 1: 开关状态
	uint8_t fcs[2];				// 20, we allow space for the CRC as it is logically part of the message. However ScenSor TX calculates and adds these bytes.
} BROADCAST_TAG_RANGE_MIN_MESSAGE_V2;		// 以上合计 22 字节
```

Reduced from 53 bytes to 22 bytes. In this way, the time a location packet occupies the frequency during transmission is reduced by 42%. This means more tags can be accommodated.

You should notice that the minimized data packet does not even contain voltage data. We also define a location data packet containing voltage data, which is sent regularly at intervals, because the battery voltage does not change rapidly. A packet of voltage information will do.

A customer customized a tag with a heart rate sensor, and we also added a location data package containing heart rate and blood oxygen data.

Unlike the anchor that can communicate through the network, the tag can only communicate via USB. The USB interface uses HID mode and can be driver-free. Do not use an analog COM port, because if you use COM communication, on the one hand, you need to install a driver, on the other hand, COM conflicts may occur, which will cause more trouble. Using the HID method is much simpler.

USB communication requires the use of an external crystal oscillator, and the built-in RC oscillator of STM32 cannot be used. Because the frequency of the RC oscillator is not very accurate, if you use an RC oscillator, it may cause compatibility issues. On some computers, communication may fail due to inaccurate frequencies.

The power saving mode of STM32 uses STOP mode, which is relatively moderate.

Pay attention to the clock frequency of STM32. Under normal circumstances, the clock of STM32 is 72MHz. Because we do not need such high computing power, we can use 48MHz. A lower frequency can save power. Other lower frequencies will not work, because we also need to provide a 12MHz frequency for the USB part, and lower frequency combinations cannot produce 12MHz.

Also, such a simple program does not need to run an OS, just run naked. There are two reasons for not running the OS:

1. The structure of RTOS is complex, and we also need to enter the STOP state, which makes the program logic much more complicated.
2. The RAM/Flash of the STM32F103CBT6 chip is relatively small and is not suitable for large programs.

If you have any ideas, please leave me a message or comment.

*That's it for this article, except that the source code is not given directly. The next article will introduce the TDOA algorithm and location engine.*

*By the way, I'm looking for a job. Due to business reasons, the company was dissolved. I have 30+ years of work experience, 20+ years of experience in C/C++/Java/Delphi, and a little bit of Javascript/Python/Lua. In 199x, I wrote a Chinese character system in x86 assembly, and later used Delphi to write a mail server. I have written countless application systems, and the code I have written should exceed 2 million lines. 10+ years of hardware design experience, designing multiple embedded products. In the early stages of this UWB location system, the hardware and software were all developed by me alone, and the team was only built later. Base Guiyang, China, or remotely. If there are job opportunities, please contact me for a detailed resume.*

 

