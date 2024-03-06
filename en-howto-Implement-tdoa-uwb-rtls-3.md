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


*The previous article introduced the method of clock synchronization, in order to get some practical information out as soon as possible, so that readers would not think that I only wrote some general talk without technical content. Write slowly next step.*

# Anchor firmware design-Flash layout

When writing firmware for an MCU, if you are a beginner, you may start writing applications directly as soon as you get started. Veterans will first consider the layout, what things are to be done, what is the relationship between them, etc., and must be considered thoroughly.

The Flash of the anchor is divided into 4 blocks: Bootloader, Application1, Application2, and Data.\

# Anchor firmware design-online upgrade

It is important that the product's firmware is field upgradeable. On-site upgrade means that the firmware can be upgraded directly at the customer's site through a local area network or the like without removing the installed product. You know, when a product has been installed at a customer site thousands of kilometers away, and you suddenly find a bug in the firmware, what should you do? If it can be upgraded on site, the code will be corrected in the office, new firmware will be generated, sent to customers on site, and the firmware will be upgraded on site. During this process, you don't need to leave your office at all.

If the firmware of your product cannot be upgraded on site, I'm sorry. After discovering the bug, you can only ask the customer to disassemble the product and send it back to the company. You can flash the new firmware and send it to the customer... It's a headache to think about it.

# Anchor firmware design-Bootloader

Bootloader is the startup code, Application1/Application2 are the applications, and Data holds the configuration data.

When powering on, the program in the Bootloader is executed first. It determines which Application1/Application2 is valid and jumps to the valid Application for execution. There are two variables in the Data area: Application1Valid and Application2Valid. If True, the corresponding Application is valid.

In the production stage, the worker responsible for production is only responsible for flashing the Bootloader. At this time, Application1Valid and Application2Valid are both False.

After powering on, if a valid Application cannot be found, the Bootloader will start the network part of the code: DHCP Client/TCP Client/DNS Client/UDP Server/UDP Client, etc. If it is connected to the network, it will always send a UDP packet to the network to request initialization.

After the factory initialization program receives the UDP packet requesting initialization, it will register the basic information of the new anchor, such as MCU ID, etc., and then assign it an EUI64 address. This address will later be the ID of the anchor. A serial number is also assigned. Then the default configuration data must be written. The factory initialization process then begins sending firmware to the new anchor. After the bootloader receives the firmware, it writes it to an invalid Application area (at the beginning, the Application1 area must be invalid). The firmware will be sent in many packages. After receiving the firmware, the Bootloader will set Applicaiton1Valid to True and then restart.

After powering on again, the Bootloader finds that Application1 is valid and jumps to Application1 for execution.

This completes the startup of the new firmware.

In fact, the network-related code and firmware receiving code in the Bootloader are only executed once during factory initialization and will not be executed again in the future. Because after leaving the factory, one of Application1 and Application2 is always valid.

If the bootloader finds that both areas are valid, it will invalidate the second area. Only one area is allowed to be valid, and the invalid area is reserved for the new upgraded firmware.

# Anchor firmware design-firmware upgrade
In the firmware Application, there is also a firmware upgrade code similar to the aforementioned Bootloader. It just needs to determine which area is invalid, save the received new firmware to the invalid area, and set the corresponding variables to be valid, and set the variables corresponding to the running area to be invalid. The next time the power is turned on, the Bootloader will jump to the new firmware for execution.

# Anchor firmware design-program structure
Now I will introduce how to write Application.

Because the anchor still has a lot of functions, several functions need to be executed "concurrently". "Concurrency" here refers to logic, because STM32 does not support preemptive execution/does not support multi-threading/has only one core, all codes can only be executed sequentially, and there will be no real concurrency.

Therefore, MCU cannot run naked and must use RTOS. At first, because I was familiar with uCOS, I used it. If I were to choose now, I would most likely use FreeRTOS.

- For the clock synchronization part, if it is used as a clock source, UWB clock synchronization packets need to be sent regularly;
- UWB chip processing is responsible for the status management of DW1000 and the sending and receiving of UWB data packets.
- Network processing, including DHCP Client/DNS Client/UDP Server/UDP Client, etc.
- Watchdog task

Let’s start with the simple part first.

## watchdog task

