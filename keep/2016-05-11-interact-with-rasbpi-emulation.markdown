---
title:  Robotic systems for cleaning up solar panels
author: xp
---
I was working on a robotic side project, and programming the Raspberry Pi to control the robot. One day, I was on the road, and need to do some minor code tweaking on my laptop computer. But I did not have the RPi board with me. I did some testing on the laptop, then I suddenly remembered I had the Raspbian image on my disk, which I have downloaded but have never done anything with it. I was thinking, hmm, why not spin this thing up in Qemu, just for fun?

But I still needed a kernel for Qemu. Looking up on the Internet where I could grab one quickly, I found this: [https://github.com/dhruvvyas90/qemu-rpi-kernel](https://github.com/dhruvvyas90/qemu-rpi-kernel).

Good thing that I had a fast connection pipe at the time. A few minutes later, I had this Qemu console up, showing the boot sequence of Raspbian. But you probably prefer to have the login prompt in your own terminal (whichever terminal you use), which you have probably done a bit of customization, instead of that Qemu raw thing.

So, I tried to change the Qemu command to interact in the terminal directly:

```
qemu-system-arm -kernel kernel-qemu-4.1.13-jessie -cpu arm1176 -m 256 \
    -M versatilepb -no-reboot -serial stdio -curses \
    -append "root=/dev/sda2 panic=1 rootfstype=ext4 rw console=ttyAMA0" \
    -hda 2016-03-18-raspbian-jessie-lite.img
```

In the boot sequence, I saw a message saying that

```
A start job is running for dev-ttyAMA0.device...
```

After a while, the boot sequence stopped, and I got this:

```
[  OK  ] Started /etc/rc.local Compatibility.
         Starting Wait for Plymouth Boot Screen to Quit...
         Starting Terminate Plymouth Boot Screen...
[  OK  ] Started Wait for Plymouth Boot Screen to Quit.
[  OK  ] Started Terminate Plymouth Boot Screen.
         Starting Getty on tty1...
[  OK  ] Started Getty on tty1.
[  OK  ] Started LSB: Start NTP daemon.
[ TIME ] Timed out waiting for device dev-ttyAMA0.device.
[DEPEND] Dependency failed for Serial Getty on ttyAMA0.
[  OK  ] Reached target Login Prompts.
[  OK  ] Reached target Multi-User System.
[  OK  ] Reached target Graphical Interface.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Update UTMP about System Runlevel Changes.
```

Ok, so no login prompt. Looking up for some information, lots of instructions, such as [this](https://www.aurel32.net/info/debian_arm_qemu.php)>, told to add the following line to `/etc/inittab` file:

```
T0:23:respawn:/sbin/getty -L ttyAMA0 9600 vt100
```

But we had a Jessie version of Raspbian, which used Systemd. Therefore, those instructions do not apply. What's up with this? We were trying to ask the kernel to use this `ttyAMA0` serial port as the console, but it seemed to get stuck there.

But you might ask, what's wrong with ssh into the Qemu guest? Absolutely nothing wrong about it, I was just thinking about having the interaction console on my terminal. As a matter of fack, let's cheat a bit and login with ssh first. To login with ssh, you can map a local port on your host OS to the 22 port on the Qemu guest, like this:

```
qemu-system-arm -kernel kernel-qemu-4.1.13-jessie -cpu arm1176 -m 256 \
    -M versatilepb -no-reboot -serial stdio -curses -redir tcp:2222::22 \
    -append "root=/dev/sda2 panic=1 rootfstype=ext4 rw console=ttyAMA0" \
    -hda 2016-03-18-raspbian-jessie-lite.img
```

Here, we map the local port 2222 to the 22 ssh port of the Qemu guest. After booting up the Qemu guest OS, you can now login like this:

```
ssh -p 2222 pi@localhost
```

This will redirect any traffic to the local port 2222 to the 22 ssh port on the emulated guest virtual machine.

After logging in, let's see if we could send and receive data on ttyAMA0. To send data, do this:

```
pi@raspberrypi:~ $ echo "hello world" > /dev/ttyAMA0
```

And on the boot console, we saw:

```
[ TIME ] Timed out waiting for device dev-ttyAMA0.device.
[DEPEND] Dependency failed for Serial Getty on ttyAMA0.
[  OK  ] Reached target Login Prompts.
[  OK  ] Reached target Multi-User System.
[  OK  ] Reached target Graphical Interface.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Update UTMP about System Runlevel Changes.
hello world
```

We actually did receive the message on the console. Now, let's try to receive messages on `ttyAMA0` by doing this:

```
pi@raspberrypi:~ $ cat /dev/ttyAMA0 
```

This was now waiting for incoming message. We went to the boot console, and typed "hello back" and Enter. And we could see the "hello back" message on the ssh console. So far so good, but we were still not getting anywhere.
