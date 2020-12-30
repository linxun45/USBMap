# USBMap

用于在macOS中映射USB端口并创建自定义注入 kext 的Python脚本。

***

# 特征

- [x] 不依赖于USBInjectAll
- [x] 可以映射XHCI（芯片组，第三方和AMD），EHCI，OHCI和UHCI端口
- [ ] ~~可以映射USB 2集线器~~ *目前已停用*
- [x] 根据类名称而不是端口或控制器名称进行匹配
- [x] 允许将别名设置为发现中最新出现的填充端口
- [x] 通过会话ID而不是断开的端口地址来聚合连接的设备
- [x] 可以使用最佳方法生成ACPI以根据需要重命名控制器或重置RHUB设备

***

# 索引

- [Installation](#installation)
- [Vocab Lesson](#vocab-lesson)
- [What Is USB Mapping?](#what-is-usb-mapping)
  - [A Little Background](#a-little-background)
    * [Finding Ports](#finding-ports)
    * [The Port Limit](#the-port-limit)
  - [Mapping Options](#mapping-options)
    * [Port Limit Patch](#port-limit-patch)
    * [USBInjectAll](#usbinjectall)
    * [SSDT Replacement](#ssdt-replacement)
    * [Injector Kext](#injector-kext)
    

***

## 安装

### 使用 Git

在终端中依次运行以下一行：

    git clone https://github.com/corpnewt/USBMap
    cd USBMap
    chmod +x USBMap.command
    
然后运行`./USBMap.command` 或者双击运行 *USBMap.command*

### 不使用 Git

您可以获得此仓库的最新zip [here](https://github.com/corpnewt/USBMap/archive/master.zip)。然后双击 *USBMap.command*

***

## 词汇课

*在开始之前，让我们先熟悉一些单词，因为词汇是**fun**!*

~~恐怖~~ 单词 | 定义
---------- | ----------
`Port` | 可以连接USB设备的物理连接。 这可能是PC或笔记本电脑等上的USB或USB-C端口。
`Header` | 与“Port”相似，但通常在主板本身上。 这些通常带有特殊的连接器，通常插入内部设备（AiO泵控制器，Bluetooth设备等），或者使用时延伸至机箱正面端口的扩展。
`Chipset` | 主板上的硬件负责组件之间的“数据流”（在我的Maximus X Code上，这是Intel的Z370芯片组）。
`Controller` | 负责管理USB端口的硬件。
`RHUB` 或 `HUBN` | 为每个端口提供信息的软件设备
`OHCI` 和 `UHCI` | USB 1.1/1.0 协议 - `OHCI` 是UHCI的“开放”变体。 它们都做大致相同的事情，但不能互换或兼容。
`EHCI` | USB 2.0 协议使用 1.0/1.1向后兼容。
`XHCI` | 用于USB 3和更高版本的USB协议-可以仿真 USB 2.0/1.1/1.0，但是是完全不同的协议。
`Port Personality` | USB端口的软件表示。 可能对应于物理 `port`，内部 `header`, ，或者可能是孤立的。
`Mapping` | 在这种情况下，确定哪个“port personalities”对应于哪个“controllers”上哪个“ports”的过程。  
`Full Speed`/`Low Speed` | USB 1.x
`High Speed` | USB 2.0
`Super Speed` | USB 3+
`Kexts` | A contraction of **K**ernel **Ext**ension - 这些很像驱动程序，并且扩展了内核的功能。
`Injector` or `Codeless` kexts | 一种特殊的kext，没有二进制文件，仅扩展了另一个kext的功能。 通常用于添加支持的设备或其他信息。

***

## 什么是USB映射？

*好的孩子们，请拿出您的制图工具包，我们将进行制图！*

如果您到目前为止一直在认真阅读，您可能在[Vocab Lesson]中找到了简短的定义(#vocab-lesson)。 不过，我们将在此基础上进一步扩展！ 简而言之，USB映射是用于确定哪些端口个性对应于哪些物理端口的过程。

### 一点背景

*在优胜美地的辉煌年代，我们被宠坏了。 Hackintoshes在技术领域中自由漫游，懒洋洋地在肥沃的土地上萌芽的大量USB端口上吃草...然后El Capitan出现了-吹捧着鼠标光标的技巧，当您绕一束时，它会变大（呃。 （可能还有其他有用的功能），我们哈克牧场主收集了我们的牲畜，睁大眼睛跑到诱人的绿色牧场上，尽管我们几乎不知道，苹果在代码中偷了一些东西，这对我们双方来说都是荆棘 即将推出的操作系统版本... *

关于USB的一些*主要*幕后更改来自10.10 到 10.11!

#### 查找 Ports

![Ports!](/images/look-sir-ports.png)

El Capitan 更改了操作系统查找可用USB端口的方式。 该发现以3种方式完成，并按以下顺序排列优先级：

1. Ports defined in injector kexts - OSX/macOS has some built-in injectors that define ports based on SMBIOS.
    * In 10.11-10.14 the port information is stored in `/System/Library/Extensions/IOUSBHostFamily.kext/Contents/PlugIns/AppleUSB[protocol]PCI.kext/Contents/Info.plist` where `[protocol]` is one of OHCI/UHCI/EHCI/XHCI
    * In 10.15+ the port information has been moved to a single file at `/System/Library/Extensions/IOUSBHostFamily.kext/Contents/PlugIns/AppleUSBHostPlatformProperties.kext/Contents/Info.plist`
    * These injectors match specific named devices in the IORegistry (XHC1, EHC1, EHC2)
    * We **do not** want to match with these, as your motherboard likely doesn't have the same ports, or same port number configuration as an iMac or otherwise...
2. If no devices match the injectors - OSX/macOS falls back on ACPI data.  Many motherboard manufacturers define RHUBs and USB ports in their DSDT, some use SSDTs, others don't define them at all, but *the most frustrating ones* only define **some** ports, causing the OS to ignore any that aren't defined.
3. If there's no ACPI data for the ports, OSX/macOS will then ask the hardware directly *"Hey bud, what kind of port action you rockin?"*

*Okay, so if we don't match built-in injectors, and we don't have ACPI information for our ports (or they're all defined properly), we're good, right?*

Uh... Let me set the scene...

#### The Port Limit:

*您终于有了安装USB，当您将那把黑色魔法小乐器插入USB端口并摇动电源按钮时，汗水倾泻到了额头上。机器栩栩如生，风扇呼啸而来-灯都亮着。您的显示屏闪烁，睁开它的隐喻性眼睛，BIOS初始屏幕以其“我是80年代的未来梦想”的美感向您致意-随后是您选择的启动管理器的启动选择器。选择器移至您安装的USB，然后有条按Enter键。详细的文字在屏幕上逐行细致地跑动，让您窥视幕后并进入引导过程的核心……但是……不对劲。文字乱码...大的“禁止”标志正好贴在显示屏的中央，并且似乎在嘲笑您，因为启动暂停会保存一行缓慢重复的乱码文字。当您在几乎是破碎的文本上跟踪它们时，您的眼睛起眼睛...“仍在等待根设备...” *

*等等...发生了什么事？*

好吧，影响我们Hackintoshers的最大变化之一是，苹果现在对每个控制器施加了15个USB端口限制。 从表面上看，这听起来并不是什么大问题。 大多数主板的物理端口远远少于15个-更不用说一些具有可以共享负载的第三方芯片组（因为*每个*控制器都有自己的15个端口限制）。

*为什么15？*

尽管这似乎有些武断，但这实际上是将端口寻址限制为单个4位地址的一种方法。 macOS / OSX使用十六进制寻址（0-9A-F）显示设备-并且当设备上连接了其他* stuff *（例如，USB RHUB会连接端口）时，它会使用第一个设备的地址作为起点， 并增加所连接设备的下一位数字。

例如，我的芯片组XHCI控制器的RHUB显示为XHC @ 14000000。 这意味着我们添加到RHUB的第一个端口显示在地址“ @ 14100000”，第二个显示在“ @ 14200000”，第十个显示在“ @ 14a00000”，第十五个显示在“ @ 14f00000”。 从RHUB离开的每个允许的端口都非常整洁地适合一位数字的寻址（可爱！）。 当您发现`f`地址上方的*任何内容*被忽略**时，都会有些担心。

*我的主板附近没有15个端口，所以...有什么用？*

我很高兴你问！ 大多数现代主板USB控制器都利用XHCI协议来处理其所有USB端口，而USB 3有点“鬼*”。 肯定比其前辈更加狡猾。

当EHCI（USB 2.0）出现时，它实际上只是对现有UHCI / OHCI（USB 1.x）协议的扩展，该协议将端口路由的大部分责任移至了硬件端，因此使用同一物理端口实现向后兼容性 布局很容易确保。 实际上，许多早期的EHCI控制器与UHCI或OHCI控制器并存。

XHCI后来因USB 3的宏伟计划而陷入困境-并完全取代EHCI / UHCI / OCHI，同时将所有USB优点包装在一个整洁的小包装中。 我们的朋友XHCI与以前的协议有点不同，因此需要一些仿真才能实现该功能（有些对此仿真存在问题，但大多数人永远不会注意到）。

(您可以阅读有关不同协议的更多信息 [here](https://en.wikipedia.org/wiki/Host_controller_interface_(USB,_Firewire))!)

*嗯，那其中有什么偷偷摸摸的？*

让我们看一下USB 3端口的内部（图片由usb.com提供）：

![USB3](images/USB3.png)

那里有9个针脚-但它们的设置非常明确。 USB 2或先前版本的设备仅会使用前4个引脚，而USB 3+设备实际上会利用这4个引脚，*加上*额外的5个。每个物理USB 3端口都带有*分离的个性！*当USB 2 或将先前的设备插入USB 3端口，则*被视为USB 2端口特征-而插入同一端口的USB 3设备被视为USB 3端口特征。 这意味着每个物理USB 3端口**占用该控制器上有限的15个端口中的2个**。

让我们看一下我的Asus Maximus X Code上的端口，并详细说明所有工作原理。 每 [spec page](https://www.asus.com/us/Motherboards/ROG-MAXIMUS-X-CODE/specifications/), 我们可以在* USB端口*标头下看到以下内容：

```
ASMedia® USB 3.1 Gen 2 controller :
1 x USB 3.1 Gen 2 front panel connector port(s)
ASMedia® USB 3.1 Gen 2 controller :
2 x USB 3.1 Gen 2 port(s) (2 at back panel, black+red, Type-A + USB Type-CTM)
Intel® Z370 Chipset :
6 x USB 3.1 Gen 1 port(s) (4 at back panel, blue, 2 at mid-board)
Intel® Z370 Chipset :
6 x USB 2.0 port(s) (4 at back panel, black, 2 at mid-board)
```

Let's break this down - there are 2 *separate* ASMedia controllers, one with a single USB 3.1 Gen 2 front panel connector, the other with 2 USB 3.1 Gen 2 ports on the back panel.  Neither of those should surpass the limit, as they're both only going to provide 2 USB 3.x ports, and since we know that each physical USB 3 port *counts as 2*, we can do some quick math and find how many total port personalities each of the ASMedia controllers provide:

- 1 USB 3.1 Gen 2 front panel connector (which breaks out into 2 physical USB 3.1 ports - *each with split personalities*) gives us **4 port personalities total**.
- 2 USB 3.1 Gen 2 (2 physical USB 3.1 ports - *each with split personalities*) gives us **4 port personalities total**.

4 personalities for each of the separate controllers is well under the 15 port limit, so we're **A OK** in that regard.


Looking on, there are 2 entries for the Z370 chipset, but this is a bit different.  There is only *one chipset*, and as such, both of these entries *share* the same controller.  That tightens up our wiggle room a bit, so let's look at how many total port personalities we're working with...

- 6 USB 3.1 Gen 1 (*each with split personalities*) gives us **12 port personalities**.
- 6 more USB 2.0 ports (these are physically USB 2, and do not have split personalities) gives us **6 port personalities**.

Combine the two values (since they share the chipset controller), and we're sitting at a toasty **18 port personalities total**.  *This is over the 15 port limit!*

This is where mapping really shines, as it gives us a way to *pick* which ports we'd like to omit, thus keeping us under the limit in a controlled way.  Let's look at a few options for mapping next!

***

### Mapping Options

As USB mapping has been a necessity since 2015, there have been a few approaches put into place to try and mitigate issues.

#### Port Limit Patch

*You stand in the sun, the gentle breeze stealing some of the heat as you walk through the orchard picking apples - you've gotten at least 10lbs put together at this point - a solid amount!  Your hands shield your eyes from the sun as you stop to catch your breath; you catch a glimpse of your OS walking up as it hands you a bag for your haul of apples.  A measly 5lb bag... How can you fit 10lbs of apples in a 5lb bag?*

One of the mitigations for the USB port limit is to, well, just patch it out.  Seems *epic*, no?  The port limit patches have been in circulation for some time and exist to lift the 15 port limit patch in only a few key places so all ports the OS can "see" are available.  OpenCore has a quirk that attempts to do this on any OS version called `XhciPortLimit`.

*That sounds amazing, my 5lb bag is bigger on the inside than the outside?  Time to shove all these apples in!*

While it sounds like the best-case solution, it does come with some drawbacks... The port limit is *hardcoded* in a ton of places all over the OS, and as we're only lifting it in a few, this causes access outside the bounds of a fixed array.  We're accessing things that shouldn't even be there, and that can cause some odd or unpredictable side effects.  Everyone who sees you skipping along with your bag of apples will *know* that it's only 5lbs, even if it's filled with 10lbs worth.

Ultimately, it's considered *best practice* to **only** leverage the port limit patch for the mapping process, and then to disable it.

**Pros:**

* Gets around the 15 port limit

**Cons:**

* Changes with each OS version
* Causes issues with fixed array bounds
* Only patched in some spots
* Can cause unpredictable side effects

#### USBInjectAll

Remember those *super cool* ports that were only sorta sometimes defined in firmware/ACPI?  Well - RehabMan saw an opportunity, and came up with a solution.  He wrote a kext that has a ton of different Intel chipset controller ports hardcoded and named (for real, it's [*a ton*](https://github.com/RehabMan/OS-X-USB-Inject-All/blob/master/USBInjectAll/USBInjectAll-Info.plist)).  What USBInjectAll tries to do is inject **all possible ports** for a given controller.  From there, you can determine which correspond to physical connections, decide which to keep, and discard whatever you need to bring you under the limit.

**Pros:**

* Has a ton of hardware pre-defined
* Utilizes boot args to map in sweeps (USB 2 port personalities or USB 3 port personalities)
* Can be customized with an SSDT

**Cons:**

* Another kext to load, with code to execute
* Uses the IORegistry as "scratch paper"
* No longer maintained
* Cannot map third party or AMD ports

#### SSDT Replacement

If you feel confident in your [ACPI](https://uefi.org/sites/default/files/resources/ACPI_6_3_final_Jan30.pdf) abilities, you can redefine the RHUB and ports for whichever ports you'd like to utilize.  You can see an example of this in Osy's HaC-Mini repo [here](https://github.com/osy86/HaC-Mini/blob/master/ACPI/SSDT-Xhci.asl).

**Pros:**

* Clean - fewer kexts (especially to inject)
* Potentially more durable if Apple changes its dependence on injectors

**Cons:**

* Tougher to accomplish - ACPI isn't many people's language of choice, and it can be tough to know where to find what you need, what to move over, etc.

#### Injector Kext

*So, we know we can leverage the port limit patch temporarily, we don't want to use the IORegistry as scratch paper, and we're all afraid of ACPI - what's the solution for us?*

Well, we can actually accomplish this *the same way* Apple has!  Apple's injector kexts for USB just extend the functionality of a target kext by providing information on which ports should be there, the port types, and what SMBIOS these settings apply to.  Let's grab an example from Big Sur's `AppleUSBHostPlatformProperties.kext` for iMac17,1:

![iMac17,1 Ports](images/imac171.png)

Looking at this image, we can see that the `IONameMatch` corresponds to `XCH1`, which means the OS will look for that when running this SMBIOS, and allow the following ports - HS02, HS03, HS04, HS05, HS06, HS10, SSP1, SSP4, SSP5, SSP6.  Each of these ports are referenced by their `port` number (HS02 is port number `<02000000>` - which when converted from little-endian Hex to an integer is port 2).  They're also given a `UsbConnector` value - which corresponds to the type of connection they use, some common values are 0 for USB 2 physical ports, 3 for USB 3 physical ports, 255 for internal ports.

We can actually leverage this template to create our own injector kext that maps the ports on our motherboard!  This is the approach we'll use for this guide, and the approach that USBMap uses.

**Pros:**

* Uses the same approach Apple does
* No extra code to execute
* Can be mostly automated

**Cons:**

* May not always be Apple's approach
* Not as clean as an ACPI-only solution
