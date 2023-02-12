---
layout: post
title:  "Pop!_OS on Razer Blade Stealth (i7-8550U)"
date:   2018-01-14 13:33:17 -0600
categories: pop_os linux razer
---

Getting Pop!_OS running on the new Razer Blade Stealth is a little bit challenging, but through a lot of trial and spelunking through half through what seemed like half the forums on the internets I have a combination of options that work for me.

## Install

Burn the Pop!_OS ISO to a USB drive and boot.

I tripped over two things.


- Connect to your WIFI network before you start the install.
- When you get the warning about `Force UEFI Instalation` choose `Continue in UEFI mode`.

That's it.

## First boot

Was a bit of a mess. I chose full disk encryption so I was greeted by the Plymouth password prompt, except the screen was flickering and tearing all over the place.

Not a great start :(

Just put in your password and get to the login screen. That too will probably be blinking all over the place, but hang in there. Log in, open the settings and reduce the screen resolution. I'm running at 1920x1080. This will stabilize things so apply some fixes.

Once you have the kernel params in place you can run at native resolution.

## Fixes

### Grub Kernel Options

Once you're in, add some kernel parameters.

Update the `GRUB_CMDLINE_LINUX_DEFAULT` line in `/etc/defaults/grub`. Add the following Kernel/Module options and run `update-grub` to make them active on reboot.

- `pcie_aspm=off` - Disable Some pcie Power Management
- `i915.enable_rc6=0` - Disable Some GPU Power Management, Stops the flickering.
- `button.lid_init_state=open` - This will keep the system from freaking out after you wake it from sleep.


Edit /etc/defaults/grub with your favorite editor.

```
GRUB_CMDLINE_LINUX_DEFAULT="pcie_aspm=off button.lid_init_state=open i915.enable_rc6=0 quiet splash"
```

Run `update-grub` so the changes will be active on reboot

```
sudo update-grub
```

Reboot the system

```
sudo reboot
```

### Update Intel Drivers

This may not be entirely necessary, but in the troubleshooting I updated the Intel Drivers with the <a href="https://01.org/linuxgraphics/downloads/intel-graphics-update-tool-linux-os-v2.0.5">intel-graphics-update-tool</a>.

Install the 64bit .deb package.

Before you run the tool you need to make some temporary changes to `/etc/lsb-release` since this stupid tool "only supports Ubuntu 17.04"

Make a backup of the current `lsb-release`

```
sudo cp /etc/lsb-release ./lsb-release
```

Edit /etc/lsb-release and make it "look" like a Ubuntu 17.04 release.

```
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=17.04
DISTRIB_CODENAME=zesty
DISTRIB_DESCRIPTION="Pop!_OS 17.10 (Artful Aardvark)"
```

Run `intel-graphics-update-tool` and follow the prompts.

```
sudo intel-graphics-update-tool
```

When you're done restore the `lsb-release` file.

```
sudo cp ./lsb-release /etc/lsb-release
```

### Add xorg.conf file.

This also may be optional. I added an xorg config with some `intel` driver options.

Create `/usr/share/X11/xorg.conf.d/20-intel.conf`

```
Section "Device"
    Identifier "Intel Graphics"
    Driver "intel"
    Option "TearFree"
EndSection
```

## Things that are still weird

Sometimes when I plug in the HDMI for my external monitor, the laptop will detect the screen, and the monitor will come out of sleep, but no image will be shown. Just stays black. Sometimes re-plugging works. Rebooting with the HDMI plugged in always works.
