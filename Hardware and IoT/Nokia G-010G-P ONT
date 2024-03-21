# Nokia G-010G-P ONT
### TL;DR
This is a Nokia G-010G-P ONT (Optical network terminal) recovered from a scrapyard runnning an embedded Linux 2.6.30.9. The ONTs are network devices usually installed in home fiber-optics broadband facilities. Their purpose is to convert the optical signal from the ISP into an electrical signal, so home Gigabit-Ethernet routers can interpret it. These devices can be internal, if integrated inside the router delivered by the ISP; or external, as in the case of this device. When external, it must be installed between the home router and the ISP network, and acts as a demarcation point between the carrier's local loop and the customer's premises wiring.

This is very easy to root this device as telnet port is open and root credentials can be easily found in internet; therefore, it is a good starting point for pentesters wanting to initiate themselves in the field of hardware/IoT hacking. Apart from telnet root, this write-up includes: rooting via UART port, busybox upgrade, port scanning and web interface enumeration.
### KEYWORDS
Nokia G-010G-P, ONT, UART, CP2120, GPON.
### REFERENCES
https://www.normann-engineering.com/en/refurbished/products/1234/nokia-g-010g-p/<br>
https://www.normann-engineering.com/products/product_pdf/gpon_equipment/nokia/EN_G-010G-P.pdf<br>
https://www.winbond.com/hq/product/code-storage-flash-memory/serial-nor-flash/?__locale=en&partNo=W25Q128JV<br>
https://www.busybox.net/downloads/binaries/1.18.4/busybox-mips<br>
https://hack-gpon.org/ont-nokia-g-010g-p/<br>
https://forum.openwrt.org/t/help-for-how-to-root-the-g-010g-p-via-usb-ttl/115656<br>
<br>
### TOOLING
Screwdriver, CP2102 USB to UART converter, Dupont wire connectors.
### ENUMERATION
A quick search returns links and datasheets.
- https://www.normann-engineering.com/en/refurbished/products/1234/nokia-g-010g-p
- https://www.normann-engineering.com/products/product_pdf/gpon_equipment/nokia/EN_G-010G-P.pdf

First step is to remove the cover case to get access to the device internals. Just turn the ONT around and remove the top pads, there are 2 screws beneath them, unscrew them and open the case.

