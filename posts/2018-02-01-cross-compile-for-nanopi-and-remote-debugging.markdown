---
title:  Cross compile for NanoPi and remote debugging
author: xp
tags: Programming, C++
---
Recently, I was called in to provide technical training to a young team of a new client, a startup developing remote control devices. I won't go into details of the idea which they are cooking, just to say that the team of C++ developers working on the device are bright young programmers, except the team does not have a team leader to manage the software process.

Another problem is that both founders of the company are not technical persons, but they are feeling that the productivity could use some improvement.

The first day I got there, I saw every C++ programmer has a Nanopi board attached to their laptop computer and was feverishly hacking away on it. Taking a closer look, I saw some are writing their code directly on the board, while others write their code on the laptop, but would copy over to the board for compiling and debugging.

That is the first thing that needs some improvement. Here is the setup to code on Linux on your workstation, cross-compile for the target (the Nanopi with Arm CPU in this case), run it on the target and do remote debugging from the workstation.

First of all, it is assumed that you have already installed the usual suspect packages for C/C++ development on your workstation. For cross-compiling for the Nanopi board (or any other arm-based board), you need to install the following (here, I'm assuming you are running Debian or some Debian-derived distros) on your workstation:

```
sudo apt-get install binutils-arm-linux-gnueabihf cpp-arm-linux-gnueabihf gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf binutils-multiarch gdb-multiarch
```

As we want to remotely debug our program on the target board, we will install the following on the target:

```
sudo apt-get install gdbserver
```

Now, let's create a small test project with the following code:

```
#include <iostream>

void test1()
{
    int meaning = 42;
    std::cout << "The meaning of life is " << meaning << std::endl;
}

int main()
{
    test1();
    return 0;
}
```

To cross-compile your project without having to install the necessary target libraries on your workstation, the easiest way is to grab a copy of the system libraries from your target board itself. Let's create a script called `sync-nano` in your project directory to fetch it:

```
#!/bin/bash
HOST=$1
rsync -rlzv --delete-after root@${HOST}:/{usr,lib} nano-m1-rootfs/
```

When you run

```
./sync-nano nanohost
```

where `nanohost` being the hostname or IP address of your Nanopi board, it will grab the directories `/usr` and `/lib` from the target board, and put them under the directory `nano-m1-rootfs`.

What is important about this method is that, we have the target libraries in one location, clean and neat. There is no littering all over the place. If you need to develop for multiple targets, it is very important to keep the house clean.

Let us build the project with `cmake`, and let's create a `CMakeLists.txt` file:

```
cmake_minimum_required (VERSION 3.9)
set (LANGUAGES CXX)

set (CMAKE_SYSTEM_NAME Linux)
set (CMAKE_SYSTEM_VERSION 1)
set (CMAKE_C_COMPILER /usr/bin/arm-linux-gnueabihf-gcc)
set (CMAKE_CXX_COMPILER /usr/bin/arm-linux-gnueabihf-g++)
set (CMAKE_FIND_ROOT_PATH $ENV{HOME}/test-nano/nano-m1-rootfs)
set (CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set (CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set (CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)

set (CMAKE_BINARY_DIR ${CMAKE_SOURCE_DIR}/build)
set (LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR}/lib)
set (EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin)

add_definitions ("-std=c++14 -g -Wall -Wextra -Werror=return-type")

include_directories(${PROJECT_SOURCE_DIR}/include)
set (testsrc
  src/test1.cpp
)

add_executable (test1 ${testsrc})
```

The build toolchain file above has nothing special, except telling it that we want to use the C/C++ compilers for arm-based target, and that they should look in the `nano-m1-rootfs` directory for headers and libraries. For more details on cmake for cross compiling, please visit the [wiki here](href="https://cmake.org/Wiki/CMake_Cross_Compiling").

Now, we are ready to build the project. Let's do the following:

```
mkdir build
cd build
cmake ..
make
```

Now, we can copy the compiled program onto the board:

```
scp bin/test1 xp@nanohost:
```

Let's run the program on the board, and we want to debug it from our workstation.

```
xp@nanopim1:~$ gdbserver :8888 ./test1
Process ./test1 created; pid = 5047
Listening on port 8888
```

We just started our program with `gdbserver` listening on port `8888`, and waiting for connection from a remote debugger.

Now, on our workstation, still in the `build` directory, let's start the `gdb` debugger and connect to the remote target board:

```
xp @ λ ★ gdb-multiarch bin/test1
GNU gdb (Debian 7.12-6+b1) 7.12.0.20161007-git
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law. Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from bin/test1...done.
(gdb) set sysroot ../nano-m1-rootfs/
(gdb) break main
Breakpoint 1 at 0x856: file /home/xp/test-nano/src/test1.cpp, line 12.
(gdb) target remote 192.168.1.3:8888
Remote debugging using 192.168.1.3:8888
Reading symbols from ../nano-m1-rootfs/lib/ld-linux-armhf.so.3...(no debugging symbols found)...done.
0xb6fd7980 in ?? () from ../nano-m1-rootfs/lib/ld-linux-armhf.so.3
(gdb) continue
Continuing.

Breakpoint 1, main (argc=1, argv=0xbefff784) at /home/xp/test-nano/src/test1.cpp:12
12 test1();
(gdb) break test1
Breakpoint 2 at 0x400806: file /home/xp/test-nano/src/test1.cpp, line 5.
(gdb) continue
Continuing.

Breakpoint 2, test1 () at /home/xp/test-nano/src/test1.cpp:5
5 int meaning = 42;
(gdb) n
7 std::cout << "The meaning of life is " << meaning << std::endl;
(gdb) continue
Continuing.
[Inferior 1 (process 5047) exited normally]
```

As you can see, we just went through a normal debugging session, as if the program is running on the workstation.

A few things need explanation.

We have to start the debugger `dbg-multiarch`, and not the native `gdb` on your workstation.

Then, we use the `set sysroot` command to specify the local directory that contains copies of target libraries in the corresponding subdirectories. And that is the `nano-m1-rootfs` directory that we had created earlier, and that we had copies of the `/usr` and `/lib` directories fetched from the board.

And from here on, everything would just work as if the program is run locally, on the workstation.

When the program exits, `gdbserver` on the remote target also exits:

```
xp@nanopim1:~$ gdbserver :8888 ./test1
Process ./test1 created; pid = 5047
Listening on port 8888
Remote debugging from host 192.168.1.9
The meaning of life is 42

Child exited with status 0
xp@nanopim1:~$
```

As we can see here, `gdbserver` on the board got a connection from host `192.168.1.9`, which is the IP address of my workstation, and as the program runs, it prints the phrase `The meaning of life is 42`, as expected.

With the method described above, you can also start up `gdbserver` and attach to a running process, such as:

```
xp@nanopim1:~$ gdbserver :8888 --attach 4321
Attached; pid = 4321
Listening on port 8888
```

Here, we have `gdbserver` attached to process `4321`, and is waiting for a remote connection. From your workstation, you can connect normally, as described above, and now, you have a local `gdb` controlling a remote process `4321` on the target board.

That's it. You can do a lot more fun things with `gdbserver` as your remote stub. Time for you to experiment now.
