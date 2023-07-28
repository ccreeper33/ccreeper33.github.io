---
title: NAS 内存故障维修记录
date: 2023-07-28 14:18:08
tags: 
- 硬件
- 故障
- nas
categories: NAS 
---

近日忽然发现 NAS 上跑的 qbittorrent 经常报错，表现为总有部分文件块的哈希出错，包括旧文件重新校验也是相同的状况。

使用 dd 和 md5 进行简单的测试：写入全0文件并计算 md5

```bash
dd if=/dev/zero of=test bs=1M count=10000
md5sum test
```

结果可见，由于未知故障，我获得了一块真正意义上的随机存储器（

<!--more-->

![](IOtest.jpg)

初步的怀疑对象是硬盘：

`/pool` 是两块二手 HC320 组成的 btrfs，元数据 RAID1，数据 RAID0。只用于存储 BT/aria 等对安全性需求较低的文件。其中一块在前主人交付前出现了44次 CRC 错误，这也让我将其确定为本次故障的头号嫌疑犯。

```
=== START OF INFORMATION SECTION ===
Model Family:     HGST Ultrastar HC310/320
Device Model:     HGST HUS728T8TALE6L4
Serial Number:    VGJXVAAG
LU WWN Device Id: 5 000cca 0bee94c40
Firmware Version: V8GNW980
User Capacity:    8,001,563,222,016 bytes [8.00 TB]
Sector Sizes:     512 bytes logical, 4096 bytes physical
Rotation Rate:    7200 rpm
Form Factor:      3.5 inches
Device is:        In smartctl database 7.3/5319
ATA Version is:   ACS-2, ATA8-ACS T13/1699-D revision 4
SATA Version is:  SATA 3.2, 6.0 Gb/s (current: 6.0 Gb/s)
Local Time is:    Fri Jul 28 14:35:10 2023 CST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled

SMART Attributes Data Structure revision number: 16
Vendor Specific SMART Attributes with Thresholds:
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  1 Raw_Read_Error_Rate     0x000b   100   100   016    Pre-fail  Always       -       0
  2 Throughput_Performance  0x0005   133   133   054    Pre-fail  Offline      -       92
  3 Spin_Up_Time            0x0007   158   158   024    Pre-fail  Always       -       515 (Average 515)
  4 Start_Stop_Count        0x0012   100   100   000    Old_age   Always       -       2376
  5 Reallocated_Sector_Ct   0x0033   100   100   005    Pre-fail  Always       -       0
  7 Seek_Error_Rate         0x000b   100   100   067    Pre-fail  Always       -       0
  8 Seek_Time_Performance   0x0005   128   128   020    Pre-fail  Offline      -       18
  9 Power_On_Hours          0x0012   098   098   000    Old_age   Always       -       20086
 10 Spin_Retry_Count        0x0013   100   100   060    Pre-fail  Always       -       0
 12 Power_Cycle_Count       0x0032   100   100   000    Old_age   Always       -       96
192 Power-Off_Retract_Count 0x0032   095   095   000    Old_age   Always       -       6396
193 Load_Cycle_Count        0x0012   095   095   000    Old_age   Always       -       6396
194 Temperature_Celsius     0x0002   153   153   000    Old_age   Always       -       39 (Min/Max 15/55)
196 Reallocated_Event_Count 0x0032   100   100   000    Old_age   Always       -       0
197 Current_Pending_Sector  0x0022   100   100   000    Old_age   Always       -       0
198 Offline_Uncorrectable   0x0008   100   100   000    Old_age   Offline      -       0
199 UDMA_CRC_Error_Count    0x000a   200   200   000    Old_age   Always       -       44

SMART Error Log Version: 1
ATA Error Count: 44 (device log contains only the most recent five errors)
        CR = Command Register [HEX]
        FR = Features Register [HEX]
        SC = Sector Count Register [HEX]
        SN = Sector Number Register [HEX]
        CL = Cylinder Low Register [HEX]
        CH = Cylinder High Register [HEX]
        DH = Device/Head Register [HEX]
        DC = Device Command Register [HEX]
        ER = Error register [HEX]
        ST = Status register [HEX]
Powered_Up_Time is measured from power on, and printed as
DDd+hh:mm:SS.sss where DD=days, hh=hours, mm=minutes,
SS=sec, and sss=millisec. It "wraps" after 49.710 days.

Error 44 occurred at disk power-on lifetime: 18677 hours (778 days + 5 hours)
  When the command that caused the error occurred, the device was active or idle.

  After command completion occurred, registers were:
  ER ST SC SN CL CH DH
  -- -- -- -- -- -- --
  84 41 00 00 00 00 00  Error: ICRC, ABRT at LBA = 0x00000000 = 0

  Commands leading to the command that caused the error were:
  CR FR SC SN CL CH DH DC   Powered_Up_Time  Command/Feature_Name
  -- -- -- -- -- -- -- --  ----------------  --------------------
  60 00 30 f0 ef 77 40 00      04:52:19.198  READ FPDMA QUEUED
  60 00 28 f0 f0 77 40 00      04:52:19.183  READ FPDMA QUEUED
  60 00 20 f0 f1 77 40 00      04:52:19.175  READ FPDMA QUEUED
  60 00 18 f0 f2 77 40 00      04:52:19.167  READ FPDMA QUEUED
  60 00 10 f0 f3 77 40 00      04:52:19.160  READ FPDMA QUEUED

SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Extended offline    Aborted by host               80%     20003         -
# 2  Short offline       Completed without error       00%     19952         -
# 3  Short offline       Completed without error       00%     18993         -

SMART Selective self-test log data structure revision number 1
 SPAN  MIN_LBA  MAX_LBA  CURRENT_TEST_STATUS
    1        0        0  Not_testing
    2        0        0  Not_testing
    3        0        0  Not_testing
    4        0        0  Not_testing
    5        0        0  Not_testing
Selective self-test flags (0x0):
  After scanning selected spans, do NOT read-scan remainder of disk.
If Selective self-test is pending on power-up, resume after 0 minute delay.
```

![](disktest.jpg)

然而不论是 smart 测试还是 disk genius 均表明硬盘状况良好。

此时，从分级存储结构考虑，如果硬盘状况良好，那故障只能出现在内存或者cpu了。果不其然，memtest 找到了损坏的内存：

![](memtest.jpg)

在移除损坏的内存后，NAS 完全恢复正常。