![IMG_20240321_130907 (Medium)](https://github.com/g1vi/AllTheWriteUps/assets/120142960/71c00c59-0cc9-4149-a089-b62e48c96660)

Inspect the PCB and identify the relevant items.
- Realtek RTL90601b network chipset.
- UART port, with soldered connection pins.
- Winbond 25Q128JSVQ flash memory to store the firmware (https://www.winbond.com/hq/product/code-storage-flash-memory/serial-nor-flash/?__locale=en&partNo=W25Q128JV)

![IMG_20240321_130056 (Medium)](https://github.com/g1vi/AllTheWriteUps/assets/120142960/b2dc8e8f-cba2-4c54-978c-9f655a746774)
### UART ROOT
First step is to find the UART ground with a multimeter, then proceed to find Rx and Tx pins. Then connect the UART pins to the CP2102 USB converter, which in turn is connected to the laptop USB port.

![IMG_20240320_112236 (Medium)](https://github.com/g1vi/AllTheWriteUps/assets/120142960/67919b53-90e4-4de4-9809-72a8e3966cbf)

![IMG_20240320_112435 (Medium)](https://github.com/g1vi/AllTheWriteUps/assets/120142960/9a7853dc-3279-4de9-ab6a-7bc897d08cbf)

Keep in mind that UART GND goes to CP2102 GND, Tx UART goes to Rx CP2102, and Rx UART goes to Tx CP2102.

![IMG_20240321_130205 (Medium)](https://github.com/g1vi/AllTheWriteUps/assets/120142960/0273d625-85c9-4fe7-80ab-6b51f6225429)

![IMG_20240320_112408 (Medium)](https://github.com/g1vi/AllTheWriteUps/assets/120142960/6afdf899-1e2d-4505-a08c-68558971aea8)

Open a `picocom` serial terminal on 115200 bauds in Kali and power on the device, the bootlog is dumped.
```markdown
> picocom -b 115200 /dev/ttyUSB0
picocom v3.1

port is        : /dev/ttyUSB0
flowcontrol    : none
baudrate is    : 115200
parity is      : none
databits are   : 8
stopbits are   : 1
escape is      : C-a
local echo is  : no
noinit is      : no
noreset is     : no
hangup is      : no
nolock is      : no
send_cmd is    : sz -vv
receive_cmd is : rz -vv -E
imap is        : 
omap is        : 
emap is        : crcrlf,delbs,
logfile is     : none
initstring     : none
exit_after is  : not set
exit is        : no

Type [C-a] [C-h] to see available commands
Terminal ready


PRELOADER.3.2.0.REL_R2787.w11
DD: efuse2: 00000060 => 55e6 => 0x01c4; 00000000 => 0xbcc8... done
DD: efuse3: 0000bebe => 0xbcc4... done
DD: efuse4: 00008888 => 0xbcdc... done
DD: efuse5: 0000aaaa => 0xbce0... done
DD: efuse6: di: a1; sds_num: 1; gphy_num: 0
DD: efuse gphy: done
DD: efuse sds: [0000fd2b => 00000421] done
DD: efuse lxb: break
II: Disabling OCP/LX timeout monitors... done
Get prestate = 1 in the init stage...
II: NOR SPI-F... Generic 3B NOR SPI-F [00ef4018] selected... done
II: OCP 100MHz(400/P4/C1), MEM 325MHz, LX 200MHz, SPIF 50MHz(1000/(5*4))

DRAM AUTO CALIBRATION
AutoK: ZQ Calibration Passed
AutoK: MR0: 0x00100b62
AutoK: MR1: 0x00110040
AutoK: MR2: 0x00120000
AutoK: MR3: 0x00130000
bit:00, max_w_s(00), max_w_len(17), max_r_s(00), max_r_len(31): (0xb8001510) = 0x1e0f00
bit:01, max_w_s(00), max_w_len(19), max_r_s(02), max_r_len(29): (0xb8001514) = 0x1e1102
bit:02, max_w_s(00), max_w_len(17), max_r_s(04), max_r_len(27): (0xb8001518) = 0x1e1304
bit:03, max_w_s(00), max_w_len(19), max_r_s(00), max_r_len(31): (0xb800151c) = 0x1e0f00
bit:04, max_w_s(00), max_w_len(19), max_r_s(04), max_r_len(27): (0xb8001520) = 0x1e1304
bit:05, max_w_s(00), max_w_len(17), max_r_s(02), max_r_len(29): (0xb8001524) = 0x1e1102
bit:06, max_w_s(00), max_w_len(19), max_r_s(00), max_r_len(31): (0xb8001528) = 0x1e0f00
bit:07, max_w_s(00), max_w_len(19), max_r_s(00), max_r_len(31): (0xb800152c) = 0x1e0f00
bit:08, max_w_s(00), max_w_len(17), max_r_s(02), max_r_len(29): (0xb8001530) = 0x1e1102
bit:09, max_w_s(00), max_w_len(19), max_r_s(00), max_r_len(31): (0xb8001534) = 0x1e0f00
bit:10, max_w_s(04), max_w_len(03), max_r_s(00), max_r_len(31): (0xb8001538) = 0x1e0f00
bit:11, max_w_s(00), max_w_len(19), max_r_s(02), max_r_len(29): (0xb800153c) = 0x1e1102
bit:12, max_w_s(00), max_w_len(21), max_r_s(00), max_r_len(31): (0xb8001540) = 0x1e0f00
bit:13, max_w_s(00), max_w_len(19), max_r_s(02), max_r_len(29): (0xb8001544) = 0x1e1102
bit:14, max_w_s(00), max_w_len(19), max_r_s(02), max_r_len(29): (0xb8001548) = 0x1e1102
bit:15, max_w_s(00), max_w_len(19), max_r_s(00), max_r_len(31): (0xb800154c) = 0x1e0f00
bit:16, max_w_s(00), max_w_len(17), max_r_s(00), max_r_len(31): (0xb8001550) = 0x1e0f00
bit:17, max_w_s(00), max_w_len(17), max_r_s(02), max_r_len(29): (0xb8001554) = 0x1e1102
bit:18, max_w_s(00), max_w_len(17), max_r_s(02), max_r_len(29): (0xb8001558) = 0x1e1102
bit:19, max_w_s(00), max_w_len(17), max_r_s(02), max_r_len(29): (0xb800155c) = 0x1e1102
bit:20, max_w_s(00), max_w_len(17), max_r_s(04), max_r_len(27): (0xb8001560) = 0x1e1304
bit:21, max_w_s(00), max_w_len(17), max_r_s(02), max_r_len(29): (0xb8001564) = 0x1e1102
bit:22, max_w_s(00), max_w_len(19), max_r_s(00), max_r_len(31): (0xb8001568) = 0x1e0f00
bit:23, max_w_s(00), max_w_len(19), max_r_s(00), max_r_len(31): (0xb800156c) = 0x1e0f00
bit:24, max_w_s(00), max_w_len(15), max_r_s(02), max_r_len(29): (0xb8001570) = 0x1e1102
bit:25, max_w_s(10), max_w_len(03), max_r_s(02), max_r_len(29): (0xb8001574) = 0x1e1102
bit:26, max_w_s(00), max_w_len(15), max_r_s(02), max_r_len(29): (0xb8001578) = 0x1e1102
bit:27, max_w_s(00), max_w_len(17), max_r_s(04), max_r_len(27): (0xb800157c) = 0x1e1304
bit:28, max_w_s(00), max_w_len(19), max_r_s(02), max_r_len(29): (0xb8001580) = 0x1e1102
bit:29, max_w_s(00), max_w_len(17), max_r_s(02), max_r_len(29): (0xb8001584) = 0x1e1102
bit:30, max_w_s(00), max_w_len(19), max_r_s(04), max_r_len(27): (0xb8001588) = 0x1e1304
bit:31, max_w_s(00), max_w_len(17), max_r_s(00), max_r_len(31): (0xb800158c) = 0x1e0f00
AutoK: dram auto calibrtaion is done

II: MEM_PROBE_OK
II: MEM_XLAT_OK
II: MEM_TO_REG_OK
II: MEM_CNTLR_ZQ_STRICT_OK
II: MEM_CAL_OK
II: cntlr zq result: 0018e975
II: dcr result: 11210000
II: Decompressing U-Boot (0x80f00000 <- 0x9fc1a200)... (212 KB <- 80 KB) OK
II: Starting U-boot... 

CIG Version: 3.04.09

U-Boot 2011.12.NA-svn94986 (Jan 15 2016 - 17:00:30)

Board: RTL9601B CPU:100MHz LX:200MHz DDR2:325MHz..
SPI-F: 1x16 MB (plr_flash_info @ 9f002d20)
[debug] start to init flash_info (1)
[debug] <0> unit size=0x10000, unit count=256
*** Warning - bad CRC, using default environment

*** !!! Check Result: Chk_CRC: 0x0, Chk_Empty: 0x0
Net:   LUNA GMAC 
Warning: eth device name has a space!

**************************************
*                                    *
*  KEY -- Enter console terminal     *
*                                    *
**************************************
waiting for your select .....
do_jffs2_fsload offset 0x80400000 loadaddr=0xbd088000..
get loadbit 0,bootbit 0
Get current Image = ImageB..

--- jffs2_part_info: partition number 3 for device sflash0 (sflash.0)

Loading 'uImage' from  CRAMFS Partition 'imageb' to 0x80400000.

Root Filesystem crc check successfully!


PRODUCT:  sfu
SS  :  noss
DSP :  nodsp
WIFI:  nowifi
XDSL:  nodsl
Software version:  3FE45655BOCK56
## Booting kernel from Legacy Image at 80400000 ...
   Image Name:   Linux Kernel Image
   Created:      2015-12-11   6:28:09 UTC
   Image Type:   MIPS Linux Kernel Image (gzip compressed)
   Data Size:    1440099 Bytes = 1.4 MB
   Load Address: 80000000
   Entry Point:  80000000
   Verifying Checksum ... OK
   Uncompressing Kernel Image ... OK

Starting kernel ...

Linux version 2.6.30.9-cig-sfu-1 (taojipeng@sw1-dev) (gcc version 4.4.6 (Realtek RSDK-1.5.6p2) ) #1 Fri Dec 11 11:48:11 CST 2015
CPU revision is: 0000dc02
Determined physical RAM map:
 memory: 03800000 @ 00000000 (usable)
User-defined physical RAM map:
 memory: 01f00000 @ 00000000 (usable)
Zone PFN ranges:
  Normal   0x00000000 -> 0x00001f00
Movable zone start PFN for each node
early_node_map[1] active PFN ranges
    0: 0x00000000 -> 0x00001f00
Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 7874
Kernel command line: console=ttyS0,115200 mem=31M root=/dev/mtdblock3 mtdparts=sflash:512K@0x0(Boot),0x180000@0x80000(Config),7M@0x200000(ImageA),7M@0x900000(ImageB) rootfstype=cramfs hasEeprom=0 5srst=0
root_dev_setup line 179 root: /dev/mtdblock3
icache: 64kB/32B, dcache: 32kB/32B, scache: 0kB/0B
NR_IRQS:128
PID hash table entries: 128 (order: 7, 512 bytes)
console [ttyS0] enabled
Dentry cache hash table entries: 4096 (order: 2, 16384 bytes)
Inode-cache hash table entries: 2048 (order: 1, 8192 bytes)
Memory: 28056k/31744k available (2448k kernel code, 3688k reserved, 653k data, 108k init, 0k highmem)
Calibrating delay loop... 97.79 BogoMIPS (lpj=48896)
Mount-cache hash table entries: 512
IMEM section size = 0x3244
net_namespace: 788 bytes
NET: Registered protocol family 16
bio: create slab <bio-0> at 0
Sangoma WANPIPE Router v1.1 (c) 1995-2000 Sangoma Technologies Inc.
NET: Registered protocol family 2
IP route cache hash table entries: 1024 (order: 0, 4096 bytes)
TCP established hash table entries: 1024 (order: 1, 8192 bytes)
TCP bind hash table entries: 1024 (order: 0, 4096 bytes)
TCP: Hash tables configured (established 1024 bind 1024)
TCP reno registered
NET: Registered protocol family 1
Realtek GPIO Driver for Flash Reload Default
netlog: listening on port 4660
NTFS driver 2.1.29 [Flags: R/W].
JFFS2 version 2.2. (NAND) © 2001-2006 Red Hat, Inc.
msgmni has been set to 54
alg: No test for stdrng (krng)
io scheduler noop registered
io scheduler deadline registered
io scheduler cfq registered (default)
Serial: 8250/16550 driver, 1 ports, IRQ sharing disabled
serial8250: ttyS0 at MMIO 0x0 (irq = 49) is a 16550A
loop: module loaded
PPP generic driver version 2.4.2
NET: Registered protocol family 24
===============================================================================
init_luna_nor_spi_map: flash map at 0xbd000000
luna spi probe...
luna SPI FLASH G2 driver version 1.00-II: NOR SPI-F... Generic 3B NOR SPI-F detected... done
master->name sflash
cmd: cmdlinepart
4 cmdlinepart partitions found on MTD device sflash
add luna nor spi partition
MTD partitions obtained from kernel command line
Creating 4 MTD partitions on "sflash":
0x000000000000-0x000000080000 : "Boot"
0x000000080000-0x000000200000 : "Config"
0x000000200000-0x000000900000 : "ImageA"
0x000000900000-0x000001000000 : "ImageB"
===============================================================================
u32 classifier
nf_conntrack version 0.5.0 (496 buckets, 1984 max)
ip_tables: (C) 2000-2006 Netfilter Core Team
TCP cubic registered
Realtek SD2-FastPath v1.00beta_2.4.26-uc0
/proc/FastPath created
Realtek MCast FastPath
/proc/mc_FastPath created
NET: Registered protocol family 10
lo: Disabled Privacy Extensions
IPv6 over IPv4 tunneling driver
sit0: Disabled Privacy Extensions
ip6tnl0: Disabled Privacy Extensions
NET: Registered protocol family 17
Bridge firewalling registered
Ebtables v2.0 registered
VFS: Mounted root (cramfs filesystem) readonly on device 31:3.
Freeing unused kernel memory: 108k freed

# num is 10,nothing need to do
ONT>SSP read eqvid :3FE45458ACAA05
Starting Application: 0x00002000, /bin/TimerMgr................Done.
Starting Application: 0x00001000, /bin/LogMgr................Done.
app 0x00007000 stop invalid timer[0]
### Invalid Decimal_Factor, use default 0xdb
### Invalid Settle_Time, use default 0x68
### Invalid P1_Mask, use default 0x5a
Tx has been calibrated!!
Starting Application: 0x00007000, /bin/MiscMgr................Done.
starting to upgrade boot
file[/bootimg/00a9_01_01] length=189440
Flash bootload versin is 3.04.09 
Upgrading bootload versin is 3.04.09 
Upgrading boot is old
Check base image file CRC ... cal_crc (706307a4) ori_crc (706307a4) Success.
Mount Backup[/dev/mtdblock2] as cramfs...Success.
major ID[0xa9] minor ID[0x1]
cp: can't stat '/mnt/rwdir2/sys.log': No such file or directory
mv: can't create '/mnt/rwdir2/sys.log': Read-only file system
cp: can't stat '/mnt/rwdir2/sys.log': No such file or directory
mv: can't create '/mnt/rwdir2/sys.log': Read-only file system
receive MMR_MSG_SET_ACTIVE_IMAGE_ID 
receive MMR_MSG_SET_COMMIT_IMAGE_ID 
Starting Application: 0x00003000, /bin/MecMgr................Done.
Starting Application: 0x00004000, /bin/PonMgr................Done.
------>NET open default xml failedVOS_RegisterEventListener:2672, Init event socket 9
vos_OnKernelEvent:2593, Start receving event on socket 9
Starting Application: 0x00009000, /bin/NetMgr................Done.
Starting Application: 0x00005000, /bin/EthMgr................Done.
Starting Application: 0x0000c000, /bin/WebMgr................Done.
Laser TX OFF
Laser TX ON

INFO: ALL APPs are ready.
Preparing stress ...

ONT>
```
After load finishes, first step is to enable the login prompt, and login credentials are `root:huigu309`. These 2 resources help:
- https://hack-gpon.org/ont-nokia-g-010g-p
- https://forum.openwrt.org/t/help-for-how-to-root-the-g-010g-p-via-usb-ttl/115656

```markdown
ONT>enable
#ONT>login
User name:root
Password:
```
![nokia Untitled1](https://github.com/g1vi/AllTheWriteUps/assets/120142960/fc778537-1436-4024-a2c6-386425281d2c)

After entering credentials the screen is cleared and a new the prompt appears. In the new prompt, type `help` for a list of possible input commands.
```markdown
#ONT>help
  Description: CLI Root
    +traffic             Service CLI menu    
    +system              System CLI menu
```
Spawn a Linux shell by typing `system` and then `shell`

![nokia Untitled2](https://github.com/g1vi/AllTheWriteUps/assets/120142960/0dc2c1ec-35c6-4795-babf-b71cfeccfd8f)

From this shell we can further enumerate the `busybox` version, and view its allowed commands.
```markdown
#ONT/system/shell>cd /bin

#ONT/system/shell>ls
Console         dnsdomainname   klogd           pwd
EthMgr          dropbear        ln              qc
GponCLI         dsp             loadconfig      reboot
GponSLID        echo            login           rm
LogMgr          ecmh            lp              rmdir
MecMgr          egrep           lpadmin         route
MiscMgr         expr            lpstat          run-parts
NetMgr          factory         ls              saveconfig
PonMgr          false           mReport         sed
ShowStatus      fgrep           mdev            sh
Ssp             flash           mfcv6d          sleep
TimerMgr        flatfsd         miniupnpd       slogd
WebMgr          ftp             mkdir           startup
arp             ftpd            mknod           stat
ash             fuser           mktemp          stty
brctl           grep            mount           sync
busybox         gunzip          mountpoint      tar
cat             gzip            mpoactl         telnetd
catv            halt            mpoad           tftpd
chat            hostname        msh             touch
chgrp           ifconfig        mv              traceroute
chmod           inetd           netstat         true
chown           ip              nice            udhcpd
config          ipaddr          ntpclient       umount
configd         iplink          parallel        uname
cp              iproute         pidof           usleep
date            iprule          ping            vi
dd              iptables        ping6           vsntp
df              iptables-batch  poweroff        zcat
diag            iptunnel        printenv
dmesg           kill            ps

#ONT/system/shell>./busybox
BusyBox v1.18.4 (2017-03-15 17:29:57 CST) multi-call binary.
Copyright (C) 1998-2009 Erik Andersen, Rob Landley, Denys Vlasenko
and others. Licensed under GPLv2.
See source distribution for full notice.

Usage: busybox [function] [arguments]...
   or: busybox --list[-full]
   or: function [arguments]...

        BusyBox is a multi-call binary that combines many common Unix
        utilities into a single executable.  Most people will create a
        link to busybox for each function they wish to use and BusyBox
        will act like whatever it was invoked as.

Currently defined functions:
        [, [[, ar, arp, arping, ash, awk, basename, bunzip2, bzcat, bzip2, cal,
        cat, catv, chgrp, chmod, chown, cksum, cmp, comm, cp, cut, date, dd,
        df, dirname, dmesg, dnsdomainname, dos2unix, du, echo, egrep, env,
        expand, expr, false, fgrep, fold, free, ftpget, ftpput, grep, gunzip,
        gzip, halt, head, hostid, hostname, id, ifconfig, ifdown, ifup, init,
        insmod, install, ip, ipaddr, ipcs, iplink, iproute, iprule, iptunnel,
        kill, killall, killall5, length, linuxrc, ln, logname, logread,
        losetup, ls, lsmod, lspci, lsusb, md5sum, mkdir, mkfifo, mknod, mktemp,
        modinfo, modprobe, mount, mountpoint, mv, netstat, nice, nohup, od,
        pidof, ping, ping6, poweroff, printenv, printf, ps, pwd, readlink,
        realpath, reboot, rm, rmdir, rmmod, route, run-parts, sed, seq, sh,
        sha1sum, sha256sum, sha512sum, sleep, sort, split, stat, stty, sum,
        sync, sysctl, syslogd, tac, tail, tar, tee, test, tftp, top, touch, tr,
        traceroute, traceroute6, true, tty, udhcpc, umount, uname, unexpand,
        uniq, unix2dos, usleep, uudecode, uuencode, vconfig, vi, wget, zcat
```
The defined functions are limited, but enough to continue enumerating the system.
```markdown
#ONT/system/shell>uname -a
Linux (none) 2.6.30.9-cig-sfu-1 #1 Fri Dec 11 11:48:11 CST 2015 rlx GNU/Linux
```
And the network config.
```markdown
#ONT/system/shell>ifconfig
eth0      Link encap:Ethernet  HWaddr A4:81:7A:CB:72:2C  
          inet6 addr: fe80::a681:7aff:fecb:722c/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:11 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:866 (866.0 B)
          Interrupt:26 Base address:0x2000 

lan0      Link encap:Ethernet  HWaddr A4:81:7A:CB:72:2C  
          inet addr:192.168.100.1  Bcast:192.168.100.255  Mask:255.255.255.0
          inet6 addr: fe80::a681:7aff:fecb:722c/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:468 (468.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```
A full `busybox` version will provide more flexibility. The the binary can be downloaded from here: https://www.busybox.net/downloads/binaries/1.18.4/busybox-mips<br>
<br>
Good news are the currently installed `busybox` allows to run `wget`, so we can transfer files from our Kali machine. Also, a writable directory is needed as destination, for example this one `/var/tmp`
```markdown
#ONT/system/shell>cd /var/tmp

#ONT/system/shell>wget http://192.168.100.5/busybox-mips
Connecting to 192.168.100.5 (192.168.100.5:80)
busybox-mips         100% |***********************************************************************|  1506k 00:00:00 ETA

#ONT/system/shell>chmod +x busybox-mips

#ONT/system/shell>./busybox-mips
BusyBox v1.18.4 (2011-04-04 19:31:40 CDT) multi-call binary.
Copyright (C) 1998-2009 Erik Andersen, Rob Landley, Denys Vlasenko
and others. Licensed under GPLv2.
See source distribution for full notice.

Usage: busybox [function] [arguments]...
   or: busybox --list[-full]
   or: function [arguments]...

        BusyBox is a multi-call binary that combines many common Unix
        utilities into a single executable.  Most people will create a
        link to busybox for each function they wish to use and BusyBox
        will act like whatever it was invoked as.

Currently defined functions:
        [, [[, acpid, add-shell, addgroup, adduser, adjtimex, arp, arping, ash, awk, base64, basename, beep, blkid, blockdev, bootchartd, brctl, bunzip2, bzcat, bzip2, cal, cat, catv, chat, chattr, chgrp, chmod, chown, chpasswd, chpst, chroot, chrt, chvt, cksum, clear, cmp, comm, cp, cpio, crond, crontab, cryptpw, cttyhack, cut, date, dc, dd, deallocvt, delgroup, deluser, depmod, devmem, df, dhcprelay, diff, dirname, dmesg, dnsd, dnsdomainname, dos2unix, du, dumpkmap, dumpleases, echo, ed, egrep, eject, env, envdir, envuidgid, ether-wake, expand, expr, fakeidentd, false, fbset, fbsplash, fdflush, fdformat, fdisk, fgconsole, fgrep, find, findfs, flock, fold, free, freeramdisk, fsck, fsck.minix, fsync, ftpd, ftpget, ftpput, fuser, getopt, getty, grep, gunzip, gzip, halt, hd, hdparm, head, hexdump, hostid, hostname, httpd, hush, hwclock, id, ifconfig, ifdown, ifenslave, ifplugd, ifup, inetd, init, insmod, install, ionice, iostat, ip, ipaddr, ipcalc, ipcrm, ipcs, iplink, iproute, iprule, iptunnel, kbd_mode, kill, killall, killall5, klogd, last, length, less, linux32, linux64, linuxrc, ln, loadfont, loadkmap, logger, login, logname, logread, losetup, lpd, lpq, lpr, ls, lsattr, lsmod, lspci, lsusb, lzcat, lzma, lzop, lzopcat, makedevs, makemime, man, md5sum, mdev, mesg, microcom, mkdir, mkdosfs, mke2fs, mkfifo, mkfs.ext2, mkfs.minix, mkfs.vfat, mknod, mkpasswd, mkswap, mktemp, modinfo, modprobe, more, mount, mountpoint, mpstat, mt, mv, nameif, nbd-client, nc, netstat, nice, nmeter, nohup, nslookup, ntpd, od, openvt, passwd, patch, pgrep, pidof, ping, ping6, pipe_progress, pivot_root, pkill, pmap, popmaildir, poweroff, powertop, printenv, printf, ps, pscan, pwd, raidautorun, rdate, rdev, readahead, readlink, readprofile, realpath, reboot, reformime, remove-shell, renice, reset, resize, rev, rm, rmdir, rmmod, route, rpm, rpm2cpio, rtcwake, run-parts, runlevel, runsv, runsvdir, rx, script, scriptreplay, sed, sendmail, seq, setarch, setconsole, setfont, setkeycodes, setlogcons, setsid, setuidgid, sh, sha1sum, sha256sum, sha512sum, showkey, slattach, sleep, smemcap, softlimit, sort, split, start-stop-daemon, stat, strings, stty, su, sulogin, sum, sv, svlogd, swapoff, swapon, switch_root, sync, sysctl, syslogd, tac, tail, tar, tcpsvd, tee, telnet, telnetd, test, tftp, tftpd, time, timeout, top, touch, tr, traceroute, traceroute6, true, tty, ttysize, tunctl,  udhcpc, udhcpd, udpsvd, umount, uname, unexpand, uniq, unix2dos, unlzma, unlzop, unxz, unzip, uptime, usleep, uudecode, uuencode, vconfig, vi, vlock, volname, wall, watch, watchdog, wc, wget, which, who, whoami, xargs, xz, xzcat, yes, zcat, zcip

#ONT/system/shell>./busybox-mips whoami
root

#ONT/system/shell>
```
### PORT SCAN
Connect the ONT ethernet port to your laptop with a network wire and launch Kali in bridged network mode in Virtual Box. Then configure an IP for the `eth0` interface in Kali.
```markdwon
ifconfig eth0 192.168.100.5 netmask 255.255.255.0
```
Verify there is connectivity with the ONT, keeping in mind the device configures itself with address `192.168.100.1`
```markdown
> ping 192.168.100.1
PING 192.168.100.1 (192.168.100.1) 56(84) bytes of data.
64 bytes from 192.168.100.1: icmp_seq=1 ttl=64 time=2.02 ms
64 bytes from 192.168.100.1: icmp_seq=2 ttl=64 time=1.15 ms
^C
--- 192.168.100.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1012ms
rtt min/avg/max/mdev = 1.147/1.581/2.015/0.434 ms
```
Launch a port scan with `nmap`
```
> export target=192.168.100.1

> nmap $target -p- --min-rate=5000 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-03-21 06:20 EDT
Nmap scan report for 192.168.100.1
Host is up, received user-set (0.0000040s latency).
Not shown: 45821 filtered tcp ports (no-response), 19406 filtered tcp ports (net-unreach), 306 closed tcp ports (conn-refused)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
23/tcp open  telnet  syn-ack
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 31.77 seconds
# enumerate the open ports
nmap $target -p23,80 -sV -sC -Pn -vv -n
Starting Nmap 7.93 ( https://nmap.org ) at 2024-03-21 07:03 EDT
Nmap scan report for 192.168.100.1
Host is up, received user-set (0.0014s latency).
Scanned at 2024-03-21 07:03:37 EDT for 53s

PORT   STATE SERVICE REASON  VERSION
23/tcp open  telnet  syn-ack
| fingerprint-strings: 
|   GenericLines: 
|     Login as: 
|     Login as: Login as:
|   GetRequest: 
|     Login as: GET / HTTP/1.0
|     Password:
|   Help: 
|     Login as: HELP
|     Password:
|   NCP: 
|     Login as: DmdT^@^@^@
|     ^@^@^@^A^@^@^@^@^@
|   NULL: 
|     Login as:
|   RPCCheck: 
|     Login as: 
|     ^@^@(r
|   SIPOptions: 
|     Login as: OPTIONS sip:nm SIP/2.0
|     Via: SIP/2.0/TCP nm;branch=foo
|     From: <sip:nm@nm>;tag=root
|     <sip:nm2@nm2>
|     Call-ID: 50000
|     CSeq: 42 OPTIONS
|     Max-Forwards: 70
|     Content-Length: 0
|     Contact: <sip:nm@nm>
|     Accept: application/sdp
|     Password:
|   tn3270: 
|_    Login as: ^@IBM-3279-4-E
80/tcp open  http    syn-ack GoAhead WebServer
|_http-favicon: Unknown favicon MD5: BD1E09910249CCA3E8EC3B66CE4EA36D
| http-methods: 
|_  Supported Methods: GET HEAD
|_http-title: GPON Home Gateway
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port23-TCP:V=7.93%I=7%D=3/21%Time=65FC140F%P=x86_64-pc-linux-gnu%r(NULL
SF:,19,"\xff\xfd\x01\xff\xfd\x1f\xff\xfd!\xff\xfb\x01\xff\xfb\x03Login\x20
SF:as:\x20")%r(GenericLines,31,"\xff\xfd\x01\xff\xfd\x1f\xff\xfd!\xff\xfb\
SF:x01\xff\xfb\x03Login\x20as:\x20\r\n\r\nLogin\x20as:\x20Login\x20as:\x20
SF:")%r(tn3270,2A,"\xff\xfd\x01\xff\xfd\x1f\xff\xfd!\xff\xfb\x01\xff\xfb\x
SF:03Login\x20as:\x20\^@IBM-3279-4-E\xfb\^Y")%r(GetRequest,35,"\xff\xfd\x0
SF:1\xff\xfd\x1f\xff\xfd!\xff\xfb\x01\xff\xfb\x03Login\x20as:\x20GET\x20/\
SF:x20HTTP/1\.0\r\n\r\nPassword:\x20")%r(RPCCheck,23,"\xff\xfd\x01\xff\xfd
SF:\x1f\xff\xfd!\xff\xfb\x01\xff\xfb\x03Login\x20as:\x20\x80\^@\^@\(r\xfe\
SF:^\]")%r(Help,29,"\xff\xfd\x01\xff\xfd\x1f\xff\xfd!\xff\xfb\x01\xff\xfb\
SF:x03Login\x20as:\x20HELP\r\nPassword:\x20")%r(SIPOptions,102,"\xff\xfd\x
SF:01\xff\xfd\x1f\xff\xfd!\xff\xfb\x01\xff\xfb\x03Login\x20as:\x20OPTIONS\
SF:x20sip:nm\x20SIP/2\.0\r\nVia:\x20SIP/2\.0/TCP\x20nm;branch=foo\r\nFrom:
SF:\x20<sip:nm@nm>;tag=root\r\nTo:\x20<sip:nm2@nm2>\r\nCall-ID:\x2050000\r
SF:\nCSeq:\x2042\x20OPTIONS\r\nMax-Forwards:\x2070\r\nContent-Length:\x200
SF:\r\nContact:\x20<sip:nm@nm>\r\nAccept:\x20application/sdp\r\n\r\nPasswo
SF:rd:\x20")%r(NCP,53,"\xff\xfd\x01\xff\xfd\x1f\xff\xfd!\xff\xfb\x01\xff\x
SF:fb\x03Login\x20as:\x20DmdT\^@\^@\^@\x08\x20\x08\x08\x20\x08\x08\x20\x08
SF:\x08\x20\x08\x08\x20\x08\x08\x20\x08\x08\x20\x08\x08\x20\x08\x08\x20\x0
SF:8\x08\x20\x08\^@\^@\^@\^A\^@\^@\^@\^@\^@");

Nmap done: 1 IP address (1 host up) scanned in 53.92 seconds
```
Navigate to http://192.168.100.1 with Firefox, a login portal appears.

![nokia Untitled3](https://github.com/g1vi/AllTheWriteUps/assets/120142960/6e096890-bcc8-4b3c-8cec-d56c557348a5)

Login with credentials `admin:1234`, the site basically allows to update the PLOAM password, which essentially authenticates the device in the network. Also, when clicking on "More Info" we gather additional info about the device.

![Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/f0a1be30-7118-47be-9e52-adc94b46e1fb)
