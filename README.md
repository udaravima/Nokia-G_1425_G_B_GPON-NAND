# Nokia-G_1425_G_B_GPON-NAND
This tab is to discuss and research on Nokia GPON (Sri Lanka Telecome) SLT .

## Extracted Rom file (SPI_NAND)
* [ROM File Link G1425GB](https://drive.google.com/drive/folders/1jpPFlRSkBw2cDXV-gbNmGeSVzLFAV-jO?usp=sharing)

## Serial Log UART 
 ```
QFP IC
Xtal:1
DDR3 init.
DRAMC init done. 
Calculate size.
DRAM size=512MB
Set new TRFC.
ddr-900

7528DRAMC V1.8 (0)
Press 'x' or 'b' key in 1 secs to enter or skip bootloader upgrade.


EN7528 at Fri May 13 22:20:23 CST 2022 version 1.1 free bootbase

Set SPI Clock to 40 Mhz
spi_nand_probe: mfr_id=0x2c, dev_id=0x24
Using Flash ECC.
Detected SPI NAND Flash : _SPI_NAND_DEVICE_ID_MT29F2G01, Flash Size=0x10000000
bmt pool size: 163 
BMT & BBT Init Success 
tclinux partition size = 0x1900000, real size = 0x1c35e96
tclinux partition size < its real size
tclinux_slave partition size = 0x1900000, real size = 0x524c3ee9
tclinux_slave partition size < its real size
  GE Rext AnaCal Done! (3)(0x23)  
sn: fc42c84f
Press any key in 3 secs to enter boot command mode.
disable serial by default
Initializing cgroup subsys cpuset
Initializing cgroup subsys cpu
Initializing cgroup subsys cpuacct
Linux version 3.18.21 (buildmgr@AONT18) (gcc version 4.6.3 (Buildroot 2015.08.1) ) #7 SMP Thu Dec 29 15:53:54 CST 2022
__l2cmem:0x8078d460, _l2cmem_end:0x8078d460, l2cmem_size:0 Bytes
memsize:432MB
EcoNet EN7528 SOC prom init
memsize=1b000000
+++++ nokia prom ! Filename: arch/mips/econet/prom.c, Line: 700, Func: prom_init 
+++++ nokia pstore only support more than 256M MEM ! Filename: arch/mips/econet/prom.c, Line: 703, Func: prom_init 
bootconsole [early0] enabled
CPU0 revision is: 0001992f (MIPS 1004Kc)
Determined physical RAM map:
 memory: 1abfe000 @ 00002000 (usable)
Wasting 64 bytes for tracking 2 unused pages
MIPS: machine is econet,en751221
Zone ranges:
  Normal   [mem 0x00002000-0x1abfffff]
Movable zone start for each node
Early memory node ranges
  node   0: [mem 0x00002000-0x1abfffff]
Initmem setup node 0 [mem 0x00002000-0x1abfffff]
Detected 3 available secondary CPU(s)
Primary instruction cache 32kB, VIPT, 4-way, linesize 32 bytes.
Primary data cache 32kB, 4-way, PIPT, no aliases, linesize 32 bytes
MIPS secondary cache 256kB, 8-way, linesize 32 bytes.
PERCPU: Embedded 9 pages/cpu @8135d000 s6144 r8192 d22528 u36864
Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 108710
Kernel command line: 
log_buf_len individual max cpu contribution: 4096 bytes
log_buf_len total cpu_extra contributions: 12288 bytes
log_buf_len min size: 16384 bytes
log_buf_len: 32768 bytes
early log buf free: 13896(84%)
PID hash table entries: 2048 (order: 1, 8192 bytes)
Dentry cache hash table entries: 65536 (order: 6, 262144 bytes)
Inode-cache hash table entries: 32768 (order: 5, 131072 bytes)
Writing ErrCtl register=0003e406
Readback ErrCtl register=0003e406
nmi base is 813f4200
allocated 876528 bytes of page_cgroup
please try 'cgroup_disable=memory' option if you don't want memory cgroups
Memory: 422152K/438264K available (7725K kernel code, 819K rwdata, 1692K rodata, 240K init, 599K bss, 16112K reserved)
SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=4, Nodes=1
Hierarchical RCU implementation.
NR_IRQS:64

tc3162_time_init: Init bus timeout and watchdog
set rbus timeout as NMI then enable it
CPU frequency 900.00 MHz
plat_time_init: Entered, mips_timer_ack ptr is [80009910]
 Using 200.000 MHz high precision timer.
r4k_clockevent_init: setup_irq OK, irq is [52]
console [ttyS0] enabled
bootconsole [early0] disabled
```
## Notes
* for me the serial consoles are disabled by default so I am blind half on the bootup.
* recovered passwords ```ONTUSER: SUGAR2A041``` and ```admin: 1234```
* with SSH have limited access to console and shell need a dynamic password which I don't know

### Device info
```
Device Name          G-1425G-B
Vendor               Nokia
Serial Number        ALCLFC42C84F
Hardware Version     3FE49937DAAA
Boot Version         Bootbase1.1-May-13-2022--22:20:24
Software Version     3FE49568IJJL06(1.2204.506)
Chipset              MTK7528H
```
### Rom architecture (with binwalk)
```
+-----------+-----------+-----------+-----------+---------------------------------------------------------------+
| Address   | Address   | Size      | Size      | Description                                                   |
+-----------+-----------+-----------+-----------+---------------------------------------------------------------+
| 0X1BE90   | 114320    | 16752     | 16.4 KB   | SHA256 hash constants, little endian                          |
|           |           |           |           |                                                               |
| 0X20000   | 131072    | 393496    | 384.3 KB  | LZMA compressed data, properties: 0x5D, dictionary size:      |
|           |           |           |           | 8388608 bytes (8 MB), uncompressed size: 279440 bytes (273 KB)|
|           |           |           |           |                                                               |
| 0X80118   | 524568    | 3757750   | 3.6 MB    | LZMA compressed data, properties: 0x5D, dictionary size:      |
|           |           |           |           | 8388608 bytes (8 MB), uncompressed size: 10803200 bytes (10.3 MB)|
|           |           |           |           |                                                               |
| 0X4157CE  | 4282318   | 25822944  | 24.6 MB   | Squashfs filesystem, little endian, version 4.0,              |
|           |           |           |           | compression:lzma, size: 25822944 bytes (24.6 MB), 2728 inodes,|
|           |           |           |           | blocksize: 131072 bytes (128 KB), created: 2022-07-02 02:51:42|
|           |           |           |           |                                                               |
| 0X1CB5EAE | 30105262  | 13935210  | 13.3 MB   | LZMA compressed data, properties: 0x5D, dictionary size:      |
|           |           |           |           | 8388608 bytes (8 MB), uncompressed size: 267584 bytes (261 KB)|
|           |           |           |           |                                                               |
| 0X2A00118 | 44040472  | 3762366   | 3.6 MB    | LZMA compressed data, properties: 0x5D, dictionary size:      |
|           |           |           |           | 8388608 bytes (8 MB), uncompressed size: 10819584 bytes (10.3 MB)|
|           |           |           |           |                                                               |
| 0X2D969D6 | 47802838  | 26240353  | 25.0 MB   | Squashfs filesystem, little endian, version 4.0,              |
|           |           |           |           | compression:lzma, size: 26240353 bytes (25.0 MB), 2829 inodes,|
|           |           |           |           | blocksize: 131072 bytes (128 KB), created: 2022-12-29 08:06:19|
|           |           |           |           |                                                               |
| 0X469CF37 | 74043191  | 13512905  | 12.9 MB   | LZMA compressed data, properties: 0x5D, dictionary size:      |
|           |           |           |           | 8388608 bytes (8 MB), uncompressed size: 268624 bytes (262 KB)|
|           |           |           |           |                                                               |
| 0X5380000 | 87556096  | 180879360 | 172.5 MB  | UBI erase count header, version: 1, EC: 0x76, VID header      |
|           |           |           |           | offset: 0x800 (2 KB), data offset: 0x1000 (4 KB)              |
+-----------+-----------+-----------+-----------+---------------------------------------------------------------+
| file      | 268435456 | ~256MB    |           |  * All the sizes are just calculated by length of file headers|
+-----------+-----------+-----------+-----------+---------------------------------------------------------------+
```
