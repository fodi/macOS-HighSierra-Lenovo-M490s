
# macOS on Lenovo M490s

This repository contains the EFI partition contents needed to install and run macOS 10.13 High Sierra on a Lenovo M490s laptop. There's also a short guide below on how to do it.

## Used hardware configuration

- Lenovo Lenovo M490s - UEFI BIOS version: HBET20WW(1.04)
  - Intel i3-2375M Sandy Bridge 2nd gen CPU
  - Intel HM77 (7 Series) chipset
  - 4GB RAM (a single module)
  - 120GB Kingston SMS200S3120G M.2 SSD
  - Intel Centrino Wireless-N 2230 (which is not going to work, and since Lenovo has a restrictive BIOS device whitelist on this device, replacement is probably not possible without a BIOS mod)
  - Realtek ALC269VC audio
  - Intel HD Graphics 3000
  - nVidia GeForce 710M (which is not going to work and should be disabled in the BIOS, Optimus is not supported by macOS)
  - Integrated camera
  - Keyboard and touchpad
  - SD card reader (untested)
  - Fingerprint scanner (untested)

## Guide

Mostly by following the instructions at [https://internet-install.gitbook.io](https://internet-install.gitbook.io) I was able to prepare the macOS 10.13 Internet Installer on a 8GB Kingston flash drive. I used Ubuntu for the "Preparing your installer media" part, downloading the recovery image `SecUpd2019-002HighSierra.RecoveryHDUpdate.pkg` using the amazing gibMacOS script.

During the Clover preparation part I've added the following kexts to `/EFI/CLOVER/kexts/Other`: `Lilu.kext` \+ `AppleALC.kext` (for audio support, although that didn't work) + `WhateverGreen.kext` (Lilu plugin for GPU support), `USBInjectAll.kext` \+ `FakePCIID.kext` \+ `FakePCIID_XHCIMux.kext` (for USB XHCI support, because the laptop is 2nd gen), `RealtekRTL8111.kext` (for Realtek Gigabit Ethernet support), `VoodooPS2Controller.kext` (for the integrated touchpad support) and of course the essential `VirtualSMC.kext`.

I've used the following Clover `config.plist` file (I didn't have to make any changes to it):
[https://github.com/RehabMan/OS-X-Clover-Laptop-Config/blob/master/config\_HD3000\_1366x768\_7series.plist](https://github.com/RehabMan/OS-X-Clover-Laptop-Config/blob/master/config_HD3000_1366x768_7series.plist)

Clover and the installer started up fine, then I've formatted my M.2 SSD using "`diskutil eraseDisk JHFS+` (...)" in Terminal and proceeded with the installation. After reboot, the installation resumed and I was soon faced with the error message "macOS could not be installed on your computer", "Couldn't communicate with a helper application. Quit the installer to restart your computer and try again". After some googling it turned out that this error occured because the High Sierra installer tried converting the JHFS+ filesystem to APFS on my SSD and this somehow led to this problem I've found a workaround here:
[https://www.tonymacx86.com/threads/guide-avoid-apfs-conversion-on-high-sierra-update-or-fresh-install.232855/#post-1602625](https://www.tonymacx86.com/threads/guide-avoid-apfs-conversion-on-high-sierra-update-or-fresh-install.232855/#post-1602625)

I'm quoting the relevant part:

>Even though you create a new HFS+J partition, if the target is an SSD, the installer will still convert it to APFS.  
>  
>To avoid that, after running the installer, and upon the first reboot where you would be normally directing Clover to boot the next stage of the installer by selecting "Boot macOS Install from ...", instead, boot the "install\_osx" partition on USB again. When that is finished booting, choose Terminal from the Utilities menu.  
>  
>`# list /Volumes to remind yourself of the name you gave it`  
>`ls -l /Volumes`  
>`# then change your working directory to it (in my case, I used '1013')`  
>`cd /Volumes/1013`  
>`# now change to the "macOS Install Data" directory`  
>`cd "macOS Install Data"`  
>`# now fix the minstallconfig.xml with PlistBuddy`  
>`/usr/libexec/PlistBuddy -c "Set :ConvertToAPFS false" minstallconfig.xml`  
>  
>That's it! Now you're ready to quit Terminal, reboot, and continue the installation process by booting the "Boot macOS Install from ..." partition. When you're done, you'll have a fresh install on HFS+J instead of APFS.

I had to restart installation and perform the above after the first stage. That did the trick and I've finally reached the macOS desktop. The display, keyboard, touchpad and the Ethernet LAN was working thanks to the kexts, but there was no audio.

At this point I've copied the Clover files from the USB flash drive to my SSD's 200MB EFI partition (after deleting the existing EFI folder from it). Some more Terminal action:

`# create a folder for mounting the EFI partition`
`sudo mkdir /Volumes/EFI`
`# mount EFI partition`
`sudo mount -t msdos /dev/disk0s1 /Volumes/EFI`
`# go to that folder`
`cd /Volumes/EFI`
`# delete existing EFI folder and its contents`
`sudo rm -rf EFI`

After that I used Finder to copy the `EFI` folder from my USB drive to `/Volumes/EFI`. After rebooting, Clover started up from the SSD and I was able to boot to the macOS desktop. After that I've started looking into the sound problem - the easiest solution turned out to be `VoodooHDA.kext`, which I've grabbed from here:
[https://sourceforge.net/projects/voodoohda/](https://sourceforge.net/projects/voodoohda/)

I've copied it to the `/EFI/CLOVER/kexts/Other` folder on the EFI partition (and also removed `AppleALC.kext`) and after reboot audio worked like a charm, even the keyboard volume control using Fn + Left / Fn + Right - yey!

Since the battery in my laptop has been dead for years (it's not even installed anymore), I'm not sure about that part working right; and since the stock wifi chip is from Intel, there's probably no chance to get it working (and replacing is probably out of the question since I've bought a different Intel wireless card years ago and then learnt the bitter lesson that is the existence of BIOS device/vendor whitelisting - cheers Lenovo. If a different Intel chip was denied, I don't think a different vendor would be allowed either). But then again I only wanted to have macOS on this machine for building the iOS versions of Cordova / Flutter apps in Xcode, so I'm very happy with the current state of it.

## Enormous thanks to

- The [Clover EFI bootloader](https://sourceforge.net/projects/cloverefiboot/) team
- The [VoodooHDA](https://sourceforge.net/projects/voodoohda/) team
- [Acidanthera](https://github.com/acidanthera/)
- [RehabMan](https://github.com/rehabMan)
- The good people at [/r/hackintosh](https://www.reddit.com/r/hackintosh/) (especially for the amazing [Internet Install](https://internet-install.gitbook.io/macos-internet-install/) guide)
- The good people at [tonymacx86.com](https://www.tonymacx86.com/)