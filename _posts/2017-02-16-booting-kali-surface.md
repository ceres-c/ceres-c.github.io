---
layout: post
title: Booting Kali Linux on a Surface Pro 4
excerpt: "or: How I Learned to Stop Worrying and Love Linux.<br>If only it was easier to boot it on this thing..."
tags: [Kali, SP4, Surface]
image:
  feature: booting-kali/refindkali.jpg
---

Generally, running a Linux distro on your PC is a straightforward process, you put the iso in the thumb drive, set it as default boot device and you’re ready to rock. Generally.

In another universe lives a happy student with his Surface. He really loves his tablet-PC hybrid. It’s lightweight, it’s powerful, it has got pen support (which is nice to take notes during lectures) and good battery life. He was a happy camper. Until he wanted to try and play with NFC tags. Libnfc is great, mfcuk and mfoc are good at their work, but they do not work with Windows. Our student could have tried Cygwin, but at that moment he didn’t have at home enough vodka. VMs could have been another option, but sadly libusb does not like with virtualization (at least in his limited experience).

So, he decided that installing a proper Linux environment was the best choice. "How hard could that be?" Fool.

As I said the Surface is a great device, but *being good has its <s>perks</s> downsides*. It’s a device with highly customized hardware and, apparently, Linux does not play nice with it. At least you can boot Ubuntu straight from UEFI using nothing but a standard live USB and a left swipe. It’s not **that** easy with Kali, or at least it hasn’t been for me, maybe you’re luckier than I was and everything works fine in the first place.

To finally boot Kali (let alone install it and make the Type Cover work, it’s going to be another chapter I still have to test on) I had to install rEFInd and then fine tune a USB stick prepared with rufus.

I am writing this post to make things easier for someone with the same intention, so let’s start with the guide itself.

**<font size="6">TOC</font>**

* TOC
{:toc}


# rEFInd #
This great piece of software is the first step into our headache. It is a boot manager (not a bootloader) developed for EFI/UEFI systems (from now on EFI will suffice) to make out life a little less miserable.

## Downloading ##
You can download rEFInd from the [official website][1], go for the binary archive as we'll have to install it manually. Extract the content in a folder of your choice.

## Installing ##
The install procedure is well documented on the [apposite page][2], I advice you to read it and understand the steps before going further.

>Please note <span style="color: red">I won't be responsible</span> for any damage to your precious device, be sure to know what you're doing!
>Make a backup of your EFI partition, even with [EasyUEFI][3], it might save your ass!

First of all you need to disable secure boot in the UEFI screen. Press and hold Power+Volume UP keys while booting your device and, in the Security tab, Change configuration and then select None.

This is the TL;DR version of that page.

Run terminal as admin and:

    cd <your refind-VersionNO folder path>
	mountvol S: /S
	xcopy /E refind S:\EFI\refind\
	S:
	cd EFI\refind
	rename refind.conf-sample refind.conf
	bcdedit /set {bootmgr} path \EFI\refind\refind_x64.efi
	bcdedit /set {bootmgr} description "rEFInd boot manager"
	
No, {bootmgr} is not a variable, you have to input that exact string.

We're all set. rEFInd is installed now. After a reboot you should be welcomed by something similar to the following, but with only a single entry: Windows.
<figure><a href="{{ site.url }}/images/booting-kali/refind.png"><img src="{{ site.url }}/images/booting-kali/refind.png" alt="rEFInd sample screen"></a></figure>

## Theming ##
Yeah, I know it's not important, but it's still nice to have something which goes along well with the rest of the environment. You can install a theme for rEFInd made especially for surface devices, which mimics Microsoft design choices.

The theme has been shared with this [reddit post][4] but I've archived the rar file [locally][5]. Once you've extracted the archive, run terminal as admin and:

    cd <theme extraction path>
	mountvol S: /S
	xcopy /E surface-theme S:\EFI\refind\surface-theme

## Configuring ##
We now want to edit rEFInd configuration in order to make it more convenient. My main OS is still going to be windows, even after I'll have installed Kali (please, don't hate me Linux fans), so I want my tablet to boot automatically in Windows if I don't press any key in a couple of seconds after the OS selection screen appears. To do so, we have to edit rEFInd configuration file, refind.conf.

Boot in windows, run another terminal as admin and:

    mountvol S: /S
	S:
	cd EFI\refind
	refind.conf

You'll be asked to choose the default application to open .conf file if it is not already assigned to a program in your computer. Even notepad could do, but why not notepad++? If you don't have it already installed it might be the right time to get it!

Now, once we're able to edit the file, you should:

1. Edit the number next to "timeout" from 20 to 5
2. Uncomment the line "default_selection Microsoft" (remove the hash)
3. Add "include surface-theme/theme.conf" in the includes section of this file

Then save and close this file. Again, in the same terminal window as before:

	cd surface-theme
	theme.conf

Here you should change "resolution 2160 1440" to "resolution 2736 1820", otherwise you'd get an error from rEFInd.

Once you've saved this file we're ready for the next part.

