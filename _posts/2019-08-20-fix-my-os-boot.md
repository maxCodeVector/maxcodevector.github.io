---
layout:     post
title:      "Fix OS boot when suffer Bad Sector"
subtitle:   "My experience of fix my three-operate system by moving efi partition from bad sector"
date:       2019-08-22
author:     "Huang Yu'an"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Boot
    - Bad Sector
    - OS
---


前几天因为Manjaro系统在安装某个包的时候卡住了，点关机按钮也没有用，于是就长按电源键强制关机了，结果就GG了，再开机时屏幕显示找不到设备，请在硬盘上安装操作系统。最终折腾了一天，总算是把系统恢复了，特此记录一下。

![java-javascript](/img/in-post/os-fix/none.jpg)


出现系统无法进入的情况后直接想到的就是引导坏了，查看启动方式，果然连硬盘都检测不到了。按F2进行检查，内存检测通过，但硬盘短时检测fail, 在网上搜索解决方案时，惠普官方网站说出现这个结果意味着硬盘的寿命快到了（并且还提供了客服的联系方式/捂脸）.

![java-javascript](/img/in-post/os-fix/nodisk.jpg)

找同学借了个ubuntu 16.04的live CD(启动盘), 看看能不能先进入系统，可以试着输入一些命令进行修复，但是在load kernel的过程中出现了sector 2048 i/o error, 上面也有提到不能挂在NTFS分区，同学说只要用命令NTFXfix命令（大概是这个， 没有去找）就可以修复，不过现在也没有控制台，而且也可以看出主要的问题出在硬盘上，现在还不能确定是引导损坏或者是整个硬盘已经损坏。

![java-javascript](/img/in-post/os-fix/ubuntu.jpg)

于是借到了一个大白菜PE盘，经过漫长的等待（加载PE桌面很慢），我的电脑终于出现了图形界面。

pe只帮我检测到了windows系统（我电脑上还装了ubuntu18.04和manjaro18.0的系统）。先尝试着修复引导，结果修复失败，说分区有问题，让先修复分区。于是打开data genus,发现自己的分区大部分都在，也能打开，除了efi分区，不过并不知道是因为这个分区被保护还是被损坏导致打不开。然后打开引导记录查看工具，提示某个扇区出现crc校验不通过，无法读取，这就更确定了是磁盘硬件层面的损坏而不仅仅是引导被删了的原因。


![java-javascript](/img/in-post/os-fix/2048.jpg)

总结目前掌握的情况，硬盘自检失败，efi分区无法打开，引导记录查看工具提示某扇区crc校验不通过，那么试试看能否定位到坏掉的扇区。结合用ubuntu启动盘启动的情况，提示io error, sector 2048不可读，于是使用扇区编辑工具查看第2048扇区，果然发现了异常，从2048开始连续8个扇区记录均为0x3f。可以确定，2048号扇区发生损坏。

这里解释一下为什么是8个，我的逻辑扇区大小为512B, 8个就是4KB，刚好是一个扇区的物理大小，这正好说明了是一整个扇区损坏。

而使用diskpart命令查看分区情况，可以看出第2048扇区就是efi引导分区所在的第一个分区，这个就解释了整个系统无法启动的原因。

![java-javascript](/img/in-post/os-fix/efi.jpg)

尝试修复，经过删除重建以及格式化efi分区，发现2048扇区仍然不可读，可以确定以及是硬件层面的损坏。于是决定将efi分区起始位置后延，由于efi分区必须是磁盘上的第一个分区，所以分区大小不得不减小。
```
create partition efi size=251 offset=10240
format quick fs=FAT32
```
<small class="img-hint">指定偏移为10240字节，也就是20480扇区</small>


经过重建后，格式化也成功了，这是个很好的兆头。然后再用引导修复工具，windows的引导成功被修复。

![java-javascript](/img/in-post/os-fix/fix.jpg)
<small class="img-hint">成功修复windows引导</small>

只要一个引导被修复后一切都好办了，之后linux启动盘也可以正常进入系统了，使用boot-repair很快就把ubuntu的引导修复好了。不过遗憾的是Majaro的引导最后也没有恢复，并不知道是什么原因，毕竟引导被删的开始就是从它开始的。最后重装了Majaro, 不过好在home所在的那个分区可以复用，很多数据都还在，而且也有蛮多应用安装在那，然后又是配置环境，折腾了近乎一个周末，电脑终于恢复到原来的样子了。

但是硬盘自检还是失败的。毕竟是硬件层面的损坏。在此推荐一个linux平台叫做gsmartcontrol的检测工具，可以检测硬盘健康情况，原理是利用了硬盘本身的SMART自检功能。（Sadly, 后来我的磁盘被检测到坏了3个扇区， 2048，44724656，165160216，也就是1M, 21G, 78G字节开始的地方，好在这些地方都在Windows C盘，现在也不怎么用，看来电脑迟早都要换的了）

#### 最后
感谢借我PE盘的同学，感谢一直远程指导我的某Yin, 感谢为我做启动盘的同学。