STM32 has a watchdog. If the dog feeding function is not called in time, the MCU will think that the program has hung up and it will automatically reset. Our program is divided into many tasks, and each task may fail, so we create a watchdog task. Each task has a watchdog variable, which is set to 0 by each task in time. The watchdog task loop adds 1 to these variables and determines the value of these variables. If it exceeds a certain limit, the corresponding task is judged to have hung up. , reset the MCU. The watchdog task is also responsible for feeding the dogs.

In short, we must ensure that each task can work normally and reset it immediately when the task hangs up.

## DW1000 processing tasks
Considering that multiple DW1000s may be built into one anchor in the future, we define DW1000 related data as a structure:

```C
typedef struct __DW1000_MODULE__
{
	uint16_t		Module_SPI_Prescaler;
	SPI_TypeDef*		SPIn;
	GPIO_TypeDef*		Module_SPI_Port;
	uint16_t		Module_SPI_SCK_Pin;
	uint16_t		Module_SPI_MISO_Pin;
	uint16_t		Module_SPI_MOSI_Pin;
	GPIO_TypeDef*		Module_SPI_CS_Port;
	uint16_t		Module_SPI_CS_Pin;
	GPIO_TypeDef*		Module_RSTn_Port;
	uint16_t		Module_RSTn_Pin;
	GPIO_TypeDef*		Module_RST_IRQ_Port;
	uint16_t		Module_RST_IRQ_Pin;
	uint32_t		Module_RST_IRQ_EXTI_Line;
	uint8_t			Module_RST_IRQ_EXTI_PortSource;
	uint8_t			Module_RST_IRQ_EXTI_PinSource;
	IRQn_Type		Module_RST_IRQ_EXTI_IRQn;
	GPIO_TypeDef*		Module_IRQ_Port;
	uint16_t		Module_IRQ_Pin;
	uint32_t		Module_IRQ_EXTI_Line;
	uint8_t			Module_IRQ_EXTI_PortSource;
	uint8_t			Module_IRQ_EXTI_PinSource;
	IRQn_Type		Module_IRQ_EXTI_IRQn;
	uint8_t			Module_IRQ_EXTI_USEIRQ;
	uint8_t			Module_RST_IRQ_EXTI_USEIRQ;
	uint8_t crystal_oscillator_trim;	
	uint8_t rx_buffer[RX_BUF_LEN];		
	DW_EVENT DWEvents[MAX_EVENT_NUMBER];
	uint8_t DWEventIdxIn, DWEventIdxOut;
	DW_EVENT dw_event_g;
	CLOCKSOURCE_SYNC_INFO clockSourceSyncInfo[CLOCKSOURCE_SYNC_INFO_NUM];
	SEEN_CLOCK_SOURCE seenClockSources[SEEN_CLOCK_SOURCE_NUM];
	dwt_local_data_t dw1000Local;
} DW1000_MODULE;
typedef DW1000_MODULE* PDW1000_MODULE;
```
In this case, some code can be reused, and the program will be more flexible.

I want to complain about DecaWave here. The working condition of DW1000 is not very stable. Sometimes some strange phenomena occur. Therefore, we must often judge the status of DW1000. If it is not in the expected status, it is determined to be dead, and we must reset DW1000 in time. Of course, there aren’t many of these cases, but when they are, we have to deal with them. Otherwise, at the customer site, the anchor and firmware are alive, but the DW1000 is dead and cannot send or receive data, which would be very embarrassing.

The main job of the anchor is to receive UWB data packets. In order to ensure that the received data packets can be processed in time without affecting the reception of subsequent UWB packets, we have created a queue for caching. The DW1000 task only receives UWB packets, puts them in the queue immediately after receiving them, and then processes subsequent UWB packets. Packets in the queue are processed by other tasks.

## Network Tasks
The work of network tasks is more complicated.

One of its most important tasks is to reassemble the received UWB into a new data packet and send it to the location engine.

Other tasks are auxiliary.

**DHCP client**. In many cases, IP address management is troublesome. If each anchor configures IP addresses one by one, it will be a laborious task. Generally speaking, unless the customer requires each device to be configured with a specified IP address, it is more convenient to use DHCP to assign IP addresses. The W5500 chip has a DHCP client code example, just copy it and modify it. Its code is not complete in processing, such as the renewal of DHCP IP address contract after expiration, and how to deal with IP address conflicts, etc. We have done some improvement work on the code.

**DNS client**. In fact, it doesn’t need to be used. If the location engine is not in the local area network, but in a far away place, such as a cloud host on the Internet. It would be better to use DNS to specify the address of the location engine. In case the IP address of the location engine changes, the domain name can remain unchanged. We do not use DNS. Most of our previous customers are within the local area network, and only a few have placed their location engines on the Internet.

