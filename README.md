# Linux on Xiaomi Book Pro 14

This repository contains a record of all the steps required to get the Xiaomi Book Pro 14 laptop working with Linux.

While issues (and solutions) can be discussed and tracked in this repository, its purpose is not to provide code, but rather to organize things and provide the necessary information needed to run Linux on this device.


## Status

### What works (with tweaks)

- :heavy_check_mark: WiFi
- :heavy_check_mark: Keyboard (dumb mode)
- :heavy_check_mark: Touchpad
- :heavy_check_mark: Audio
- :heavy_check_mark: Bluetooth
- :heavy_check_mark: Touchscreen
- :heavy_check_mark: Native screen resolution (disable PSR)
- :heavy_check_mark: Power button
- :heavy_check_mark: Lid switch

### What doesn't work

- :x: Keyboard (no dumb mode)
- :x: LED on Caps-Lock
- :x: Mute key
- :x: Keyboard backlight key

### What is uncertain

- :question: USB-C (:heavy_check_mark: screen, ethernet, hid, etc…)
- :question: Thunderbolt 4 (:heavy_check_mark: screen, ethernet, hid, etc…)
- :question: Fan control
- :question: Suspend/Resume
- :question: Fingerprint Reader
- :question: Fan speed
- :question: Respect battery tresholds

### Tested Linux Distributions

#### Kubuntu 26.04

Kernel 7.0.0-14-generic
KDE 6.24.0

## Quickstart

### 1. Keyboard

Add following boot parameter to the kernel commandline to fix keyboard:

```
i8042.dumbkbd
```

### 2. Screen

Add following boot parameter to the kernel commandline to fix screen:

```
i915.enable_psr=0
```


## Machine Infos

Specs: [mi.com](https://www.mi.com/global/product/xiaomi-book-14/specs/)

Manufacturer: XIAOMI<br>




