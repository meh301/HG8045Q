# HG8045Q
This is an ongoing project aimed at reverse engineering modern Huawei ONT implementations to evaluate their security.

The target hardware can be commonly found distributed by Japanese ISP [So-Net](https://www.so-net.ne.jp) under the [Nuro](https://www.nuro.jp/hikari/) name.

The Huawei Echolife HG8045Q and similar variants is a 2gbps-capable (1gbps upload) GPON modem & router with many of its features locked down by the provider, with the most important of these *unavailable* features being Bridge mode. Furthermore, although the router is capable of speeds in excess of 1gbps on the fibre ONT input, none of its RJ45 ports are rated for anything higher than 1gbps. This is somewhat a shame for more advanced users who might want to use their own higher performance firewalls like pfsense or other deployments. The official manual outlining its features (in Japanese) can be found [here](https://www.nuro.jp/pdf/device/manual_HG8045Q.pdf).

## Security Evaluation
Huawei is not known for its security, especially in prior implementations. Some examples can be found [here, *Hacking Huawei HG8012H*](https://github.com/logon84/Hacking_Huawei_HG8012H_ONT) or [here, *pwn hg8120c pt1*](https://blog.leexiaolan.tk/pwn-huawei-hg8120c-ont-via-uart-part-1).

Usually, evaluating the security of a router is done in three main steps:
- Level 1: Finding flaws in the shipped software
- Level 2: Accessing root data through available hardware UART or JTAG points
- Level 3 and beyond: Dumping the NAND and/or direct memory read

#### Level 1: The infamous Huawei Master Account?
The biggest flaw in most Huawei products is of course their use of a master account that differs from the user-accessible admin account. To my knowledge, basically every model still has this account integrated. In the past few years though, ISPs have started to customize the master account with custom credentials. This is the case with Nuro's implementation of Huawei hardware. The usual credentials (**telecomadmin**) do not work. On top of this, the hacking scene in Japan is nowhere near as active as those in Mainland China who tend to find these new credentials on a weekly basis, either through security flaws or internal company leaks. Such things are not possible with Japanese variants. 

#### level 1.3: Router XML Configuration File Export?
Some ISPs trust that Huawei's security is up to scratch and allow you to download directly from the router's webUI a backup configuration XML that includes all user accounts and passwords for convenience. This XML is usually encrypted (but not always). Of course, it was quickly found out that Huawei uses the exact same (hardcoded) encryption key on **all** implementations. So if you can access this configuration XML, you essentially have access to the aforementioned master account. It was also found that some older webUI implementations would graphically disable access to this configuration backup but it could still be downloaded via a direct HTML link. In the case of the HG8045Q, the configuration file is *to my knowledge* not accessible in any shape or form.

#### Level 1.5: What about the other main flaw like an open telnet or ssh port?
This too has been a significant flaw in past Huawei implementations. Many routers used to have these ports open and sometimes even available on the WAN side for attackers to access from anywhere in the world. A quick **nmap** scan reveals that these services still do exist but have been disabled in software, as they should be. Sadly this means we cannot get a root shell to the router via sofware though.

*The following is a NAT reflection scan - these services are not visible when scanning from outside.*
```
Starting Nmap 7.80 ( https://nmap.org ) at 2020-09-11 22:41 JST
Nmap scan report for fpcXXXXXXX.tkycXXX.ap.nuro.jp (XXX.XX.XXX.XXX)
Host is up (0.011s latency).
Not shown: 996 closed ports
PORT   STATE    SERVICE
22/tcp filtered ssh
23/tcp filtered telnet
53/tcp open     domain
80/tcp open     http

Nmap done: 1 IP address (1 host up) scanned in 8.36 seconds
```


#### Level 2: Accessing the Hardware UART
Unable to find any significant software flaws, the next step was to open up the router and poke around at the hardware. What was immediately apparent was that the mainboard design hasn't really changed over the years. It was therefore easy to refer to older Huawei devices and find the pinout of the UART terminal.

A quick note: the HG8045Q required two tiny SMD 100ohm resistors to be added to the board at R1 and R2 respectively for a UART readout. Other Huawei modems are also known to do this. This requires decent soldering skills, especially if you are soldering on full size resistors to these small SMD pads (⇀_⇀)

(I will add better photos in the future)
![UART Port](img/nse-1001950187558289042-545515.jpg)

Sadly, this route resulted in not much. Unlike most other Huawei ONT modems, UART on this implementation is **unloaded** during the boot process. This results in an incomplete readout lacking full partition tables.

```
HuaWei StartCode 2012.02 (R15C10 Jan 30 2015 - 16:43:10)

NAND:  Nand(Hardware): 128 MiB
startcode select the uboot to load
the high RAM is :8080103c
startcode uboot boot count:0
Use the UbootA to load first
Use the UbootA to load success


U-Boot 2010.03 (R16C00 Jan 28 2016 - 20:16:15)

DRAM:  128 MB
Boot From NAND flash
Chip Type is SD5115T
NAND:  Special Nand id table Version 1.23
Nand ID: 0x01 0xF1 0x00 0x1D 0x01 0xF1 0x00 0x1D
ECC Match pagesize:2K, oobzie:64, ecctype:4bit

Nand(Hardware): Block:128KB Page:2KB Chip:128MB*1 OOB:64B ECC:4bit 
128 MiB
Using default environment

In:    serial
Out:   serial
Err:   serial
PHY power down !!!
[main.c__6058]::CRC:0x3d80a8b4, Magic1:0x5a5a5a5a, Magic2:0xa5a5a5a5, count:0, CommitedArea:0x0, Active:0x0, RunFlag:0x0
Start from main system(0x0)!
CRC:0x3d80a8b4, Magic1:0x5a5a5a5a, Magic2:0xa5a5a5a5, count:1, CommitedArea:0x0, Active:0x0, RunFlag:0x0

0x000000100000-0x000008000000 : "mtd=1"
UBI: attaching mtd1 to ubi0
Main area (A) is OK!

CRC:0x93e83925, Magic1:0x5a5a5a5a, Magic2:0xa5a5a5a5, count:1, CommitedArea:0x0, Active:0x0, RunFlag:0x0

doublecore not found!
Unmounting UBIFS volume file_system!
Unmount ubifs success!
Bootcmd:ubi read 0x85c00000 kernelA 0x1b2c86; bootm 0x85c00054
BootArgs:noalign mem=118M console=ttyAMA1,115200 ubi.mtd=1 root=/dev/mtdblock11 rootfstype=squashfs mtdparts=hinand:0x100000(startcode),0x7f00000(ubifs),-(reserved) pcie0_sel=x1 pcie1_sel=x1 maxcpus=2 l2_cache=l2hi coherent_pool=4M user_debug=0x1f panic=1 skb_priv=128
U-boot Start from NORMAL Mode!

## Booting kernel from Legacy Image at 85c00054 ...
   Image Name:   Linux-3.10.53-HULK2
   Image Type:   ARM Linux Kernel Image (uncompressed)
   Data Size:    1780722 Bytes =  1.7 MB
   Load Address: 80e08000
   Entry Point:  80e08000
   Memory Start: 80a00000

   Loading Kernel Image ... OK
OK
   kernel loaded at 0x80a08000, end = 0x80bbabf2

Starting kernel ...
```
#### Dumping the NAND
I accidentally shorted 12v to ground, immediately killing the router mainboard soon after accessing the UART port. To salvage what was left of the board, I decided to order some NAND flash dumping hardware (I didn't have anything that could do TSOP48 NAND chips) and dump the NAND of the HG8045Q.

In the case of this mainboard, the NAND flash used was [Spansion S34ML01G100TF100 SLC 128MB NAND](https://www.kynix.com/Detail/589387/S34ML01G100TF100.html). *(Judging from hex readouts of the flash dump, this chip is most probably not the only one in use as the filesystem has a list of compatible chips integrated.)*
```
NAND ID: 0x1f1001d_0x1f1001d
Manufacturer: Spansion
Page data area size: 2048 bytes
Page spare area size: 64 bytes
Pages per block: 64 pages
Chip-select signals: 1
Chip-select blocks: 1024
Chip blocks: 1024
Total memory size: 128Mbytes
Range 0x0 - 0x7ffffff
Memory type: SLC NAND
```

![Nand Chip in Programmer](img/nse-4504352926381896433-546030.jpg)

I have provided two NAND dumps in this GIT repository: 
- [The raw dump](NAND_Dump/hg8045q_raw.bin) - 132MB
- [The dump with OOB blocks removed](NAND_Dump/hg8045q_noOOB.bin) - 128MB

*(Shoutout to [Jean-Michel Picod](https://www.j-michel.org/blog/2014/05/27/from-nand-chip-to-files) for his [OOB removing script](https://github.com/Hitsxx/NandTool))*

I have also included my attempts at extracting the [NAND dump](Partially_Extracted/).

These dumps and extracted images resulted in some more information via Binwalk:
```
binwalk hg8045q_raw.bin

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
88740         0x15AA4         CRC32 polynomial table, little endian
90530         0x161A2         CRC32 polynomial table, little endian
91776         0x16680         CRC32 polynomial table, little endian
1081344       0x108000        UBI erase count header, version: 1, EC: 0x0, VID header offset: 0x800, data offset: 0x1000
```

```
binwalk hg8045q_noOOB.bin

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
86052         0x15024         CRC32 polynomial table, little endian
87794         0x156F2         CRC32 polynomial table, little endian
89008         0x15BB0         CRC32 polynomial table, little endian
1048576       0x100000        UBI erase count header, version: 1, EC: 0x0, VID header offset: 0x800, data offset: 0x1000
```

```
binwalk img-1245770326_vol-file_system.ubifs
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
126976        0x1F000         UBIFS filesystem master node, CRC: 0x5C012B49, highest inode: 189, commit number: 34
129024        0x1F800         UBIFS filesystem master node, CRC: 0xA88CE39, highest inode: 189, commit number: 35
131072        0x20000         UBIFS filesystem master node, CRC: 0xC5AE5F0D, highest inode: 189, commit number: 36
133120        0x20800         UBIFS filesystem master node, CRC: 0xD1FF402A, highest inode: 189, commit number: 36
135168        0x21000         UBIFS filesystem master node, CRC: 0x9F399C91, highest inode: 189, commit number: 37
137216        0x21800         UBIFS filesystem master node, CRC: 0xBBAB57C2, highest inode: 189, commit number: 37
139264        0x22000         UBIFS filesystem master node, CRC: 0x140F9B62, highest inode: 189, commit number: 38
141312        0x22800         UBIFS filesystem master node, CRC: 0x5D84F324, highest inode: 189, commit number: 38
143360        0x23000         UBIFS filesystem master node, CRC: 0x5BA9FAA9, highest inode: 189, commit number: 39
145408        0x23800         UBIFS filesystem master node, CRC: 0x7F3B31FA, highest inode: 189, commit number: 39
147456        0x24000         UBIFS filesystem master node, CRC: 0x67E599BC, highest inode: 189, commit number: 40
149504        0x24800         UBIFS filesystem master node, CRC: 0x6BD56CA1, highest inode: 189, commit number: 40
151552        0x25000         UBIFS filesystem master node, CRC: 0x591320DD, highest inode: 189, commit number: 41
153600        0x25800         UBIFS filesystem master node, CRC: 0x4D423FFA, highest inode: 189, commit number: 41
155648        0x26000         UBIFS filesystem master node, CRC: 0xABD72128, highest inode: 189, commit number: 42
157696        0x26800         UBIFS filesystem master node, CRC: 0xEEC24293, highest inode: 189, commit number: 42
159744        0x27000         UBIFS filesystem master node, CRC: 0xA66156B3, highest inode: 189, commit number: 43
161792        0x27800         UBIFS filesystem master node, CRC: 0x82F39DE0, highest inode: 189, commit number: 43
163840        0x28000         UBIFS filesystem master node, CRC: 0x904E0AC6, highest inode: 189, commit number: 44
165888        0x28800         UBIFS filesystem master node, CRC: 0x165438AD, highest inode: 189, commit number: 44
167936        0x29000         UBIFS filesystem master node, CRC: 0x84A9D7CC, highest inode: 189, commit number: 45
169984        0x29800         UBIFS filesystem master node, CRC: 0x25D4A04C, highest inode: 194, commit number: 46
172032        0x2A000         UBIFS filesystem master node, CRC: 0x3185BF6B, highest inode: 194, commit number: 46
174080        0x2A800         UBIFS filesystem master node, CRC: 0x919D0AAA, highest inode: 194, commit number: 47
176128        0x2B000         UBIFS filesystem master node, CRC: 0x7DFADE9C, highest inode: 199, commit number: 48
178176        0x2B800         UBIFS filesystem master node, CRC: 0x69ABC1BB, highest inode: 199, commit number: 48
180224        0x2C000         UBIFS filesystem master node, CRC: 0xA7B5D7C9, highest inode: 199, commit number: 49
182272        0x2C800         UBIFS filesystem master node, CRC: 0x7277912F, highest inode: 199, commit number: 50
184320        0x2D000         UBIFS filesystem master node, CRC: 0x3762F294, highest inode: 199, commit number: 50
186368        0x2D800         UBIFS filesystem master node, CRC: 0x363981AB, highest inode: 199, commit number: 51
188416        0x2E000         UBIFS filesystem master node, CRC: 0xF5D20A09, highest inode: 199, commit number: 52
190464        0x2E800         UBIFS filesystem master node, CRC: 0xF9E2FF14, highest inode: 199, commit number: 52
192512        0x2F000         UBIFS filesystem master node, CRC: 0x9962E5AE, highest inode: 199, commit number: 53
194560        0x2F800         UBIFS filesystem master node, CRC: 0x3932BD59, highest inode: 199, commit number: 54
196608        0x30000         UBIFS filesystem master node, CRC: 0x2D63A27E, highest inode: 199, commit number: 54
198656        0x30800         UBIFS filesystem master node, CRC: 0x12E27226, highest inode: 199, commit number: 55
200704        0x31000         UBIFS filesystem master node, CRC: 0x4AD5BA64, highest inode: 199, commit number: 56
202752        0x31800         UBIFS filesystem master node, CRC: 0x46E54F79, highest inode: 199, commit number: 56
204800        0x32000         UBIFS filesystem master node, CRC: 0x4A41BC9C, highest inode: 199, commit number: 57
206848        0x32800         UBIFS filesystem master node, CRC: 0x1D0E241, highest inode: 199, commit number: 58
208896        0x33000         UBIFS filesystem master node, CRC: 0x1581FD66, highest inode: 199, commit number: 58
210944        0x33800         UBIFS filesystem master node, CRC: 0x7390F444, highest inode: 199, commit number: 59
253952        0x3E000         UBIFS filesystem master node, CRC: 0x5031DE54, highest inode: 189, commit number: 34
256000        0x3E800         UBIFS filesystem master node, CRC: 0x6B83B24, highest inode: 189, commit number: 35
258048        0x3F000         UBIFS filesystem master node, CRC: 0xD1FF402A, highest inode: 189, commit number: 36
260096        0x3F800         UBIFS filesystem master node, CRC: 0xDDCFB537, highest inode: 189, commit number: 36
262144        0x40000         UBIFS filesystem master node, CRC: 0xBBAB57C2, highest inode: 189, commit number: 37
264192        0x40800         UBIFS filesystem master node, CRC: 0xB79BA2DF, highest inode: 189, commit number: 37
266240        0x41000         UBIFS filesystem master node, CRC: 0x5D84F324, highest inode: 189, commit number: 38
268288        0x41800         UBIFS filesystem master node, CRC: 0x51B40639, highest inode: 189, commit number: 38
270336        0x42000         UBIFS filesystem master node, CRC: 0x7F3B31FA, highest inode: 189, commit number: 39
272384        0x42800         UBIFS filesystem master node, CRC: 0x730BC4E7, highest inode: 189, commit number: 39
274432        0x43000         UBIFS filesystem master node, CRC: 0x6BD56CA1, highest inode: 189, commit number: 40
276480        0x43800         UBIFS filesystem master node, CRC: 0x4F47A7F2, highest inode: 189, commit number: 40
278528        0x44000         UBIFS filesystem master node, CRC: 0x4D423FFA, highest inode: 189, commit number: 41
280576        0x44800         UBIFS filesystem master node, CRC: 0x4172CAE7, highest inode: 189, commit number: 41
282624        0x45000         UBIFS filesystem master node, CRC: 0xEEC24293, highest inode: 189, commit number: 42
284672        0x45800         UBIFS filesystem master node, CRC: 0xE2F2B78E, highest inode: 189, commit number: 42
286720        0x46000         UBIFS filesystem master node, CRC: 0x82F39DE0, highest inode: 189, commit number: 43
288768        0x46800         UBIFS filesystem master node, CRC: 0x8EC368FD, highest inode: 189, commit number: 43
290816        0x47000         UBIFS filesystem master node, CRC: 0x165438AD, highest inode: 189, commit number: 44
292864        0x47800         UBIFS filesystem master node, CRC: 0x1A64CDB0, highest inode: 189, commit number: 44
294912        0x48000         UBIFS filesystem master node, CRC: 0x889922D1, highest inode: 189, commit number: 45
296960        0x48800         UBIFS filesystem master node, CRC: 0x3185BF6B, highest inode: 194, commit number: 46
299008        0x49000         UBIFS filesystem master node, CRC: 0x3DB54A76, highest inode: 194, commit number: 46
301056        0x49800         UBIFS filesystem master node, CRC: 0xB50FC1F9, highest inode: 194, commit number: 47
303104        0x4A000         UBIFS filesystem master node, CRC: 0x69ABC1BB, highest inode: 199, commit number: 48
305152        0x4A800         UBIFS filesystem master node, CRC: 0x659B34A6, highest inode: 199, commit number: 48
307200        0x4B000         UBIFS filesystem master node, CRC: 0xE2A0B472, highest inode: 199, commit number: 49
309248        0x4B800         UBIFS filesystem master node, CRC: 0x3762F294, highest inode: 199, commit number: 50
311296        0x4C000         UBIFS filesystem master node, CRC: 0x3B520789, highest inode: 199, commit number: 50
313344        0x4C800         UBIFS filesystem master node, CRC: 0x22689E8C, highest inode: 199, commit number: 51
315392        0x4D000         UBIFS filesystem master node, CRC: 0xF9E2FF14, highest inode: 199, commit number: 52
317440        0x4D800         UBIFS filesystem master node, CRC: 0xDD703447, highest inode: 199, commit number: 52
319488        0x4E000         UBIFS filesystem master node, CRC: 0x955210B3, highest inode: 199, commit number: 53
321536        0x4E800         UBIFS filesystem master node, CRC: 0x2D63A27E, highest inode: 199, commit number: 54
323584        0x4F000         UBIFS filesystem master node, CRC: 0x21535763, highest inode: 199, commit number: 54
325632        0x4F800         UBIFS filesystem master node, CRC: 0x57F7119D, highest inode: 199, commit number: 55
327680        0x50000         UBIFS filesystem master node, CRC: 0x46E54F79, highest inode: 199, commit number: 56
329728        0x50800         UBIFS filesystem master node, CRC: 0x52B4505E, highest inode: 199, commit number: 56
331776        0x51000         UBIFS filesystem master node, CRC: 0x46714981, highest inode: 199, commit number: 57
333824        0x51800         UBIFS filesystem master node, CRC: 0x1581FD66, highest inode: 199, commit number: 58
335872        0x52000         UBIFS filesystem master node, CRC: 0x19B1087B, highest inode: 199, commit number: 58
337920        0x52800         UBIFS filesystem master node, CRC: 0x57023F17, highest inode: 199, commit number: 59
1689648       0x19C830        gzip compressed data, from Unix, last modified: 2016-01-28 12:17:57
2442965       0x2546D5        Unix path: /sys/class/ubil
3321904       0x32B030        gzip compressed data, maximum compression, from Unix, last modified: 1970-01-01 00:00:48 (bogus date)
19324991      0x126E03F       mcrypt 2.2 encrypted data, algorithm: blowfish-448, mode: CBC, keymode: 8bit
19324999      0x126E047       mcrypt 2.2 encrypted data, algorithm: blowfish-448, mode: CBC, keymode: 8bit
19327039      0x126E83F       mcrypt 2.2 encrypted data, algorithm: blowfish-448, mode: CBC, keymode: 8bit
19327047      0x126E847       mcrypt 2.2 encrypted data, algorithm: blowfish-448, mode: CBC, keymode: 8bit
20153081      0x13382F9       mcrypt 2.5 encrypted data, algorithm: "1l", keysize: 3436 bytes, mode: "m",
20155129      0x1338AF9       mcrypt 2.5 encrypted data, algorithm: "1l", keysize: 3436 bytes, mode: "m",
20157177      0x13392F9       mcrypt 2.5 encrypted data, algorithm: "1l", keysize: 3436 bytes, mode: "m",
20159225      0x1339AF9       mcrypt 2.5 encrypted data, algorithm: "1l", keysize: 3436 bytes, mode: "m",
20161273      0x133A2F9       mcrypt 2.5 encrypted data, algorithm: "1l", keysize: 3436 bytes, mode: "m",
20672560      0x13B7030       gzip compressed data, maximum compression, from Unix, last modified: 1970-01-01 00:00:30 (bogus date)
```

Going through the NAND dump with a hex editor revealed some interesting strings:
```
admin_iksyomuac13 iksyomuac13_admin_3204 HG8045-409E-bg b9tt6hre HG8045-409E-a b9tt6hre

0x000000100000-0x000008000000 : "mtd=1"

mtdparts=hinand:0x100000(startcode)ro,0x7f00000(ubifs),-(reserved)
```

Sadly, I do not have a lot of experience with extracting the data from the NAND flash, especially as the filesystem in use is UBIFS. I have tried several things to mount the filesystem but am still struggling with it. If you have any suggestions, I would be more than glad to listen to them. Feel free to send an email to **alex *[at]* kenchitaru.studio**. It seems that the offsets are still wrong. I also get a CRC error at certain points but I do not think it is due to a bad dump as reading from the chip results in the exact same file everytime.

## To be continued... (please help)
