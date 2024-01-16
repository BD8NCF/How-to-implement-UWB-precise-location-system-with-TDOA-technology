这是一个系列文章《如何从零开始实现TDOA技术的 UWB 精确定位系统》第3部分。

**重要提示（劝退说明）：**

> **Q：做这个定位系统需要基础么？**

> A：文章不是写给小白看的，需要有电子技术和软件编程的基础

> **Q：你的这些硬件/软件是开源的吗？**

> A：不是开源的。这一系列文章是授人以“渔”，而不是授人以“鱼”。文章中我会介绍怎么实现UWB定位系统，告诉你如何克服难点，但不会直接把PCB的Gerber文件给你去做板子，不会把软件的源代码给你，不会把编译好的固件给你。我不会给你任何直接的结果，我只是告诉你方法。

> **Q：我个人对UWB定位很兴趣，可不可以做出一个定位系统？**

> A：如果是有很强的硬件/软件背景，并且有大量的时间，当然可以做得出来。文章就是写给你看的！

> **Q：我是商业公司，我想把UWB定位系统搞成一个商业产品。**

> A：当然可以。这文章也是写给你看的。如果你想自己从头构建整个系统，看了我的文章后，只需要画电路打板；构思软件结构再编码。就这样，所有的难点我都会在文中提到，并介绍了解决方法。**你不需要招人来做算法研究**。如果你想省事省时间，可以直接购买我们的电路图(AD工程文件)，购买我们的软件源代码，然后快速进入生产环节。（网站: https://uwbhome.top）

*前一篇文章介绍了时钟同步的方法，是为了尽快拿点干货出来，免得读者觉得我只写一些泛泛而谈的大路货，没技术含量。下一步慢慢写吧。*

# 基站固件设计-Flash布局

给MCU写固件的时候，如果是初学者，可能一上来直接就开始写应用程序了。老手都会先考虑布局，都要做些什么事，它们之间的关系是什么等等，要考虑周全。

基站的Flash分为4块：Bootloader、Application1、Application2、Data。

基站固件设计-在线升级
产品的固件可以现场升级很重要。现场升级是指不用把已经安装好的产品拆下，直接通过局域网之类的，在客户现场就可以升级固件。要知道，当产品已经安装在几千公里外的客户现场，突然发现固件中有个bug，你怎么办？如果能现场升级，在办公室把代码修正后，生成新固件，发给现场的客户，在现场升级固件。在这个过程中，你根本不需要离开你的办公室。

如果你的产品不能现场升级固件，对不起，发现bug后，只能请客户把产品拆下，寄回公司，你刷入新固件，再寄给客户......想起来都头大。

# 基站固件设计-Bootloader

Bootloader是启动代码，Application1/Application2是应用程序，Data放配置数据。

上电的时候，Bootloader中的程序先被执行，它判断Application1/Application2哪个有效，就跳转到有效的那个Application去执行。在Data区有两个变量: Application1Valid和Application2Valid，如果为True表示对应的Application有效。

在生产阶段，负责生产的工人只负责刷入Bootloader，这时，Application1Valid和Application2Valid都是False。

上电后，如果找不到有效的Application，Bootloader会启动网络部分的代码: DHCP Client/TCP Client/DNS Client/UDP Server/UDP Client等。如果它连接在网络，会一直在网络中发出一个UDP包，请求初始化。

出厂初始化程序收到请求初始化的UDP包后，会登记新基站的基本信息，如MCU ID等，然后给它分配一个 EUI64 的地址，这个地址以后就是这个基站的ID了。还会分配一个系列号。然后还要写入缺省的配置数据。出厂初始化程序接下来开始发送固件给新基站。Bootloader收到固件后，写入到一个无效的Application区（刚开始，Application1区肯定是无效的）。固件会被分很多个包发送。固件接收完成后，Bootloader会置Applicaiton1Valid为True，然后重启。

再次上电后，Bootloader发现Application1有效，就跳转到Application1去执行。

