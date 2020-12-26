---
layout: post
title: "Debugging the ChameleonMini with GDB and VSCode"
excerpt: "Because it's easier than compiling in Atmel Studio"
tags: [Hardware, Debugging, NFC, AVR]
image:
  feature: chameleondebugging.jpg
---

I was working on a new codec for the [ChameleonMini](https://github.com/emsec/ChameleonMini) and it become necessary to debug in a reasonable fashion (eg. with something better than a blinking led).

The project uses an Atmel XMega 128A4U MCU, so the first choice for debugging would probably be Atmel Studio. However, the project hasn't been setup to use the proprietary IDE, but rather a makefile. For the life of me I haven't been able to successfully import the Makefile and build with Atmel Studio itself. Also, given that I use linux daily, it bothered me to reboot in windows every time I wanted to develop.

Luckily my debug board, the cheapo [Atmel ICE PCB](https://www.digikey.it/product-detail/en/microchip-technology/ATATMEL-ICE-PCBA/ATATMEL-ICE-PCBA-ND/4753383), is supported by [avarice](http://avarice.sourceforge.net/), an open source bridge between Atmel Jtag/PDI and GDB.

I was less lucky as the 128A4U MCU is not in supported by upstream avarice, but there is an [unmerged patch](https://sourceforge.net/p/avarice/patches/24/) which adds support for the chip.

# Avarice #
Avarice should be built from latest svn version as the snapshot on Sourceforge isn't building, so you

    svn checkout http://svn.code.sf.net/p/avarice/code/trunk

Then the above patch should be applied to `src/devdescr.cc`, for your convenience you can download a formatted .patch file [here]({{ site.url }}/assets/avarice-xmega128a4u.patch). Once done you can build it with

    ./Bootstrap
    ./configure --prefix=/usr
    make

Once the build is done it's finally possible to connect the Atmel board (don't forget to power up the chameleon as well), then run the following command

    sudo ./avarice --edbg --ignore-intr --part atxmega128a4 --pdi :4242

`sudo` is needed as I haven't configured udev rules and otherwise avarice would simply segault

# GDB and VSCode #
You'll now need the avr gdb _flavor_, which should be available in your distribution's repositories under the name `avr-gdb`.

You can now open the `ChameleonMini/Firmware/Chameleon-Mini` folder in VSCode (or modify accordingly the `executable` path specified below) and finally add the debug configuration via the appropriate menu (Debug -> Add Configuration -> GDB). This configuration should work for the avarice command written above and with current ChameleonMini's makefiles output path.

    {
        "version": "0.2.0",
        "configurations": [
            {
                "type": "gdb",
                "request": "attach",
                "name": "ChameleonMini debug",
                "target": ":4242",
                "remote": true,
                "cwd": "${workspaceRoot}",
                "executable": "${workspaceRoot}/Chameleon-Mini.elf",
                "valuesFormatting": "parseText",
                "gdbpath": "avr-gdb"
            }
        ]
    }

Now you should be able to debug the software as if you were using Atmel Studio, but you're not tied to a specific IDE or Operating System. Microsoft's VSCode [documentation](https://code.visualstudio.com/docs/cpp/cpp-debug) explains its capabilites better than I could ever do.

## Post Scriptum
Pay attention to Atmel-ICE pinout if you plan to buy the bare PCB version as I did, [it might not be as obvious as it should](https://www.avrfreaks.net/comment/2548651#comment-2548651).