# Install medium #

## Creation ##
It sounds a bit pompous, doesn't it?

To create an install device you simply have to download the ISO from Kali website and use rufus (default settings are ok) to make a bootable USB. "Iso image mode" worked for me, so it should do for you too.

--Note--
Rufus up to 2.12 is working with default settings, I haven't been able to make newer version work but I haven't even tried, actually...

## Get it to work ##
The USB made with rufus won't be recognised by rEFInd.

1. Create a folder

       /EFI/boot

    in your install medium.

2. Download from [fedora's archives][6] these 2 files:

       BOOTX64.EFI
       grubx64.efi
	
	and place them in the boot folder we've just made.
	
3. Create a file grub.cfg in the same folder with this content

       # Config file for GRUB2 - The GNU GRand Unified Bootloader
       # /boot/grub/grub.cfg

       # DEVICE NAME CONVERSIONS
       #
       # Linux Grub
       # -------------------------
       # /dev/fd0 (fd0)
       # /dev/sda (hd0)
       # /dev/sdb2 (hd1,2)
       # /dev/sda3 (hd0,3)
       #
       # root=UUID=dc08e5b0-e704-4573-b3f2-cfe41b73e62b persistent

       set menu_color_normal=yellow/blue
       set menu_color_highlight=blue/yellow

       function load_video {
       insmod efi_gop
       insmod efi_uga
       insmod video_bochs
       insmod video_cirrus
       insmod all_video
       }

       load_video
       set gfxpayload=keep

       # Timeout for menu
       set timeout=5

       # Set default boot entry as Entry 0
       set default=0
       set color_normal=yellow/blue

       menuentry "Kali - Live Non-persistent" {
       set root=(hd0,1)
       linuxefi /live/vmlinuz boot=live noconfig=sudo username=root hostname=kali
       initrdefi /live/initrd.img
       }

       menuentry "Kali - Live Persistent" {
       set root=(hd0,1)
       linuxefi /live/vmlinuz boot=live noconfig=sudo username=root hostname=kali persistence
       initrdefi /live/initrd.img
       }

       menuentry "Kali Failsafe" {
       set root=(hd0,1)
       linuxefi /live/vmlinuz boot=live config memtest noapic noapm nodma nomce nolapic nomodeset nosmp nosplash vga=normal
       initrdefi /live/initrd.img
       }

       menuentry "Kali Forensics - No Drive or Swap Mount" {
       set root=(hd0,1)
       linuxefi /live/vmlinuz boot=live noconfig=sudo username=root hostname=kali noswap noautomount
       initrdefi /live/initrd.img
       }

       menuentry "Kali Graphical Install" {
       set root=(hd0,1)
       linuxefi /install/gtk/vmlinuz video=vesa:ywrap,mtrr vga=788
       initrdefi /install/gtk/initrd.gz
       }

       menuentry "Kali Text Install" {
       set root=(hd0,1)
       linuxefi /install/vmlinuz video=vesa:ywrap,mtrr vga=788
       initrdefi /install/initrd.gz
       }

	Or download it from [here][7]

We're done now! Your drive should be recognized by rEFInd when you plug it in the USB port or even through a USB hub, which was incompatible with the official boot manager. It simply hang, which made it impossible to install the OS because the installer did not have Type Cover support.

# Conclusion #
I'd like to thank John Ramsden for [his guide][8] about installing Arch on a SP4, and because he pointed me in the right direction suggesting to give rEFInd a try.

Also, thanks to Rnihton for [his guide][9] on Kali forums. Yes, I know he called it "Beginner Way", but it works. I'm not ashamed.

Thanks to frebib, the creator of the [Surface Theme for rEFInd][4].

And, for last, I'd like to thank you for making it to the end of this guide, ignoring my *not-so-perfect English* (what a nice euphemism).

<br><br>
<p style="text-align: right"><font size="2">PS Sorry for that introduction, I won’t do that again, I promise.</font></p>


[1]: http://www.rodsbooks.com/refind/getting.html
[2]: http://www.rodsbooks.com/refind/installing.html#windows
[3]: http://www.easyuefi.com/faq/en_US/Backup-EFI-System-Partition.html
[4]: https://www.reddit.com/r/SurfaceLinux/comments/2wm1ad/surface_bootloader_theme_domnload_in_comments/
[5]: {{ site.url }}/assets/kali_boot_files/SurfaceThemerEFInd.zip
[6]: https://archives.fedoraproject.org/pub/archive/fedora/linux/releases/25/Everything/x86_64/os/EFI/BOOT/
[7]: {{ site.url }}/assets/kali_boot_files/grub.cfg
[8]: https://ramsdenj.com/2016/08/29/arch-linux-on-the-surface-pro-4.html
[9]: https://forums.kali.org/showthread.php?21641-How-to-EFI-install-Kali-Linux-(Beginner-Ways)
[10]: https://www.reddit.com/r/SurfaceLinux/comments/2wm1ad/surface_bootloader_theme_domnload_in_comments/
