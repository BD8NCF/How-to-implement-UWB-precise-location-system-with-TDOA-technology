This is a series of articles that will introduce you to how to implement a UWB precise location system using TDOA technology.

**IMPORTANT NOTE:**

>**Q: Do I need basic knowledge to create this location system?**
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

# Anchor firmware design-clock synchronization
## Clock synchronization principle

We have said before that for TDOA technology, each anchor needs to have unified time, which is a difficulty. How to synchronize the clock?

Assume that there are four anchors A/B/C/D in a location area. We also use another anchor CS as the clock source to send clock synchronization packets regularly. The structure of this clock synchronization package is as follows:

```C
typedef struct ieee154_broadcast_clock_sync_frame {
    uint8_t frameCtrl[2];           // 0, 2, frame control bytes 00-01: 0x01 (Frame Type 0x01=date), 0xC8 (0xC0=src extended address 64 bits, 0x08=dest address 16 bits)
    uint8_t seq8;                   // 2, 1, sequence_number 02
    uint8_t panID[2];               // 3, 2, PAN ID 03-04
    uint8_t destAddr[2];            // 5, 2, 0xFFFF
    uint8_t sourceAddr[8];          // 7, 8, 64Bits EUI地址
    uint8_t messageType;            //15, 1, Message Type = RTLS_MSG_TYPE_CLOCK_SYNC
    uint8_t seq64[8];               //16, 8, 发出的测距消息序号, 比 seq8 有更大的最大值
    uint8_t timeStamp[8];           //24, 8, 时间戳
    uint8_t fcs[2];                 //32, 2
} BROADCAST_CLOCK_SYNC_MESSAGE;
```

This is a typical UWB packet for DW1000. This is a broadcast type packet, which has several important fields. timeStamp is the timestamp, which is the time when anchor A sends this packet.

When the four anchors A/B/C/D receive this synchronization packet, they record the timestamp and the time when the packet was received (local timestamp). In this way, we have two timestamps.

After a while, CS sends another synchronization packet. After a anchor, such as anchor B, receives it, it records the remote timestamp of the CS anchor and the time stamp received locally, and obtains two more timestamps.

Assume that the interval between the two packets sent by the CS is 100ms, that is, the time interval between the two synchronous transmissions is 100ms. So under normal circumstances, when anchor B receives these two packets, the local time interval should also be 100ms. Actually not!

Because the crystal oscillators used by the DWM1000 of the two anchors will not have exactly the same frequency due to various reasons, there will always be some differences.

What is the clock difference between the two anchors?

deta = (ST2 - ST1) - (RT2 - RT1). where deta is the difference, ST2 is the sending time of the second packet, ST1 is the sending time of the first packet, RT2 is the receiving time of the second packet, and RT is the receiving time of the first packet.

The difference rate is Ratio = deta / (ST2-ST1)

In this way, the start time is known, and the clock discrepancy rate is also known. We can convert any local timestamp to a timestamp based on CS.

When a tag sends a UWB location data packet, all four anchors A/B/C/D receive the packet. A/B/C/D convert the local timestamp upon receipt to the CS time as the basis. Everyone sends this timestamp to the location engine. The location engine gets 4 times.
Based on their difference, the coordinates of the tag can be calculated.

Seeing this, you should have a question in your mind. The sending time of the synchronization data packet sent by the CS is the moment when the UWB data packet leaves a gate of the DW1000 chip in the CS. The UWB signal then passes through the internal circuit of the chip, and then is transmitted into the air through the transmitting antenna, reaches the antenna of anchor B, is transmitted inside the antenna of anchor B, passes through some internal circuits of the chip, and reaches the receiving gate of DW1000 in anchor B.
At that moment is the moment of reception. This process takes a lot of time. How do you get this time? It is introduced in the manual of the DW1000 chip that we need to consider the transmitting antenna delay/receiving antenna delay.

