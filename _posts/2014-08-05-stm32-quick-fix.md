---
layout: post
title: Quick fix for st-util unable to flash Discovery board
---

I was developing an application which required SIM communication. Not being carful enough, I setted up PA14 as output and set it high, and st-util was no longer able to connect to it. Here is how I "fixed" it.

**EDIT:**

First time I thought that the problem was caused by a brief short-circuit caused by hot-plugging the SIM. This was not the cause. After some research, I found out that I was setting PA14 as output and high, and SWDIO is connected to this pin. This is why there were those problems, and why "Connect under reset" worked, and why the problem went away after erasing the flash.

The symptoms were: If I tried to connect normaly using st-util (https://github.com/texane/stlink), the connection LED on the STM32 board would turn yellow and I would get this output:

~~~
2014-08-05T08:47:19 INFO src/stlink-usb.c: -- exit_dfu_mode
2014-08-05T08:47:19 INFO src/stlink-common.c: Loading device parameters....
2014-08-05T08:47:19 WARN src/stlink-common.c: unknown chip id! 0xe0042000
Chip ID is 00000000, Core ID is  00000000.
Target voltage is 2903 mV.
Listening at *:4242...
~~~

I started googling what could be the problem, and found that maybe holding reset while connecting may help. The LED no longer turned yellow, it was green how it was supposed to be, however, I was still unable to program the flash. This was the st-util output:

~~~
2014-08-05T08:47:37 INFO src/stlink-common.c: Loading device parameters....
2014-08-05T08:47:37 WARN src/stlink-common.c: unknown chip id! 0
Chip ID is 00000000, Core ID is  0bb11477.
Target voltage is 2898 mV.
Listening at *:4242...
~~~

As we can see it properly detected the Core ID now, but it still could not get proper chip ID.

The solution was to use ST LINK utility running under Windows. I set up virtual machine with windows and enabled USB subsystem, selected "Connect under reset" setting and connected to the board sucessfully.

~~~
08:48:52 : ST-LINK SN : Old ST-LINK firmware/ST-LINK already used
08:48:52 : ST-LINK Firmware version : V2J14S0 (Need Update)
08:48:52 : Old ST-LINK firmware detected!
                  Please upgrade it from ST-LINK->'Firmware update' menu.
08:48:52 : Connected via SWD.
08:48:52 : Connection mode : Connect Under Reset.
08:48:52 : Debug in Low Power mode enabled.
08:48:53 : Device ID:0x440
08:48:53 : Device flash Size : 64KBytes
08:48:53 : Device family :STM32F030xx/F051xx/F071xx
~~~

After that, I just erased the device (Ctrl+E) and after that, I was able to use st-util again

**EDIT:**
This was the old solution. In the meantime I came up with another. All you need to do is to press the reset button before the flash operation, and release it just in the right moment, that way you can program the flash from st-util

~~~
2014-08-05T08:49:47 INFO src/stlink-common.c: Loading device parameters....
2014-08-05T08:49:47 INFO src/stlink-common.c: Device connected is: F0 device, id 0x20006440
2014-08-05T08:49:47 INFO src/stlink-common.c: SRAM size: 0x2000 bytes (8 KiB), Flash: 0x10000 bytes (64 KiB) in pages of 1024 bytes
Chip ID is 00000440, Core ID is  0bb11477.
Target voltage is 2918 mV.
Listening at *:4242...
~~~

Thinking back about this, this problem seems trivial, however, installing Windows to save a Discovery board was an act of desperation. If you have similar symptoms, this may save you some time looking a solution.

It would make a nice pull request implementing "Connect under reset" functionality to Texane's stlink.
