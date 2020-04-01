---
title:  Issue with Realtek SD card reader on Debian 8.1
author: xp
---
Just got a new Lenovo Thinkpad X250 laptop last month, the first thing to do was to format the disk, got rid off Windows and installed Debian 8.1. The only extra thing I needed to do was to install the `iwlwifi` firmware, and everything just worked. Well, almost, since I haven't had time to play with the fingerprint scanner yet.

Over the course of the month, I ran `apt-get upgrade` twice, for security update. Everything seemed ok, until today, when I inserted an SD card to look up something. Nothing came up. I looked at the system log, not even a single event. It was as if the card reader never detected any insertion.

I took a look at the hardware status, `sudo lspci -v`  showed:

```
02:00.0 Unassigned class [ff00]: Realtek Semiconductor Co., Ltd. RTS5227 PCI Express Card Reader (rev 01)
        Subsystem: Lenovo Device 2226
        Flags: fast devsel, IRQ 16
        Memory at f1100000 (32-bit, non-prefetchable) [size=4K]
        Capabilities: [40] Power Management version 3
        Capabilities: [50] MSI: Enable- Count=1/1 Maskable- 64bit+
        Capabilities: [70] Express Endpoint, MSI 00
        Capabilities: [100] Advanced Error Reporting
        Capabilities: [140] Device Serial Number 00-00-00-01-00-4c-e0-00
        Capabilities: [150] Latency Tolerance Reporting
        Capabilities: [158] L1 PM Substates
```

Hmm, it looked like the module is not even loaded. What's up? Let's take a look at 
`dmesg`:

```
[    1.315456] usbcore: registered new interface driver hub
[    1.315520] rtsx_pci 0000:02:00.0: irq 56 for MSI/MSI-X
[    1.315546] rtsx_pci 0000:02:00.0: rtsx_pci_acquire_irq: pcr->msi_en = 1, pci->irq = 56
[    1.315687] thermal LNXTHERM:00: registered as thermal_zone0
[    1.315690] ACPI: Thermal Zone [THM0] (45 C)
[    1.315884] pps_core: LinuxPPS API ver. 1 registered
[    1.315885] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    1.316046] PTP clock support registered
[    1.317042] e1000e: Intel(R) PRO/1000 Network Driver - 2.3.2-k
[    1.317045] e1000e: Copyright(c) 1999 - 2014 Intel Corporation.
[    1.317176] e1000e 0000:00:19.0: Interrupt Throttling Rate (ints/sec) set to dynamic conservative mode
[    1.317192] e1000e 0000:00:19.0: irq 57 for MSI/MSI-X
[    1.317479] SCSI subsystem initialized
[    1.317643] usbcore: registered new device driver usb
[    1.318186] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    1.318398] ehci-pci: EHCI PCI platform driver
[    1.319230] libata version 3.00 loaded.
[    1.415653] rtsx_pci: probe of 0000:02:00.0 failed with error -110
[    1.483569] e1000e 0000:00:19.0 eth0: registered PHC clock
```

Hmm, the module `rtsx_pci` failed to load, `with error -110`. This is funny, I'm very sure that the card reader was working flawlessly many times before, across multiple boot sessions. I have a couple of encrypted SD cards to store personal stuffs, and I had used them many times on this laptop. Something went wrong along one of the updates.

Browsing through the kernel bugzilla showed a few cases which had the same error code, and it seemed to be related to the `msi_en` option. Well, let's try to work around by disabling that. We need to unload the module first, because of its half-baked state:

```
sudo modprobe -r rtsx_pci
```

Then, load it again, with the option disabled:

```
sudo modprobe rtsx_pci msi_en=0
```

Now, the system log showed:

```
[ 5234.004011] rtsx_pci 0000:02:00.0: rtsx_pci_acquire_irq: pcr->msi_en = 0, pci->irq = 16
```

Ok, it seems to be loaded. Let's look at the status again, the command `sudo lspci -v` showed:

```
02:00.0 Unassigned class [ff00]: Realtek Semiconductor Co., Ltd. RTS5227 PCI Express Card Reader (rev 01)
        Subsystem: Lenovo Device 2226
        Flags: bus master, fast devsel, latency 0, IRQ 16
        Memory at f1100000 (32-bit, non-prefetchable) [size=4K]
        Capabilities: [40] Power Management version 3
        Capabilities: [50] MSI: Enable- Count=1/1 Maskable- 64bit+
        Capabilities: [70] Express Endpoint, MSI 00
        Capabilities: [100] Advanced Error Reporting
        Capabilities: [140] Device Serial Number 00-00-00-01-00-4c-e0-00
        Capabilities: [150] Latency Tolerance Reporting
        Capabilities: [158] L1 PM Substates
        Kernel driver in use: rtsx_pci
```

Now, let's insert an SD card into the slot, and look at system log, we got:

```
Aug  9 15:35:23 venus kernel: [ 5280.782622] mmc0: new high speed SDHC card at address aaaa
Aug  9 15:35:23 venus kernel: [ 5280.783923] mmcblk0: mmc0:aaaa SU08G 7.40 GiB 
Aug  9 15:35:23 venus kernel: [ 5280.797583]  mmcblk0: p1 p2
```

Bingo, now the card has been detected, and devices are created properly. All we have to do is to mount it.

To keep the module option across reboot sessions, create a file `/etc/modprobe.d/rtsx_pci.conf` and add the following line:

```
options rtsx_pci msi_en=0
```

Rebooted and it still worked!