这就完成了新固件的启动。

其实，Bootloader中的网络相关代码和固件接收代码，只在出厂初始化的时候执行一次，以后就不会再被执行到。因为在出厂之后，Applicaiton1和Application2总有一个是有效的。

如果Bootloader发现两个区都有效，会把第二个区置为无效。只允许一个区有效，无效的那个区用保留给新的升级固件使用。

# 基站固件设计-固件升级

在固件的Application中，也有类似前述Bootloader的固件升级代码。只是它要判断哪个区无效，把收到的新固件保存到无效的那个区，并置对应的变量为有效，把正在运行的这个区对应的变量置为无效。下次上电时，Bootloader就会跳转到新固件去执行。

# 基站固件设计-程序结构

现在介绍Application怎么写。

因为基站的功能还是有点多，好几个功能需要“并发”执行。这里的“并发”是指逻辑上的，因为STM32不支持抢先式执行/不支持多线程/只有一个核，所有代码都只能顺序执行，并不会有真正的并发。

所以，MCU不能裸奔，得使用RTOS。当初，因为比较熟悉uCOS，所以就用它了。如果是现在选择，大概率会用FreeRTOS。

- 时钟同步部分，如果作为时钟源，需要定期发送UWB时钟同步包；
- UWB芯片处理，负责DW1000的状态管理，以及UWB数据包的收发。
- 网络处理，包括DHCP Client/DNS Client/UDP Server/UDP Client等。
- 看门狗任务

先从简单的部分开始介绍。

## 看门狗任务

STM32有看门狗，如果不及时调用喂狗函数，MCU就会认为程序挂了，它会自动复位。我们的程序分为很多个任务，每一个任务都有挂掉的可能，所以，我们搞一个看门狗任务。每个任务有一个看门狗变量，由各个任务及时把它置0，看门狗任务循环给这些变量加1，并判断这些变量的值，如果超过某个界限，就判断对应的任务挂了，就复位MCU。看门狗任务还负责喂狗。

总之，我们要保证各个任务都能正常工作，当任务挂掉后，立即复位。

## DW1000处理任务

考虑到将来可能会在一个基站内置多个DW1000，所以我们把DW1000相关数据定义为一个结构：

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
这样的话，有些代码就可以复用了，程序处理起来会灵活一些。

在这里要吐槽一下DecaWave，DW1000的工作情况并不很稳定。有时会出现一些奇怪的现象。所以，我们还要经常判断DW1000状态，如果不在期望的状态，判定死了，我们要及时对DW1000进行复位。当然，这种情况并不太多，但是只要有，我们就得处理。否则在客户现场，基站活着、固件活着，但是DW1000死了，无法收发数据，就很尴尬了。

基站的主要工作就是接收UWB数据包，为了保证收到的数据包能及时处理，不影响后续的UWB包的接收，我们搞了一个队列来做缓存。DW1000任务只管收UWB包，收到后立即放入队列，然后又处理后续的UWB包。队列中的包由其他任务来处理。

## 网络任务

网络任务的工作比较杂乱。

它最重要的一个工作就是把收到的UWB重新组装成一个新的数据包，发给定位引擎。

其它的工作就是辅助性的了。

**DHCP客户端**。很多时候，IP地址的管理很麻烦，如果每个基站都一一配置IP地址，是一件比较辛苦的事。一般来说，除非客户要求每台设置都要配置指定的IP地址，否则使用DHCP来分配IP地址是比较方便的。W5500芯片有DHCP客户的代码例子，直接复制过来改改就可以了。它的代码在处理上不太完整，DHCP的IP地址合约到期后的续签，还有如果有IP地址冲突怎么处理等等，我们对代码做了一些完善工作。

**DNS客户端**。其实可以不用的。如果定位引擎不在局域网内，而是在很远的地方，例如Internet上的云主机。使用DNS指定定位引擎的地址会好一些，万一定位引擎的IP地址变了，但是域名可以不变。我们没有使用DNS，之前的客户基本都在局域网内，把定位引擎放在Internet上的只有为数不多的几个。

