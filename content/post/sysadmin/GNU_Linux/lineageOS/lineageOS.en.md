+++
author = "Adur"
title = "LineageOS Installation"
date = "2021-10-21"
description = ""
featured = true
tags = [
    "lineageOS",
    "android"
]
categories = [
	"Infrastructure",
    "Self-Hosting",
]
series = ["F-Droid"]
aliases = [""]
thumbnail = "images/f-droid.png"
toc = true
draft = false
+++

## What is LineageOS?

## Installing LineageOS on Xiaomi Mi A1

### Requirements

Materials used in this scenario:
- **Xiaomi Mi A1**
- **LineageOS16 official (nightly=development)**
- **[LineageOS17 Unofficial](https://sourceforge.net/projects/test-tissot/files/Lineage/)**
- [TWRP Recovery](https://eu.dl.twrp.me/tissot/)

### Downloads

Download and install the Android development files (SDK platform) adb and fastboot (in my case: minimal_adb_fastboot_v1.4.3).

### Enable OEM Unlocking

Enable OEM unlocking and USB debugging in the phone settings.

### Bootloader Unlock

Use the following commands to unlock the bootloader. Run them from the adb and fastboot folder:
```zsh
adb devices
# Reboot the phone into fastboot mode
adb reboot bootloader
# Check the device status
fastboot oem device-info
# Unlock the bootloader
fastboot oem unlock
# Reboot the phone
```
Verify that it has been unlocked successfully with:
```zsh
adb devices
adb reboot bootloader
fastboot oem device-info
```

### Recovery Mode Installation

After enabling USB debugging mode and unlocking the OEM, install recovery mode using the .img file in the same folder as adb and fastboot:

```zsh
fastboot flash recovery twrp-installer-3.3.1-0-tissot.img
```

The following error occurs:

```
	target reported max download size of 534773760 bytes
	sending 'recovery' (29348 KB)...
	OKAY [  0.683s]
	writing 'recovery'...
	FAILED (remote: partition table doesn't exist)
	finished. total time: 0.699s
```

The solution is:

```zsh
fastboot flash boot twrp-3.3.1-0-tissot.img
```

The following result is obtained:
```
RESULT:
	fastboot flash boot twrp-3.3.1-0-tissot.img
	target reported max download size of 534773760 bytes
	sending 'boot' (29348 KB)...
	OKAY [  0.679s]
	writing 'boot'...
	OKAY [  0.181s]
	finished. total time: 0.860s
```

### Image Installation

Reboot into recovery mode with `power button + volume up` and follow these steps:

1. Wipe -> Format Data
2. Advanced Wipe -> Dalvik, system and data
3. Install -> lineage-1-...-tissot.zip
4. Wipe Dalvik and reboot system.

Once completed, LineageOS is installed. By default, Google Play is not included. If you encounter any errors, you can install the minimal version of GApps, for example `open_gapps-arm-9.0-pico-20200408.zip`.