**UDP**. We use UDP to implement automatic discovery of location engines. If configured for automatic discovery of the location engine, the network task will check whether a TCP connection is established with the location engine. If not, it will broadcast a UDP packet (location engine discovery packet) and request a reply from the location engine. After the location engine receives the UDP packet, it will reply with information such as its own IP address. The anchor will establish a TCP connection with the location engine.

**Anchor configuration**. In earlier versions, we used UDP to transmit anchor configuration information. The advantage of using UDP is that it can broadcast. The disadvantage is that UDP is connectionless and may be lost. Later, we uniformly changed to using TCP to transmit anchor configuration information. The anchor configuration program first uses UDP broadcast to find out which anchors are in the LAN, and then establishes TCP connections with them to transmit configuration information in both directions.

# Anchor Firmware Design-Miscellaneous Functions

**Reset factory setting**. In order to ensure data security, the anchor is set with a password, and the anchor configuration program must use the correct user name and password to configure the anchor. Since it is a password, there is a possibility of forgetting it. In addition, it is also possible that the user configured the anchor and messed up the data, and was too lazy to change it one by one, so we designed a factory reset function.

```C
// RS232端口 GPIOA (被挪用于检测是否恢复出厂设置)
#define USART1_TX		    PA9 	// out
#define USART1_RX		    PA10	// in
#define USART1_PORT			GPIOA

if ((!GPIO_ReadInputDataBit(USART1_PORT, USART1_TX)) || (!GPIO_ReadInputDataBit(USART1_PORT, USART1_RX))) {
	//== 恢复出厂设置
	RestoreToFactoryConfig();
	HardReset();
}
```
USART1_TX and USART1_RX are useless on factory products. Correspondingly, PA9 and PA10 are used to detect whether to restore factory settings. When you need to restore factory settings, short one of these two pins to GND and power on again.

**Alarm function** . A customer plans to use our system in a coal mine (important reminder, the product requires intrinsic safety certification before being used in a coal mine) and needs to add an alarm function. When a danger occurs, the system can send an alarm message to the tag so that workers carrying the tag can receive evacuation signals in time. The process is roughly like this. The application calls the interface of the location engine. The location engine sends the data packet to the clock source (or the anchor that plays the role of the clock name) through the network. The clock source sends a UWB alarm data packet in the task of sending the clock synchronization data packet.
After the tag receives the UWB alarm data packet, it will immediately alarm (red light, or vibrate). There are two types of alarm messages: broadcast to everyone, and tags with specified IDs.

# Anchor firmware design - trap you may encounter
**The first trap is the status of DW1000**. It's been said before. We need to regularly check the working status of DW1000 in the system. If it is abnormal, reset DW1000 in time.

**The second trap is the W5500 crystal oscillator**. When we first mass-produced the anchor, the W5500 used a passive crystal oscillator. This was a normal circuit design. Our previous test boards were all designed in this way. The W5500 we used in some previous products also used passive crystal oscillators. However, the production of the anchor was completed. After receiving the goods, the test found that the W5500 crystal oscillators of many anchors did not vibrate. At that time, I tried various methods to find out the cause, including changing the pad capacitor, re-soldering, etc. This vibration starting problem seems to be in an uncertain state. Some will start by themselves after a period of time after powering on, some will never vibrate, and some will even start to vibrate and then stop by themselves. As a result, we have to judge the status of the W5500 during firmware initialization to see if it is normal, and we also frequently judge the status of the W5500 during network tasks. Later, our hardware design switched to active crystal oscillator, and everything was OK. After that, our other products using W5500 all used active crystal oscillators.

This is the second article on anchor firmware design. I will go back and update some details later.


*By the way, I'm looking for a job. Due to business reasons, the company was dissolved. I have 30+ years of work experience, 20+ years of experience in C/C++/Java/Delphi, and a little bit of Javascript/Python/Lua. In 199x, I wrote a Chinese character system in x86 assembly, and later used Delphi to write a mail server. I have written countless application systems, and the code I have written should exceed 2 million lines. 10+ years of hardware design experience, designing multiple embedded products. In the early stages of this UWB location system, the hardware and software were all developed by me alone, and the team was only built later. Base Guiyang, China, or remotely. If there are job opportunities, please contact me for a detailed resume.*

