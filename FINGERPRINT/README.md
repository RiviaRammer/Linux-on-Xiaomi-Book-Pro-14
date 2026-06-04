# Goodix 27c6:6890 Fingerprint on Linux

This document describes an experimental workaround for enabling the Goodix
fingerprint reader used on the Xiaomi Book Pro 14 / TM2424 on Linux.

The device tested here is:

```text
USB VID:PID: 27c6:6890
Vendor:      Shenzhen Goodix Technology Co.,Ltd.
Product:     Goodix USB2.0 MISC
Type:        Goodix MOC fingerprint device
```

The Windows driver package identifies the device as a Goodix MOC fingerprint
reader and supports both:

```text
USB\VID_27C6&PID_6890
USB\VID_27C6&PID_689A
```

## Status and Time Sensitivity

This workaround is time-sensitive.

At the time this was tested, the installed Ubuntu/Kubuntu packages contained
the `goodixmoc` libfprint driver, but the system did not expose `27c6:6890` as
a supported fingerprint reader. The same driver stack already listed nearby
Goodix MOC devices, including `27c6:689a`.

Because upstream `libfprint` and distribution packages change over time, this
manual patch may become unnecessary. Before patching anything, first check
whether your current system already supports the device. Waiting for a newer
system update may be the easier and safer option.

## Check Whether It Already Works

Install the standard fingerprint packages:

```bash
sudo apt install fprintd libpam-fprintd
```

Check the USB device:

```bash
lsusb | grep -iE '27c6|goodix|finger'
```

Expected device:

```text
ID 27c6:6890 Shenzhen Goodix Technology Co.,Ltd. Goodix USB2.0 MISC
```

Check whether `fprintd` can see it:

```bash
fprintd-list "$USER"
fprintd-enroll
```

If you get:

```text
No devices available
```

then your current `libfprint/fprintd` stack does not recognize this device yet.

Check package versions:

```bash
apt policy libfprint-2-2 libfprint-2-tod1 fprintd
```

The tested environment was:

```text
libfprint-2-2:    1:1.95.1+tod1-0ubuntu1
libfprint-2-tod1: 1:1.95.1+tod1-0ubuntu1
fprintd:          1.94.5-4
```

Check whether the installed library contains the Goodix MOC driver:

```bash
strings /usr/lib/x86_64-linux-gnu/libfprint-2.so.2 | grep -iE 'goodix|6890|689a'
```

Useful signs:

```text
FpiDeviceGoodixMoc
Goodix MOC Fingerprint Sensor
libfprint-goodixmoc
```

If the driver exists but `27c6:6890` is not listed, the device may need to be
added to the Goodix MOC USB ID table.

## Local hwdb Test

Adding a local hwdb entry is low risk, but in the tested case it was not enough
because the device ID was missing from the libfprint driver match table.

Optional test:

```bash
sudo tee /etc/udev/hwdb.d/61-goodix-6890-fingerprint.hwdb >/dev/null <<'EOF'
usb:v27C6p6890*
 ID_AUTOSUSPEND=1
 ID_PERSIST=0
EOF

sudo systemd-hwdb update
sudo udevadm trigger -s usb
sudo systemctl restart fprintd
fprintd-enroll
```

If it still says `No devices available`, remove the test file:

```bash
sudo rm /etc/udev/hwdb.d/61-goodix-6890-fingerprint.hwdb
sudo systemd-hwdb update
sudo udevadm trigger -s usb
```

## Rebuild libfprint With 27c6:6890 Added

Enable source packages first if `apt source libfprint` fails.

On systems using the newer `.sources` format, inspect the source file:

```bash
grep -R --line-number -E '^(Types:|URIs:|Suites:|Components:|deb |deb-src )' \
  /etc/apt/sources.list \
  /etc/apt/sources.list.d/* 2>/dev/null
```

If your Ubuntu source file contains:

```text
Types: deb
```

change it to:

```text
Types: deb deb-src
```

Then update and fetch the source:

```bash
sudo apt update
apt source libfprint
```

Install build dependencies:

```bash
sudo apt build-dep libfprint
sudo apt install build-essential devscripts dpkg-dev
```

Enter the source tree:

```bash
cd libfprint-*
```

Find the Goodix MOC device table:

```bash
grep -RniE '689a|goodixmoc|27c6' libfprint drivers data debian 2>/dev/null
```

In the Goodix MOC driver source, add `27c6:6890` next to the existing
`27c6:689a` entry. In the tested package this was in the Goodix MOC driver
area, with strings referring to:

```text
libfprint/drivers/goodixmoc/goodix.c
```

If the package has fingerprint autosuspend hwdb data, also add:

```text
usb:v27C6p6890*
```

near the existing:

```text
usb:v27C6p689A*
```

Build binary packages:

```bash
DEB_BUILD_OPTIONS=nocheck dpkg-buildpackage -us -uc -b
```

The `.deb` files will be generated in the parent directory.

Install the rebuilt packages:

```bash
cd ..

sudo apt install ./libfprint-2-2_*_amd64.deb \
  ./libfprint-2-tod1_*_amd64.deb
```

Reload hardware data and restart the daemon:

```bash
sudo systemd-hwdb update
sudo udevadm trigger -s usb
sudo systemctl restart fprintd
```

Test enrollment:

```bash
sudo fprintd-enroll "$USER"
```

In the tested case, after adding `27c6:6890`, the result changed from:

```text
No devices available
```

to:

```text
Using device /net/reactivated/Fprint/Device/0
Enrolling right-index-finger finger.
Enroll result: enroll-stage-passed
```

This confirms that the device is no longer merely visible over USB; it is being
used by `fprintd`.

## Permissions and Desktop Integration

If normal user commands fail with a Polkit error but `sudo` works, the driver is
probably working and the remaining issue is authorization:

```text
PermissionDenied: Not Authorized
```

Try:

```bash
sudo fprintd-verify "$USER"
```

To use fingerprints for PAM authentication, run:

```bash
sudo pam-auth-update
```

Enable:

```text
Fingerprint authentication
```

Then test:

```bash
sudo -k
sudo true
```

Kubuntu uses SDDM. Fingerprint support for login, lock screen, sudo, and KDE
authorization dialogs may vary by release and configuration. It is common for
fingerprint authentication to work for `sudo` or lock screen unlock while the
initial graphical login still requires a password.

## Prevent Package Updates From Overwriting the Patch

If the patched packages work, hold them:

```bash
sudo apt-mark hold libfprint-2-2 libfprint-2-tod1
```

When your distribution later ships native support for `27c6:6890`, unhold and
return to official packages:

```bash
sudo apt-mark unhold libfprint-2-2 libfprint-2-tod1
sudo apt install --reinstall libfprint-2-2 libfprint-2-tod1
```

## Notes

- This is not a Windows driver port.
- The Windows package is useful only for identifying the hardware ID and device
  family.
- The working path is through Linux `libfprint` and `fprintd`.
- Adding a USB ID can make the driver try the device, but it does not guarantee
  full compatibility on every firmware or hardware revision.
- Prefer official upstream or distribution support when available.
