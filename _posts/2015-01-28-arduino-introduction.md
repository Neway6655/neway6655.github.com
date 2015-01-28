---
layout: post
title: "Arduino Brief Introduction"
description: "Briefly introduce arduino"
category: open-source hardware
tags: [arduino]
---
{% include JB/setup %}

这是一篇关于Arduino的扫盲简文,用简单易懂的方式让大家了解Arduino到底是啥.

Arduino的官网上是怎么定义自己的：

>Arduino is an open-source electronics platform based on easy-to-use hardware and software. It's intended for anyone making interactive projects.


## Arduino = Arduino Board + Arduino Software ##
Arduino是一个可编程的电路板加一套编程开发环境的软件.说到这,可能很多人容易想到大学时代的MCU51单片机,那么,这Arduino和单片机有啥区别呢?我们先看一个例子：

如果有人问：“用单片机你会做个啥出来?”

我会回答：“做一个电子闹钟,还是带液晶显示的那种”（因为那个是我大学时的单片机课程作业）

但如果有人问我：“用Arduino做一个闹钟,咋样?”

我会回答：“行,那我做一个冬天会自动推迟闹铃的闹钟吧,天气越冷,就越晚一点闹铃”


这个例子并不是说用单片机就做不出来一个会自动推迟闹铃的闹钟,但做起来会麻烦很多,相比Arduino.

OK,那Arduino就能这么牛?

那我们先看看Arduino Board:

### Arduino Board ###

![Arduino UNO](http://arduino.cc/en/uploads/Main/ArduinoUno_R3_Front.jpg)

Arduino Board有一系列的板子,Arduino UNO是其中比较基础的一个,它除了有微控制器（单片机）之外,还有各个IO引脚,晶振,USB连接口,电源接口等等,这些其实就是一整套单片机的电路板,接上电源,USB接上电脑烧好程序就可以工作了.

但它的强大之处并仅仅在于提供了一套完整的电路板,而是还有一套完善的编程开发环境:

### Arduino Software ###

用[这套软件开发环境](http://arduino.cc/en/Main/Software),你可以轻易的和很多其他的外围设备集成,比如前面例子里提到的用于感测温度的传感器,Arduino提供了很多底层基本的代码库,让你和传感器的数据通信变得非常简单,而且还有很多示例代码可以参考.（开发语言是C）.

这对做软件开发的程序猿来说吸引力是不是很大?摆脱硬件上的阻碍,把更多的精力放到软件程序上.

除了这两点,Arduino还有一个及其重要的特点：

### Open Source ###

Arduino的硬件和软件都是开源的,这也是为什么它能火起来的重要原因之一.因为单单依靠Arduino Board,能做的事情还是非常有限的,但因为开源,Arduino的外围扩展板也越来越多,并提供开发包,从[这些Example](http://arduino.cc/en/Tutorial/HomePage)里可以看到用Arduino可以做的事情太多了.

所以说,Arduino的伟大在于它释放了人们无尽的想象空间,而把硬件软件结合的那些琐事留给自己.

现在,应该能理解它定义里的那句*"easy-to-use hardware and software"*了吧.

最后,附上[TED上的视频介绍](https://www.youtube.com/watch?v=UoBUXOOdLXY) .（观看需备梯子）