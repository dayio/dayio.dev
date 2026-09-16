+++
date = '2026-09-16T09:14:00+02:00'
draft = false
title = 'Fixing Dual-Boot Issues with Bluetooth Devices'
description = "How to share Bluetooth link keys between Linux and Windows in a dual-boot setup to avoid re-pairing your devices on every reboot."
tags = ['linux', 'hardware', 'windows', 'dualboot']
+++

On my main workstation, I primarily use Fedora, but I also boot into Windows for gaming or specialized software that does not run natively on Unix-like systems.

One issue that bothered me a lot was that whenever I switched operating systems, I constantly had to re-pair my Bluetooth headphones and peripherals, even though I was using the exact same physical Bluetooth adapter.

After some research, I found out why, when you pair a Bluetooth device, a unique cryptographic link key is negotiated and stored locally on both the computer and the peripheral. Pairing again on the other OS generates a new key, overwriting the peripheral's internal record and invalidating the connection stored on the first OS.

The solution is straightforward, pair the device on both systems, extract the generated link key from the Windows registry, and inject it into Linux. You don't even need to reboot into Windows to fetch the key if you can access your Windows NTFS partition from Fedora.

### 1. Identify Bluetooth MAC Addresses

First, find the MAC address of your Bluetooth adapter:

```shell
➜  ~ sudo ls /var/lib/bluetooth                  
04:7B:CB:2B:48:99  80:13:16:1C:99:99  mesh
```

Then, identify the MAC address of the peripheral you want to configure using the adapter's MAC address:

```shell
➜  ~ sudo find /var/lib/bluetooth/80:13:16:1C:99:99/ -name info -exec grep -H -i "name=" {} +
/var/lib/bluetooth/80:13:16:1C:99:99/F8:4E:17:73:CE:99/info:Name=LinkBuds S
/var/lib/bluetooth/80:13:16:1C:99:99/80:99:E7:E0:04:99/info:Name=WH-1000XM4 <---- This one for me
```

### 2. Extract the Link Key from the Windows Registry

Now comes the fun part, extracting the link key from the Windows registry. Navigate to the Windows `config` directory on your mounted NTFS partition:

```shell
cd /path/to/windows/mount/Windows/System32/config
```

Using `chntpw` (installable via `sudo dnf install chntpw` on Fedora or `sudo apt install chntpw` on Debian/Ubuntu), open an interactive shell in the Windows registry `SYSTEM` hive:

```shell
chntpw -e SYSTEM
```

Navigate to `ControlSet001\Services\BTHPORT\Parameters\Keys` and locate your Bluetooth adapter:

```shell
➜  config chntpw -e SYSTEM                                                      

chntpw version 1.00 140201, (c) Petter N Hagen
openHive(SYSTEM) failed: Read-only file system, trying read-only
Hive <SYSTEM> name (from header): <SYSTEM>
ROOT KEY at offset: 0x001020 * Subkey indexing type is: 686c <lh>
File size 24903680 [17c0000] bytes, containing 5335 pages (+ 1 headerpage)
Used for data: 391956/23918888 blocks/bytes, unused: 278/625656 blocks/bytes.

Simple registry editor. ? for help.

> cd ControlSet001\Services\BTHPORT\Parameters\Keys

(...)\Services\BTHPORT\Parameters\Keys> ls
Node has 2 subkeys and 0 values
  key name
  <047bcb2b4899>
  <8013161c9999> <---------------- This is the one !

(...)\Services\BTHPORT\Parameters\Keys> 
```

`cd` into your adapter directory and display the hexadecimal link key:

```shell
(...)\Services\BTHPORT\Parameters\Keys> cd 8013161c9999                                  

(...)\BTHPORT\Parameters\Keys\8013161c9999> ls
Node has 0 subkeys and 2 values
  size     type              value name             [value if type DWORD]
    16  3 REG_BINARY         <f84e1773ce99>
    16  3 REG_BINARY         <8099e7e00499>

(...)\BTHPORT\Parameters\Keys\8013161c9999> hex 8099e7e00499
Value <8099e7e00499> of type REG_BINARY (3), data length 16 [0x10]
:00000  61 84 C7 9E 62 99 99 48 99 99 99 99 8B E5 00 7E a...b.xH.m.....~


(...)\BTHPORT\Parameters\Keys\8013161c9999> 
```

Copy the 16-byte (32 hexadecimal characters) key and remove all spaces so that `61 84 C7 9E 62 99 99 48 99 99 99 99 8B E5 00 7E` becomes `6184C79E62999948999999998BE5007E`.

### 3. Inject the Link Key into Linux

With the key in hand, edit the device configuration file in Linux at `/var/lib/bluetooth/<ADAPTER_MAC>/<DEVICE_MAC>/info`.

For example, with adapter MAC `80:13:16:1C:99:99` and device MAC `80:99:E7:E0:04:99`:

```shell
sudo nano /var/lib/bluetooth/80:13:16:1C:99:99/80:99:E7:E0:04:99/info
```

Locate the `[LinkKey]` section and update the `Key` value with your 32-character uppercase string:

```ini
[General]
Name=WH-1000XM4
Class=0x240404
SupportedTechnologies=BR/EDR;
Trusted=true
Blocked=false
CablePairing=false
Services=00000000-deca-fade-deca-deafdecacaff;0000.....-1000-8000-00805f9>

[DeviceID]
Source=2
Vendor=1356
Product=3416
Version=625

[LinkKey]
Key=6184C79E62999948999999998BE5007E <-------- Here
PINLength=0
```

### 4. Restart Bluetooth and Verify

Finally, restart the Bluetooth service:

```shell
sudo systemctl restart bluetooth
```

Turn on your headset or Bluetooth peripheral. It should immediately connect under Linux. Reboot into Windows to confirm that it reconnects automatically there as well. Your Bluetooth devices will now pair and connect seamlessly across both operating systems without needing to be re-paired.