The answer is, leave it alone.

We assume that the DW1000 of anchor B is connected to the antenna through a long radio frequency cable. After the clock synchronization packet reaches the antenna of anchor B, it passes through a long cable before reaching the DW1000 chip. Of course, the same goes for the location data packets sent by the tags. After arriving at the antenna of the B anchor, they then go through a long cable before reaching the DW1000 chip. Assume that a section of the cable is added, causing the radio wave to run 1ms longer in the cable. Of course, the clock synchronization packet and the location packet both spend 1ms longer on the road. If we take this 1ms into account when calculating the timestamp when anchor B receives the clock synchronization packet, then when the location engine calculates the time difference, the extra 1ms spent on the tag's location packet must be deducted. So the time these two packets spend in the cable cancels out in the calculation formula.

When installing a anchor, we care about "where the anchor is installed." In fact, we don't really care about the location of the anchor, but the location of the anchor antenna. The location of the antenna is what matters. Clock synchronization packets arrive at the antenna, location data packets arrive at the antenna, it doesn't matter where the anchor itself is.

This is also one of the advantages of TDOA. There is no need to worry about antenna delay. There is no need to do antenna delay calibration before leaving the factory like TOF anchors and TOF tags.

The above analysis is in the scenario of using a separate anchor as the clock source. Our original system used a separate clock source. The hardware and firmware of the clock source are exactly the same as those of a normal anchor, depending on whether it is configured as a clock source or a normal anchor. When used as a clock source, it has only one function, which is to send clock synchronization packets regularly.
The DW1000 of the clock source only transmits but does not receive; when used as an ordinary anchor, it receives the clock synchronization packets sent by the clock source and the location packets sent by the tags.
The DW1000 of the ordinary anchor DW1000 only receives but does not transmit. Later, in order to reduce costs, we merged the two functions. An ordinary anchor can be used as both a anchor and a clock source.
It is usually in the receiving state, periodically switches to the transmitting state to send a clock synchronization packet, and then immediately switches back to the receiving state. This way, a separate clock source can be saved.

If the anchor serves as the clock source at the same time, then the antenna delay of the anchor as the clock source will have an impact on the calculation. For clock synchronization packets, it is the transmit delay; for location data packets, it is the receive delay. The difference between the internal transmit delay and receive delay of DW1000 is not big and can be ignored in calculation. However, if there is an external PA/LNA and the transmission and reception routes are quite different, this anomaly needs to be taken into consideration.

When we originally designed the entire system, we considered using a unified time for the entire system. A large location area can be divided into multiple small location areas. We can deploy a root clock, and each area has a minute clock. Synchronize the time from the root clock and synchronize the clocks step by step.

However, things are not that simple. The main problem is error. We cannot rule out errors between clocks.

For example, the time of root clock R is synchronized to A1/B1, then A1 is synchronized to A2, and B1 is synchronized to B2. Then we will find that the error between A2 and B2 will be relatively large. Because clock synchronization itself has errors, multi-level synchronization will lead to error accumulation, and the final error will become larger and larger.

Finally, we found that unifying the time of the entire system is of little significance. We only need to unify the time among several anchors in a small location area.

Careful students must have noticed that in the clock synchronization packet structure defined earlier, there is a field seq64. This is a 64-bit integer. The DW1000 chip internally defines an 8-bit seq, which is placed after the frame control field. We found that 1 byte can only express a packet sequence number of 256 at most, and it will wrap up quickly. Sometimes, when we care about packets in a larger time scale, we cannot tell the difference. Therefore, we set a separate 64-bit packet sequence number.

In addition, the effective data of the DW1000 timestamp is 40 bits, and its unit is approximately 15.6ps.

Later, in consideration of excellence, we newly defined a more concise clock synchronization package.

