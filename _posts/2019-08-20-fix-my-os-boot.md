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
<small class="img-hint">歪果仁的笑话怎么一点都不好笑</small>


出现系统无法进入的情况后直接想到的就是引导坏了，查看启动方式，果然连硬盘都检测不到了。按F2进行检查，内存检测通过，但硬盘短时检测fail, 在网上搜索解决方案时，惠普官方网站说出现这个结果意味着硬盘的寿命快到了（并且还提供了客服的联系方式/捂脸）.

![java-javascript](/img/in-post/os-fix/nodisk.jpg)

找同学借了个ubuntu 16.04的live CD(启动盘), 看看能不能先进入系统，可以试着输入一些命令进行修复，但是在load kernel的过程中出现了sector 2048 i/o error, 上面也有提到不能挂在NTFS分区，同学说只要用命令NTFXfix命令（大概是这个， 没有去找）就可以修复，不过现在也没有控制台，而且也可以看出主要的问题出在硬盘上，现在还不能确定是引导损坏或者是整个硬盘已经损坏。

![java-javascript](/img/in-post/os-fix/ubuntu.jpg)

于是借到了一个大白菜PE盘，经过漫长的等待（加载PE桌面很慢），我的电脑终于出现了图形界面。


![java-javascript](/img/in-post/os-fix/2048.jpg)

![java-javascript](/img/in-post/os-fix/efi.jpg)

![java-javascript](/img/in-post/os-fix/fix.jpg)

#### 一些资源