**UDP**。我们使用UDP来实现定位引擎自动发现。如果配置为定位引擎自动发现，网络任务会检查是否与定位引擎建立起TCP连接，如果没有，它会广播一个UDP包(定位引擎发现包)，要求定位引擎给回复。定位引擎收到UDP包后，会回复自己的IP地址之类的信息。基站就会与定位引擎建立起TCP连接。

**基站配置**。在早期的版本中，我们使用UDP来传送基站配置信息。使用UDP的好处是可以广播，缺点是UDP是无连接的，有丢失的可能。后来我们统一改成使用TCP来传送基站配置信息。基站配置程序先通过UDP广播，查找局域网内都有哪些基站，然后与它们建立TCP连接，双向传送配置信息。

# 基站固件设计-杂项功能

**恢复出厂设置**。为了保证数据安全，基站设置有密码，基站配置程序必须使用正确的用户名和密码才能对基站进行配置。既然是密码，就有忘记的可能。另外也有可能用户配置基站，把数据弄乱了，自己懒得一个一个的改，所以我们设计了一个恢复出厂设置功能。

// RS232端口 GPIOA (被挪用于检测是否恢复出厂设置)
#define USART1_TX		        PA9	// out
#define USART1_RX		        PA10	// in
#define USART1_PORT			GPIOA

if ((!GPIO_ReadInputDataBit(USART1_PORT, USART1_TX)) || (!GPIO_ReadInputDataBit(USART1_PORT, USART1_RX))) {
	//== 恢复出厂设置
	RestoreToFactoryConfig();
	HardReset();
}
USART1_TX和USART1_RX在出厂的产品上没用，对应的是PA9和PA10被挪用于检测是否恢复出厂设置。需要恢复出厂设置时，把这两个引脚之一与GND短接，重新上电就可以了。

**报警功能**。有个客户打算把我们的系统用在煤矿(重要提醒，产品需要本安认证才用在煤矿)，需要增加一个报警功能。当发生险情时，系统可以发报警信息给标签，让携带标签的工人可以得时得到撤离信号。流程大致是这样，应用程序调用定位引擎的接口，定位引擎通过网络把数据包发给时钟源(或扮演时钟名的基站)，时钟源在发送时钟同步数据包的任务中发出UWB报警数据包，标签收到UWB报警数据包后，立即报警(红灯，或震动)。报警消息有两种：广播给所有人，指定ID的标签。

# 基站固件设计-可能会遇到的坑

**第一个坑是DW1000的状态**。前面已经说过了。我们在系统中要定期检查DW1000的工作状态，如果不正常，要及时对DW1000进行复位。

**第二坑是W5500的晶振**。我们第一次量产基站时，W5500使用的是无源晶振，这是很正常的电路设计，我们之前的测试板都是这样设计的。我们在之前的一些产品中用到W5500也是使用无源晶振。但是基站生产完成，收到货后测试发现有很多基站的W5500的晶振不起振。当时想了各种办法来找原因，换pad电容，重新焊接等等。这个起振问题，似乎处于一个不确定状态，有些上电一段时间会自己起来，有些就总是不起振，有些甚至开始起振了后面会自己停。以至于我们只好在固件初始化的时候判断W5500的状态，看它状态正常不，还在网络任务中经常性的判断W5500的状态。后来我们的硬件设计改用有源晶振，就一切OK了。再之后，我们其他用到W5500的产品中，都使用有源晶振。

*这是基站固件设计的第二篇文章，以后再回头修改修改充实一些细节吧。*

*顺便做个找工作的广告。因为经营上的原因，公司散伙了。本人30+年工作经验，顺手的语言C/C++/Java/Delphi，这个UWB定位系统在初期硬件软件都是我一个人弄出来的，产品成型之后才增加人手组团队。Base贵阳，或远程。如果有工作机会，请联系我要详细的简历。*