```C
typedef struct ieee154_broadcast_mini_clock_sync_frame {
	uint8_t frameCtrl[2];			// 0, 2: frame control bytes 00-01: 0x01 (Frame Type 0x01=date), 0xC8 (0xC0=src extended address 64 bits, 0x08=dest address 16 bits)
	uint8_t seq8;				// 2, 1: sequence_number 02
	uint8_t panID[2];			// 3, 2: PAN ID 03-04
	uint8_t destAddr[2];			// 5, 2: 0xFFFF
	uint8_t sourceAddr[8];			// 7, 8: 64 Bits EUI地址
	uint8_t	messageType;			// 15,1: Message Type = RTLS_MSG_TYPE_MINI_CLOCK_SYNC
	uint8_t	seq32_3[3];			// 16,3: 发出的测距消息序号的高 24 位,即高 3 字节
	uint8_t timeStamp[5];			// 19,5: 时间戳, 40 Bits
	uint8_t fcs[2];				// 24,2:
} BROADCAST_MINI_CLOCK_SYNC_MESSAGE;		// 以上合计 26 字节
```

There is no need to make seq 64 bits, 32 bits is enough. If there is a seq8 in the front, then only 3 bytes are needed in the back, so change seq64 to seq32_3; the effective bits of the timestamp are only 40 bits, so there is no need to use it. 64 bits.

The new clock synchronization packet size is reduced from 34 bytes to 26 bytes.

Because wireless signals are essentially broadcasts, there can only be one sound in the air at the same time. If other sounds appear, it is interference. Therefore, we must reduce channel occupation as much as possible. The smaller the data packets transmitted, the shorter the occupied time. Time can be freed up to accommodate more tags; and the shorter the time a data packet occupies the channel, the smaller the chance of being interfered with.

# Anchor sharing - clock synchronization with multiple clock sources

Because the definition of UWB business is near-field wireless communication, radio regulatory authorities in almost all countries have strict restrictions on UWB signal power and do not allow high-power transmission. Because the bandwidth occupied by UWB is too wide, if the power is high, it becomes a source of interference.

When the original DW1000 transmits at maximum power and communicates at a rate of 850K, the coverage range is approximately 200 meters to 300 meters. In fact, this maximum power is beyond the standard. If the signal strength required by the Radio Administration is followed, the communication distance will be within 20 meters.

In any case, when we want to locate a large area, we often need to divide it into multiple small location areas. We assume that there is an area of ​​100 meters x 100 meters divided into 4 areas of 25 meters x 25 meters. Assuming that each area uses 4 anchors, then 4 areas require 4x4=16 anchors.

If the anchors can be shared, and the anchors at the edge of the location area can be shared by adjacent areas, then we only need to install one anchor at each vertex of the word "field", which only requires 3*3=9 anchors.

Using as few anchors as possible can reduce user investment and reduce project construction difficulty.

For the clock synchronization we mentioned earlier, the anchor can record the timestamps of several more clock sources to achieve the effect of synchronizing time with multiple clock sources.

```C
typedef struct tag_ClockSource_Sync_Info {
	uint8_t clockSourceId[8];
	double factor;
	double lastFactors[CLOCKSOURCE_SYNC_INFO_LAST_FACTOR_NUM];
	int64_t offset;
	int64_t localStartTime;
	int64_t lastSyncTime;
	SAMPLE_KALMAN_FILTER_PARAMETER kfp;
	SAMPLE_RC_FILTER_PARAMETER rfp;
	double ppm;
	double lastFactorAcc[CLOCKSOURCE_SYNC_INFO_LAST_FACTOR_NUM];
	double factorAcc;
} CLOCKSOURCE_SYNC_INFO;
```

The above structure records the clock synchronization data of a certain clock source.

```C
#define CLOCKSOURCE_SYNC_INFO_NUM	15			// 记录 15 个时钟源的同步信息
CLOCKSOURCE_SYNC_INFO clockSourceSyncInfo[CLOCKSOURCE_SYNC_INFO_NUM];
```

