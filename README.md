# Linux on Panasonic Let's Note CF-SR3

## Overview

The Panasonic Let's Note CF-SR3 is a lightweight portable computer marketed at the Japanese business sector. As a long overdue update in the line, it boasts some appealing features:
* Modern Alder Lake chipset
* Very light construction
* High quality 12" 3:2 Display (1920 x 1280) "medium DPI" with matte finish, and very thin bezel
* Non-chiclet keyboard
* Excellent connectivity: Ethernet, HDMI, 3 x USB3, 2 x USB-C, SD, 3.5mm audio, and even VGA for compatibility.
* The trackpad has actual buttons despite its odd shape
* Hinges that place the display above, not behind the main body, when engaged
* Classic business-oriented style that will be a polarizing factor

The computer works well under Windows with the vendor crap removed, but I am interested in using it with Linux as well. I will document how well various aspects of the machine work under Debian "Bookworm" Linux as of 2024-03-24.


## Networking

*Status: Good

The Ethernet port (driven by the connected Intel I219-V) worked out of the box, and the WiFi (Intel Alder Lake-P PCH CNVi) worked fine with an install of `firmware-iwlwifi`. 

## Video Driver

*Status: Good; Required Config*
 
As an Alder Lake notebook computer the video chipset is Intel iRIS Xe. While Intel Graphics supposedly works "out of the box" on Linux in general, the xf86-video-intel driver provides an upsetting experience. I strongly recommend the "modesetting" driver; without it, there are enormous performance issues for most hardware accelerated programs, even doing something so innocent as watching a YouTube video or using Discord.

The HDMI and VGA ports work with no issue, with audio and video sinks.

### Configuration

I created the following file to explicitly select the modesetting driver:

```
# /etc/X11/xorg.conf.d/20-modesetting.conf
Section "Device"
   Identifier "card0"
   Driver "modesetting"
   Option "TearFree" "true"
   Option "DRI" "2"
EndSection
```

DRI 2 is set as DRI 3 gave me some odd distortions from time to time. I don't know if it remains necessary now, but I had the following line in `/etc/environment` as well:

```
LIBGL_DRI3_DISABLE=1
```


## Backlight

*Status: Good; Required Config*

While backlight control worked "out of the box" at first, upon switching to the modesetting Intel graphics driver the backlight stopped working (or, at least it no longer is controllable by `xbacklight`). However, `/sys/class/backlight/intel_backlight` is present and I can use it by writing to the `brightness` handle. 

Unbelievably this is considered an open issue: https://gitlab.freedesktop.org/xorg/xserver/-/issues/47

### Configuration

In `/etc/default/grub`, I added the option

```
acpi_backlight=native
```

and then ran `update-grub`. It seemed to work with `none` instead of `native`, for what it's worth.

After that, some udev rules will allow users of the `video` group to control the backlight (cribbed from the gitlab link above):

```
# /etc/udev/rules.d/backlight.rules
ACTION=="add", SUBSYSTEM=="backlight", KERNEL=="intel_backlight", RUN+="/bin/chgrp video /sys/class/backlight/%k/brightness"
ACTION=="add", SUBSYSTEM=="backlight", KERNEL=="intel_backlight", RUN+="/bin/chmod g+w /sys/class/backlight/%k/brightness"

```

Then, the user must be added to the `video` group in order to benefit from the new udev rule:


``` 
$ sudo usermod -aG video <user>
```

As the same user noted, `xbacklight` doesn't seem happy still, so I switched to using `light`. I use Openbox, so I made bindings for `XF86MonBrightnessUp` to `light -A 10`, etc. I imagine users of Gnome, KDE, or whatever will have the keys "just work" at this point.

## Keyboard

*Status: Perfect*

It seems silly to write about something as basic as the keyboard, but these days you see bizarro setups with I2C for the internal keybaord so maybe it stands to be said that the keyboard operates without issue. 

The layout is Japanese, but I generally set it up as a US layout keyboard and know the quirky key locations that matter (for example, the tilde (~) is typed by pressing the 半角\全角 key in the top-left.

## Trackpad

*Status: Perfect*

The trackpad is a modern affair with multi-finger input support and all that. It provides nice precision when scrolling and is responsive. The two dedicated mouse buttons, of course, do their job, and may be chorded for the third button.

### Suggested Configuration: 

Trackpad drivers these days often come configured such that they disable input for a short period whenever typing is detected; I suspect this is a result of oversized trackpads that are easy to hit with your thumbs. I can't stand this 'feature', so I created the following configuration file:

```
# /etc/X11/xorg.conf.d/40-libinput.conf
Section "InputClass"
	Identifier "libinput touchpad"
	MatchIsTouchpad "on"
	MatchDevicePath "/dev/input/event*"
	Driver "libinput"
	Option "DisableWhileTyping" "False"
EndSection
```


### Suggested Configuration Bonus: Firefox XInput2 for Precise Scrolling

I wanted to make sure Firefox will always take advantage of the precise scrolling, rather than classic mouse wheel click emulation. I added the following in `/etc/environment`; why this is not a default in Firefox is beyond me.

```
MOZ_USE_XINPUT2=1
```

## Fan Control

*Status: Poor - Always Running *

Without configuration the fan just assumes a relatively high speed with no variation based upon load. This is annoying and wasteful.

I found a device that looks germane to our situation at `/sys/devices/LNXSYSTM:00/LNXSYBUS:00/PNP0A08:00/device:08/PNP0C09:00/INTC1048:00`. Within there is the subdirectory "power", and a file in there reports that the control mode is `auto`. Maybe the fan controller defaults to this speed as a safety matter when the OS has not kicked in and given it a response curve; the startup firmware also has the fan running like this ordinarily until Windows starts.

## Audio

*Status: Working - Infrequent Caveat*

The sound card uses `sof-hda-dsp`. Everything "just worked" not long ago, but one day I only got audio from my headphones while the speakers refused to do anything. Another day after using Windows for a bit and booting back into Linux, the speakers were fine.

For the sake of diagnostics I installed `pulseaudio` and queried sinks using `pacmd list-sinks`. The first three outputs are for HDMI output and were appropriately inactive, while the last one is for Speakers + Headphone. It correctly identified that the speaker output ought to be active and registered volume at the full setting I'd put it at.

I have a theory that there is an amplifier IC for the speakers that was just not being spoken to, or was left in some undesirable register configuration that wasn't being reset, so while audio is leaving the chipset just fine it was just not being amplified. Perhaps the current behavior where it *generally* behaves most of the time is the IC operating on register defaults on reset.

HDMI audio worked with no issues. I have not tried a display over USB-C.

The built-in microphone also worked, showing up as `sof-hda-dsp; (hw:0,7)`. Other devices gave static or nothing, so I would recommend using 0,7 as the default.

## SD Card

*Status: Good*

The SD card works fine for me with no particular configuration.

## Webcam

*Status: Good*

The built-in webcam worked with no configuration. It's fine, middle of the road, absolutely clear and fast enough for a video chat or whatever.