The above array records data from multiple clock sources.

The specific number of clock sources allowed to be synchronized is related to the RAM of the MCU. In reality, there are actually not too many areas sharing the same anchor, and 15 basically have different functions.

# Interference factors for clock synchronization

If you calculate the discrepancy rate of the clock every time you receive a clock synchronization packet, you will find that it keeps changing. We expect to get a stable, or at least a little varying, difference rate over a period of time. But this is impossible! There are so many “distraction” factors.

## Crystal oscillator frequency changes

The crystal oscillator of DW1000 is the source of the clock. There are many factors that affect the frequency production of crystal oscillators. Temperature and humidity changes/voltage changes will affect the frequency of crystal oscillators. Changes in the capacitance of the pad capacitors at both ends of the crystal oscillator will also have an impact on the frequency (some circuits adjust the frequency of the circuit by adjusting the capacitance of the two pad capacitors).
Changes in temperature, humidity and voltage will also have an impact on the two capacitors.

The Datasheet of DW1000 recommends "Crystal (38.4 MHz +/-10ppm)" and "TCXO (optional use in Anchor nodes. 38.4 MHz)". The DWM1000 module encapsulates the DW1000 chip in an iron box, which is beneficial to the stability of the crystal oscillator frequency, because there is a certain insulation effect in the iron box, which prevents the temperature of the crystal oscillator from being quickly affected by air flow.

In fact, inaccurate frequency is not terrible. No matter how large the frequency deviation is, we can calibrate it through calculation. What we fear is irregular and rapid change.

## Changes in radio wave transmission speed

Radio waves are transmitted in the air, and air, as a medium, is unstable. The temperature/humidity/air pressure of the air will affect the transmission speed of radio waves in the air.

Once, I mounted the development board on a tripod next to my desk and observed changes in the clock synchronization discrepancy rate. A colleague happened to be passing by, and I immediately saw a drastic change in the curve. This is why I emphasized the importance of the shell in the previous anchor circuit design section.

# Keep clock synchronization stability

Because there are many factors that affect clock synchronization differences, we must find ways to maintain stability. There are several ways:

Shorten the clock synchronization period. We set the clock synchronization period to 250ms. A shorter clock synchronization period can keep the current clock synchronization data updated and match the latest various influencing factors.

Kalman filter or other filtering methods. Generally speaking, changes in frequency are random points similar to a normal distribution: frequency has an overall change trend.
If the difference rate of our clocks is drawn, it is generally a gradually changing curve, but each time it is calculated The point does not fall exactly on the curve, but is near the curve. We can use filters to get the values ​​on the curve.

The effect of using filtering is very limited. The most important thing is to shorten the clock synchronization period. Because the value obtained through the filter is not the actual situation, but an ideal value. For example, because changes in air temperature cause the transmission speed of radio waves from the clock to the anchor to change, we arrive at a value that excludes this effect, but in fact this value is of little significance. Because the speed of location packets sent by tags will also be affected, the influence of air changes should be eliminated together.
If only clock synchronization is excluded, but the location packets sent by tags are not excluded, then the calculation results will definitely have a large deviation.

*I originally planned to write one article on anchor firmware design, but I found that just clock synchronization alone requires so much content. It seems that there is still a lot to write. Write slowly.*

*By the way, I'm looking for a job. Due to business reasons, the company was dissolved. I have 30+ years of work experience, 20+ years of experience in C/C++/Java/Delphi, and a little bit of Javascript/Python/Lua. In 199x, I wrote a Chinese character system in x86 assembly, and later used Delphi to write a mail server. I have written countless application systems, and the code I have written should exceed 2 million lines. 10+ years of hardware design experience, designing multiple embedded products. In the early stages of this UWB location system, the hardware and software were all developed by me alone, and the team was only built later. Base Guiyang, China, or remotely. If there are job opportunities, please contact me for a detailed resume.*
