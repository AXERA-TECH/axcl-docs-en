# FAQ

(spl_download)=
## SPL Flashing

> **This section only applies to PCIe EP device manufacturers. If you purchased a commercial product (such as the [Xinjian® AI Accelerator Card](#m2_card_xinjian)), the SPL has already been pre-flashed in FLASH by the manufacturer, so this step is not needed.**

The EP device's FLASH needs to have the SPL program pre-loaded. When the EP powers on, the chip ROM will first load and run the SPL program from FLASH. SPL performs the following key initialization processes:

1. Initialize DDR
2. PCIe EP mode configuration
   - Configure EP configuration space (Type 0 Header)
   - Configure BAR space and address mapping
   - Initialize and enable LTSSM (Link Training and Status State Machine)
3. Link establishment and speed negotiation with the RC side.

If the SPL program has not been flashed to FLASH, please follow these steps:

- FLASH programmer (recommended)
- AXDL
  - Configure SPL according to hardware BOM, mainly DDR and FLASH.
  - Build **AX650_card_Vxxx.axp** following [SDK Build](#sdk_compile), e.g., *AX650_card_V3.6.2_20250603154858_20250604212603.axp*.
  - Use the Windows flashing tool **AXDL** to flash this axp image.


::: {note}

If flashing with AXDL, the recommended sequence is:

EP device power on  ➡  AXDL download starts  ➡ SYS_RSTN_IN reset

![m.2_rstn](../res/m.2_rstn.png)

:::



## PCIe

### Theoretical Bandwidth

AX650N supports up to PCIe 2.0 x2.

| Version | Encoding | Transfer Rate | X1           | X2         | X4      | X8      | X16     |
| ------- | -------- | ------------- | ------------ | ---------- | ------- | ------- | ------- |
| 1.0     | 8/10b    | 2.5 GT/s      | 250 MB/s     | 500 MB/s   | 1 GB/s  | 2 GB/s  | 4 GB/s  |
| 2.0     | 8/10b    | 5.0 GT/s      | **500 MB/s** | **1 GB/s** | 2 GB/s  | 4 GB/s  | 8 GB/s  |
| 3.0     | 128/130b | 8.0 GT/s      | 1 GB/s       | 2 GB/s     | 4 GB/s  | 8 GB/s  | 16 GB/s |
| 4.0     | 128/130b | 16.0 GT/s     | 2 GB/s       | 4 GB/s     | 8 GB/s  | 16 GB/s | 32 GB/s |
| 5.0     | 128/130b | 32.0 GT/s     | 4 GB/s       | 8 GB/s     | 16 GB/s | 32 GB/s | 64 GB/s |


(faq_lspci_description)=
### Query Devices

**lspci** is a command-line tool for Linux systems that displays information about all PCI buses and devices connected to those buses. Through lspci, users can obtain detailed device information, including vendor ID, device ID, device class, device driver, and configuration space information.

> AXERA PCIe Vendor ID: 0x1F4B, AX650 default Device ID: 0x0650

```bash
$ lspci
0000:01:00.0 Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)
```

::: {important}

The EP device's FLASH has the SPL program pre-loaded. When the EP powers on, SPL initializes DDR, initializes the PCIe configuration space, and enables the LTSSM state machine. If the host **lspci** cannot find the EP device, please check:

1. Has the SPL program been flashed to FLASH? If not, compile and flash SPL following the [SPL Flashing](#spl_download) section. Steps to confirm SPL is running correctly:
   1. Wire the EP device's UART TX/RX/GND to a PC
   2. Power on the EP device. Does the serial port show the "**DPEI**" string (**D**: DDR initialization, **P**: PCIE driver, **EI**: Endpoint initialization)? e.g.: VNONORS**DPEI**. If DPEI appears, SPL is working correctly.

2. Connect the EP device to the host and restart the host.
3. If the above steps still cannot recognize the device, try switching PCIe slots and restarting. If it still cannot be recognized, check the gold fingers and contact hardware engineers for troubleshooting (e.g., measuring PCIe clock).

:::



#### Query PCIe Topology

```bash
$ lspci
00:00.0 Host bridge: Intel Corporation 8th Gen Core Processor Host Bridge/DRAM Registers (rev 07)
00:01.0 PCI bridge: Intel Corporation 6th-10th Gen Core Processor PCIe Controller (x16) (rev 07)
00:02.0 VGA compatible controller: Intel Corporation CoffeeLake-S GT2 [UHD Graphics 630]
01:00.0 PCI bridge: ASMedia Technology Inc. ASM2824 PCIe Gen3 Packet Switch (rev 01)
02:00.0 PCI bridge: ASMedia Technology Inc. ASM2824 PCIe Gen3 Packet Switch (rev 01)
02:02.0 PCI bridge: ASMedia Technology Inc. ASM2824 PCIe Gen3 Packet Switch (rev 01)
02:03.0 PCI bridge: ASMedia Technology Inc. ASM2824 PCIe Gen3 Packet Switch (rev 01)
02:04.0 PCI bridge: ASMedia Technology Inc. ASM2824 PCIe Gen3 Packet Switch (rev 01)
02:06.0 PCI bridge: ASMedia Technology Inc. ASM2824 PCIe Gen3 Packet Switch (rev 01)
02:07.0 PCI bridge: ASMedia Technology Inc. ASM2824 PCIe Gen3 Packet Switch (rev 01)
02:08.0 PCI bridge: ASMedia Technology Inc. ASM2824 PCIe Gen3 Packet Switch (rev 01)
02:0a.0 PCI bridge: ASMedia Technology Inc. ASM2824 PCIe Gen3 Packet Switch (rev 01)
02:0b.0 PCI bridge: ASMedia Technology Inc. ASM2824 PCIe Gen3 Packet Switch (rev 01)
02:0c.0 PCI bridge: ASMedia Technology Inc. ASM2824 PCIe Gen3 Packet Switch (rev 01)
02:0e.0 PCI bridge: ASMedia Technology Inc. ASM2824 PCIe Gen3 Packet Switch (rev 01)
02:0f.0 PCI bridge: ASMedia Technology Inc. ASM2824 PCIe Gen3 Packet Switch (rev 01)
03:00.0 Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)
04:00.0 Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)
05:00.0 Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)
09:00.0 Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)
0a:00.0 Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)
0b:00.0 Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)
0d:00.0 Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)
0e:00.0 Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)

$ lspci -vt
-[0000:00]-+-00.0  Intel Corporation 8th Gen Core Processor Host Bridge/DRAM Registers
           +-01.0-[01-0e]----00.0-[02-0e]--+-00.0-[03]----00.0  Axera Semiconductor Co., Ltd Device 0650
           |                               +-02.0-[04]----00.0  Axera Semiconductor Co., Ltd Device 0650
           |                               +-03.0-[05]----00.0  Axera Semiconductor Co., Ltd Device 0650
           |                               +-04.0-[06]--
           |                               +-06.0-[07]--
           |                               +-07.0-[08]--
           |                               +-08.0-[09]----00.0  Axera Semiconductor Co., Ltd Device 0650
           |                               +-0a.0-[0a]----00.0  Axera Semiconductor Co., Ltd Device 0650
           |                               +-0b.0-[0b]----00.0  Axera Semiconductor Co., Ltd Device 0650
           |                               +-0c.0-[0c]--
           |                               +-0e.0-[0d]----00.0  Axera Semiconductor Co., Ltd Device 0650
           |                               \-0f.0-[0e]----00.0  Axera Semiconductor Co., Ltd Device 0650
           +-02.0  Intel Corporation CoffeeLake-S GT2 [UHD Graphics 630]
```

As shown above:

> ` +-01.0-[01-0e]----00.0-[02-0e]-`

- `01.0` is a PCI bridge device, [01 - 0e] are the PCI bus lines under PCI bridge 01.0, where bus 01 is connected to device 00:01.0
- Device 01:00.0 under bus 01 is also a PCI bridge device, with [02 - 0e] buses underneath, where bus 02 is connected to device 01:00.0 (ASM2824 upstream port)
- Bus 02 has buses 03, 04, 05, 06, 07, 08, 09, 0a, 0b, 0c, 0d, 0e underneath, of which 8 buses are connected to AX650 EP devices, each connected to ASM2824 downstream ports.

![pci-bus-tupo](../res/pci-bus-tupo.png)



#### Query Detailed Device Information

```bash
$ sudo lspci -s 04:00.0 -vvv
04:00.0 Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)
        Subsystem: Axera Semiconductor Co., Ltd Device 0650
        Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0, Cache Line Size: 64 bytes
        Interrupt: pin A routed to IRQ 147
        Region 0: Memory at b6000000 (32-bit, prefetchable) [size=8M]
        Region 1: Memory at b6900000 (32-bit, prefetchable) [size=64K]
        Region 4: Memory at b6800000 (64-bit, prefetchable) [size=1M]
        Capabilities: [40] Power Management version 3
                Flags: PMEClk- DSI- D1+ D2- AuxCurrent=375mA PME(D0+,D1+,D2-,D3hot+,D3cold-)
                Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
        Capabilities: [50] MSI: Enable+ Count=4/4 Maskable+ 64bit+
                Address: 00000000fee00618  Data: 0000
                Masking: 0000000e  Pending: 00000000
        Capabilities: [70] Express (v2) Endpoint, MSI 00
                DevCap: MaxPayload 128 bytes, PhantFunc 0, Latency L0s unlimited, L1 unlimited
                        ExtTag- AttnBtn- AttnInd- PwrInd- RBE+ FLReset- SlotPowerLimit 0.000W
                DevCtl: CorrErr+ NonFatalErr+ FatalErr+ UnsupReq+
                        RlxdOrd+ ExtTag- PhantFunc- AuxPwr- NoSnoop-
                        MaxPayload 128 bytes, MaxReadReq 512 bytes
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkCap: Port #0, Speed 5GT/s, Width x2, ASPM not supported
                        ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp+
                LnkCtl: ASPM Disabled; RCB 64 bytes, Disabled- CommClk+
                        ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
                LnkSta: Speed 5GT/s (ok), Width x1 (downgraded)
                        TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
                DevCap2: Completion Timeout: Not Supported, TimeoutDis+ NROPrPrP- LTR+
                         10BitTagComp- 10BitTagReq- OBFF Not Supported, ExtFmt- EETLPPrefix-
                         EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
                         FRS- TPHComp- ExtTPHComp-
                         AtomicOpsCap: 32bit- 64bit- 128bitCAS-
                DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- LTR+ OBFF Disabled,
                         AtomicOpsCtl: ReqEn-
                LnkCap2: Supported Link Speeds: 2.5-5GT/s, Crosslink- Retimer- 2Retimers- DRS-
                LnkCtl2: Target Link Speed: 5GT/s, EnterCompliance- SpeedDis-
                         Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
                         Compliance De-emphasis: -6dB
                LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete- EqualizationPhase1-
                         EqualizationPhase2- EqualizationPhase3- LinkEqualizationRequest-
                         Retimer- 2Retimers- CrosslinkRes: unsupported
        Capabilities: [100 v2] Advanced Error Reporting
                UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
                UEMsk:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
                UESvrt: DLP+ SDES+ TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+ ECRC- UnsupReq- ACSViol-
                CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr-
                CEMsk:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+
                AERCap: First Error Pointer: 00, ECRCGenCap+ ECRCGenEn- ECRCChkCap+ ECRCChkEn-
                        MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
                HeaderLog: 00000000 00000000 00000000 00000000
        Capabilities: [148 v1] Latency Tolerance Reporting
                Max snoop latency: 71680ns
                Max no snoop latency: 71680ns
        Capabilities: [150 v1] L1 PM Substates
                L1SubCap: PCI-PM_L1.2+ PCI-PM_L1.1+ ASPM_L1.2+ ASPM_L1.1+ L1_PM_Substates+
                          PortCommonModeRestoreTime=55us PortTPowerOnTime=10us
                L1SubCtl1: PCI-PM_L1.2- PCI-PM_L1.1- ASPM_L1.2- ASPM_L1.1-
                           T_CommonMode=0us LTR1.2_Threshold=0ns
                L1SubCtl2: T_PwrOn=10us
        Capabilities: [160 v1] Vendor Specific Information: ID=0002 Rev=4 Len=100 <?>
        Capabilities: [260 v1] Vendor Specific Information: ID=0006 Rev=0 Len=018 <?>
        Capabilities: [278 v1] Physical Resizable BAR
                BAR 4: current size: 1MB, supported: 1MB
                BAR 5: current size: 1MB, supported: 1MB
        Capabilities: [2b8 v1] Virtual Resizable BAR
                BAR 4: current size: 8MB, supported: 1MB 2MB 4MB 8MB
        Kernel driver in use: ax_pcie_dev_host
        Kernel modules: ax_pcie_host_dev
```

Key information:

- *Capabilities: [70] Express (v2) Endpoint, MSI 00*: Indicates this is an EP device. For device 01:00.0 in the topology above, it shows *Upstream Port*, while 02:00.0 - 02:0f.0 show *Downstream port*.
- *LnkCap: Port #0, Speed 5GT/s, Width x2, ASPM not supported*: Device capability is PCIe 2.0 x2, ASPM not supported. This information indicates the device's capability, not the actual negotiated transfer rate.
- *LnkSta: Speed 5GT/s (ok), Width x1 (downgraded)*: The negotiated link speed is PCIe 2.0 x1, i.e., the actual transfer rate.
- BAR information:
  - *Region 0: Memory at b6000000 (32-bit, prefetchable) [size=8M]*
  - *Region 1: Memory at b6900000 (32-bit, prefetchable) [size=64K]*
  - *Region 4: Memory at b6800000 (64-bit, prefetchable) [size=1M]*
- *CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr-*: Link transmission status. When this status is abnormal (+), prioritize checking signal quality and clock.



#### Query Configuration Space Registers

```bash
$ sudo lspci -s 04:00.0 -xxx
04:00.0 Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)
00: 4b 1f 50 06 06 04 11 00 01 00 00 04 10 00 00 00
10: 08 00 00 b6 08 00 90 b6 00 00 00 00 00 00 00 00
20: 0c 00 80 b6 00 00 00 00 00 00 00 00 4b 1f 50 06
30: 00 00 00 00 40 00 00 00 00 00 00 00 0b 01 00 00
40: 01 50 d3 5b 08 00 00 00 00 00 00 00 00 00 00 00
50: 05 70 a5 03 18 06 e0 fe 00 00 00 00 00 00 00 00
60: 0e 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
70: 10 00 02 00 c0 8f 00 00 1f 20 00 00 22 c0 40 00
80: 40 00 12 10 00 00 00 00 00 00 00 00 00 00 00 00
90: 00 00 00 00 10 08 00 00 00 04 00 00 06 00 00 00
a0: 02 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
c0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
d0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```


(faq-comm-fatal)=
### Communication Exceptions

If communication between the host and PCIe device is abnormal, for example:

```bash
[2025-06-06 09:43:55.891][8779][E][channel][send][166]: send 2807 bytes, but 0 bytes sent
[2025-06-06 09:43:55.891][8804][E][pcie][send_data_by_dma][341]: [1-21] recv dma size 614400 is not equal to 9743
[2025-06-06 09:43:55.891][8804][E][channel][send][166]: send 9743 bytes, but 0 bytes sent
```

If a communication exception occurs, it is recommended to follow these steps to check if the PCIe device abnormality caused the communication exception:

1. Check the host's dmesg log for device `dead` messages, e.g.: ` [heartbeat_recv_thread, 578]: device 4: dead!`.

2. Use `sudo lspci -s BDF -vvv` to check if the PCIe link is disconnected. As shown below, Region 0, Region 1, and Region 4 are in **[virtual]** state:

   ```bash
   $ sudo lspci -s 03:00.0 -vvv
   03:00.0 Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)
           Subsystem: Axera Semiconductor Co., Ltd Device 0650
           Control: I/O- Mem- BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
           Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
           Interrupt: pin A routed to IRQ 87
           NUMA node: 0
           Region 0: Memory at e0800000 (32-bit, non-prefetchable) [virtual] [size=8M]
           Region 1: Memory at e1100000 (32-bit, non-prefetchable) [virtual] [size=64K]
           Region 4: Memory at e1000000 (64-bit, non-prefetchable) [virtual] [size=1M]
   ```

| Device Dead (No Heartbeat) | PCIe Link Disconnected | Possible Causes | Solution |
| --- | --- | --- | --- |
| ✅ |  | - Device core application (slave_daemon) crashed (e.g., coredump) | Connect serial port to sub-card for analysis |
| ✅ | ✅ | - Host system sleep<br />- PCIe switch or EP overtemperature<br />- Device system crash (panic) | Described in sections below |



#### Host System Sleep

AX650N does not support ASPM, therefore **the HOST must disable sleep** functionality, otherwise it will cause device disconnection.

From the host's dmesg log, you can see the system entered sleep mode.

```bash
[ 2065.530453] [heartbeat_recv_thread, 578]: device 4: dead!
[ 2086.012066] [heartbeat_recv_thread, 578]: device 4: dead!
[ 2103.682752] PM: suspend entry (deep)
[ 2106.493464] [heartbeat_recv_thread, 578]: device 4: dead!
[ 2108.440554] Filesystems sync: 4.757 seconds
[ 2109.226893] Freezing user space processes
[ 2109.228564] Freezing user space processes completed (elapsed 0.001 seconds)
[ 2109.228567] OOM killer disabled.
[ 2109.228567] Freezing remaining freezable tasks
[ 2109.229577] Freezing remaining freezable tasks completed (elapsed 0.001 seconds)
[ 2109.229616] printk: Suspending console(s) (use no_console_suspend to debug)
[ 2109.231782] serial 00:02: disabled
[ 2109.231849] e1000e: EEE TX LPI TIMER: 00000011
[ 2109.257414] sd 2:0:0:0: [sda] Synchronizing SCSI cache
[ 2109.259622] ata3.00: Entering standby power mode
[ 2109.681897] ACPI: PM: Preparing to enter system sleep state S3
[ 2110.922369] ACPI: PM: Saving platform NVS memory
[ 2110.922403] Disabling non-boot CPUs ...
[ 2110.923601] smpboot: CPU 1 is now offline
[ 2110.925061] smpboot: CPU 2 is now offline
[ 2110.926414] smpboot: CPU 3 is now offline
[ 2110.927791] smpboot: CPU 4 is now offline
[ 2110.929131] smpboot: CPU 5 is now offline
```



#### Overtemperature

The AX650 device hardware reset temperature threshold is Tj **120** degrees. When the operating temperature exceeds the Tj reset temperature of the device or switch, it will cause a reset and device disconnection.

::: {note}

PCIe boards typically use switch chips to bridge multiple EP devices. If the PCIe board's passive cooling hardware design is poor and exceeds the switch chip's junction temperature Tj, it will also cause the entire board to reset.

Please refer to the switch chip documentation for the switch chip's operating junction temperature Tj.

:::



#### System Crash

Refer to the [sysdump section](#faq-sysdump) to capture sysdump for R&D engineer analysis.





### Interrupts

- HOST notification to Device (EP) is not specified in the protocol. In the AX650N solution, the EP's mailbox interrupt (BAR1 inbound) is used.

- Device (EP) notification to HOST uses standard MSI interrupts. AX650N does not support MSI-X interrupts.

  ```bash
  $ cat /proc/interrupts | grep MSI
   143:          0         75          0          0          0          0  IR-PCI-MSI-0000:03:00.0    0-edge      0000:03:00.0
   147:          0          0         70          0          0          0  IR-PCI-MSI-0000:04:00.0    0-edge      0000:04:00.0
   151:          0          0          0          0          0         70  IR-PCI-MSI-0000:05:00.0    0-edge      0000:05:00.0
   155:         70          0          0          0          0          0  IR-PCI-MSI-0000:09:00.0    0-edge      0000:09:00.0
   159:          0         70          0          0          0          0  IR-PCI-MSI-0000:0a:00.0    0-edge      0000:0a:00.0
   163:          0          0         70          0          0          0  IR-PCI-MSI-0000:0b:00.0    0-edge      0000:0b:00.0
   167:          0          0          0         70          0          0  IR-PCI-MSI-0000:0d:00.0    0-edge      0000:0d:00.0
   171:          0          0          0          0         70          0  IR-PCI-MSI-0000:0e:00.0    0-edge      0000:0e:00.0
  ```





## Driver Installation

(ftrivial-auto-var-init=zero)=
### -ftrivial-auto-var-init=zero

![](../res/ftrivial_auto_var_init.png)

The gcc version from `gcc -v` and `cat /proc/version` shows the kernel gcc version is inconsistent:

```bash
$ gcc -v
Using built-in specs.
COLLECT_GCC=gcc
... ...
gcc version 11.4.0 (Ubuntu 11.4.0-1ubuntu1~22.04)

$ cat /proc/version
Linux version 6.8.0-57-generic (buildd@lcy02-amd64-104) (x86_64-linux-gnu-gcc-12 (Ubuntu 12.3.0-1ubuntu1~22.04) 12.3.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #59~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC Wed Mar 19 17:07:41 UTC 2
```

Resolved by upgrading to the kernel's gcc-12:

```bash
$ apt install gcc-12
$ sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 12
```



(gccupdate)=
### GCC Update

The AXCL runtime uses the C++17 standard, so GCC [9.4.0](https://ftp.gnu.org/gnu/gcc/gcc-9.4.0/gcc-9.4.0.tar.gz) or higher is required. The following section uses [UOS 1070 HWE](https://www.chinauos.com/resource/download-professional) as an example to describe how to compile and install gcc 9.4.0 offline.

:::{note}
To avoid polluting the root environment, please operate as a user account.
:::

**Step 1: Install Dependencies**

```bash
$ sudo apt update
$ sudo apt install -y build-essential wget gcc make gawk bison flex libgmp-dev libmpfr-dev libmpc-dev libz-dev
```

**Step 2: Download GCC Source Code**

```bash
$ wget https://ftp.gnu.org/gnu/gcc/gcc-9.4.0/gcc-9.4.0.tar.gz
$ tar -xvf gcc-9.4.0.tar.gz
$ cd gcc-9.4.0
```

**Step 3: Download Dependency Libraries**

Run the script to automatically download dependencies (isl, cloog, etc.). This process depends on network speed, please wait patiently.

```bash
$ ./contrib/download_prerequisites
```

**Step 4: Configure Build Options**

```bash
$ mkdir build && cd build
$ ../configure --prefix=/usr/local/gcc-9.4.0 --enable-threads=posix --disable-checking  --disable-multilib
```

- `--prefix`: Specify installation path (default is `/usr/local`).
- `--disable-multilib`: Disable multilib support (if only 64-bit is needed)

**Step 5: Build and Install**

```bash
$ make -j$(nproc)
$ sudo make install
```

**Step 6: Configure Environment Variables**

```bash
$ echo 'export PATH=/usr/local/gcc-9.4.0/bin:$PATH' >> ~/.bashrc
$ echo 'export LD_LIBRARY_PATH=/usr/local/gcc-9.4.0/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
$ source ~/.bashrc
```

**Step 7: Verify Installation**

```bash
$ gcc -v  # Should show "gcc (GCC) 9.4.0"
```





## Diagnostic Logs

### Device Logs

The [**axcl-smi**](#doc-axcl-smi) tool supports dumping device-side logs (kernel, syslog, and runtime log) to the HOST locally. Usage is as follows:

```bash
$ axcl-smi --help
usage: axcl-smi [<command> [<args>]] [--device] [--version] [--help]
Commands
    log                                     Dump logs from device
        -t[mask], --type=[mask]                 Specifies which logs to dump by a combination (bitwise OR) value of blow:
                                                  -1: all (default) 0x01: daemon 0x02: worker 0x10: syslog 0x20: kernel
        -o[path], --output=[path]               Specifies the path to save dump logs (default: ./)
-d, --device                            Card index [0, connected cards number - 1]
-h, --help                              Show this help menu
```

Example: dump logs from device #0:

```bash
$ axcl-smi log -d 0
[2025-05-19 18:54:58.252][687704][C][log][dump][73]: log dump finished: ./dev3_log_20250519185454.tar.gz
```


(faq-sysdump)=
### sysdump

When [device system crash (panic)](#faq-comm-fatal) occurs, the HOST driver supports dumping device-side DDR for debugging analysis. The following nodes are provided under HOST `/proc/ax_proc/pcie/sysdump` to control and configure sysdump:

| #    | Node                             | Default    | Description                                                  |
| ---- | -------------------------------- | ---------- | ------------------------------------------------------------ |
| 1    | /proc/ax_proc/pcie/sysdump/debug | 0          | 1: Enable sysdump  0: Disable<br />`echo 1 > /proc/ax_proc/pcie/sysdump/debug` |
| 2    | /proc/ax_proc/pcie/sysdump/path  | /opt       | Specify dump file storage path<br />`echo -n /mnt > /proc/ax_proc/pcie/sysdump/path`<br /> **-n** : Remove the `/n` automatically added by echo;<br />Ensure the path directory has sufficient space to store dump files, otherwise some data will be lost) |
| 3    | /proc/ax_proc/pcie/sysdump/size  | 1073741824 | Specify DDR size to dump, default 1GB. Size is decimal, **not recommended to modify**. |



#### Procedure

> [Operation Video](https://github.com/AXERA-TECH/axcl-docs/tree/main/res/sysdump.mp4)

1. Keep the device powered on, force kill (e.g., `killall -9`) host application processes.
2. `sudo su` to switch to root user.
3. Manually unload the host **axcl_host** driver: `modprobe -r axcl_host` or `rmmod axcl_host`
4. Enable sysdump, configure dump path:

```bash
$ echo 1 > /proc/ax_proc/pcie/sysdump/debug
$ echo -n /mnt > /proc/ax_proc/pcie/sysdump/path
```

5. Reload the host **axcl_host** driver: `modprobe axcl_host` or `insmod axcl_host.ko`. During driver loading, it automatically starts dumping the panicked device and reloads device firmware.
6. Provide the sysdump file and **the vmlinux file corresponding to that firmware version** to technical support.

:::{important}

- The above operation of unloading the driver will **reset all devices**, and the driver will restart all devices and dump the abnormal ones.

- If during business execution a specific device restarted abnormally, and you only need to dump the abnormal device data, you need to start the specified abnormal device in the application code (`axclrtRebootDevice`). This process will only dump the data of that specific abnormal device without affecting other running devices.

- Application code calling `axclrtRebootDevice` also needs to configure `/proc/ax_proc/pcie/sysdump`.

- Timestamp: After sysdump, the sysdump file timestamp may be incorrect. The following settings are needed (**this step does not affect the dump data content, not mandatory**):

  - Check if RTC is set to local timezone:

    Use the following command to check if RTC is set to local timezone: `timedatectl | grep RTC`

  - Set RTC to local timezone:

    Use the following command to set RTC to local timezone: `sudo timedatectl set-local-rtc 1`

  - Sync system time to RTC:

    If needed, sync system time to RTC: `sudo hwclock --systohc`

:::



### Device Information

(faq_device_ddr_bw)=

#### DDR Bandwidth

SDK V3.6.1 (inclusive) and later versions support reading device DDR bandwidth via axcl-smi. Steps are as follows:

```bash
$ axcl-smi sh "insmod /soc/ko/ax_perf_monitor.ko && sleep 1 && cat /proc/ax_proc/bw/bw && rmmod ax_perf_monitor" -d 0
```



(faq_device_npu_tilization)=

#### NPU Utilization

- **axcl-smi** can query the average NPU utilization.

- For more detailed per-core NPU utilization, follow these steps:

  ```bash
  $ axcl-smi sh "cat /proc/ax_proc/npu/top" -d 0
  $ axcl-smi sh "cat /proc/ax_proc/npu/top_sum" -d 0
  ```

:::{note}

  If the NPU is idle, the device will return `nputop info is empty!` or `nputop info not updated over xxx ms!`

:::




(faq_axcl_json)=
### axcl.json

AXCL provides the `axcl.json` file to adjust log levels, DMA transfer buffer sizes, etc. Format is as follows:

```json
{
	"log": {
		"host": {
			"path": "/tmp/axcl/axcl_logs.txt",
			"//   ": "0: trace, 1: debug, 2: info, 3: warn, 4: error, 5: critical, 6: off",
			"level": 2
		},
		"device": {
			"//   ": "0: trace, 1: debug, 2: info, 3: warn, 4: error, 5: critical, 6: off",
			"level": 2
		}
	},
    "dma buf size": "0x400000"
}
```

| Keyword      | Default                 | Description                                                  |
| ------------ | ----------------------- | ------------------------------------------------------------ |
| path         | /tmp/axcl/axcl_logs.txt | HOST runtime log file path                                   |
| level        | 2 (info)                | Log level                                                    |
| dma buf size | "0x400000"              | DMA transfer buffer size, default 4MBytes. SDK total DMA transfer buffer size = *dma buf size x 2* |



## Device Memory

### Memory Layout

```
 0x100000000
      |          Linux OS           |   ramdisk   |                    CMM                    |
```

- **DDR base address**: 0x100000000

- **ramdisk**

  - **Partition size** (`rootfs/card/Makefile`):

    ```bash
    $(HOME_PATH)/tools/mkext4fs/make_ext4fs -l 128M $(BUILD_PATH)/out/$(PROJECT)/images/rootfs.ext4 $(BUILD_ROOT_DIR)/rootfs
    $(HOME_PATH)/tools/mkext4fs/make_ext4fs -l 128M -s $(BUILD_PATH)/out/$(PROJECT)/images/rootfs_sparse.ext4 $(BUILD_ROOT_DIR)/rootfs
    ```

  - **Kernel DTS ramdisk configuration** (`kernel/linux/linux-5.15.73/arch/arm64/boot/dts/axera/AX650_card.dts`)

| Field | Description                                                  | Example                                                      |
| ----- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| reg   | <start_addr_hi32  start_addr_lo32  size_hi32  size_lo32>    | `<0x1 0x40000000 0x0 0x8000000>`<br />Start address 0x140000000, Size 0x8000000 (128MB) |
| addr  | <start_addr_hi32  start_addr_lo32>                          | `<0x1 0x40000000>` Start address 0x140000000                 |
| size  | <size_hi32  size_lo32>                                      | `<0x0 0x8000000>` Size 0x8000000 (128MB)                     |

- **Sub-card firmware rootfs download address** (`tools/mkaxp/AX650X_card_pac.xml`)
  Base: ramdisk address

  ```
  <Img flag="1" name="ROOTFS" select="1">
  			<ID>ROOTFS</ID>
  			<Type>CODE</Type>
  			<Block>
  				<Base>0x140000000</Base>
  				<Size>0x0</Size>
  			</Block>
  			<File>rootfs.ext4</File>
  			<Auth algo="0" />
  			<Description>Download ROOTFS image file</Description>
  </Img>
  ```

- **Makefile** (`build/projects/AX650_card.mak`)
  - **OS_MEM**: Total size of OS+ramdisk
  - **CMM_POOL_PARAM**: CMM partition name; flag (= 0); start address; total partition size. Where start address = Linux OS + ramdisk offset address



### Recommended Configuration

#### 4+4 8GB Capacity

| Linux OS | ramdisk |  CMM   |
| :------: | :-----: | :----: |
|  1024MB  |  128MB  | 7040MB |

```
kernel/linux/linux-5.15.73/arch/arm64/boot/dts/axera/AX650_card.dts:
		ramdisk_mem@140000000 {
			compatible = "axera, ramdisk";
			reg = <0x1 0x40000000 0x0 0x8000000>;
			addr = <0x1 0x40000000>;
			size = <0x0 0x8000000>;
			no-map;
		};

build/projects/AX650_card.mak:
# OS:RAMDISK:CMM
OS_MEM         := mem=1152M
# cmm memory config
CMM_POOL_PARAM := anonymous,0,0x148000000,7040M

tools/mkaxp/AX650X_card_pac.xml:
		<Img flag="1" name="ROOTFS" select="1">
			<ID>ROOTFS</ID>
			<Type>CODE</Type>
			<Block>
				<Base>0x140000000</Base>
				<Size>0x0</Size>
			</Block>
			<File>rootfs.ext4</File>
			<Auth algo="0" />
			<Description>Download ROOTFS image file</Description>
		</Img>
```



#### 2+2 4GB Capacity

| Linux OS | ramdisk |  CMM   |
| :------: | :-----: | :----: |
|  1024MB  |  128MB  | 2944MB |

```
kernel/linux/linux-5.15.73/arch/arm64/boot/dts/axera/AX650_card.dts:
		ramdisk_mem@140000000 {
			compatible = "axera, ramdisk";
			reg = <0x1 0x40000000 0x0 0x8000000>;
			addr = <0x1 0x40000000>;
			size = <0x0 0x8000000>;
			no-map;
		};

build/projects/AX650_card.mak:
# OS:RAMDISK:CMM
OS_MEM         := mem=1152M
# cmm memory config
CMM_POOL_PARAM := anonymous,0,0x148000000,2944M

tools/mkaxp/AX650X_card_pac.xml:
		<Img flag="1" name="ROOTFS" select="1">
			<ID>ROOTFS</ID>
			<Type>CODE</Type>
			<Block>
				<Base>0x140000000</Base>
				<Size>0x0</Size>
			</Block>
			<File>rootfs.ext4</File>
			<Auth algo="0" />
			<Description>Download ROOTFS image file</Description>
		</Img>
```



### Software Compatibility for Different DDR Materials

The following describes the design for a single software image to be compatible with different DDR materials (e.g., 2+2 and 4+4 DDR).

:::{note}

- **In the following solutions, Linux OS memory is the SDK default 1024MB, RAMDISK 128MB**.
- The example adds two hardware configurations: board id = 4 and board id = 5.

:::


(faq_hw_board_id)=
#### Hardware Design

- ADC channel connected to **THM_VINS0**.
- board id = 1 and 2 are used by AXERA SDK by default, corresponding to AX650N demo and EVB development boards respectively. It is recommended to start board id from 3.

![hw-board-id](../res/hw-board-id.png)

| board id | ADC DATA[9:0]   | R_Up(R21) | R_Down(R22) | Note            |
| -------- | --------------- | --------- | ----------- | --------------- |
| 0        | 0x000 - 0x020   | NC        | 0           |                 |
| **1**    | **0x020~0x060** | **1.5K**  | **100**     | **AX650N demo** |
| **2**    | **0x060~0x0A0** | **1.4K**  | **200**     | **AX650N_EVB**  |
| 3        | 0x0A0~0x0E0     | 1.3K      | 301         |                 |
| 4        | 0x0E0~0x120     | 1.21K     | 402         |                 |
| 5        | 0x120~0x160     | 1.1K      | 511         |                 |
| 6        | 0x160~0x1A0     | 1K        | 604         |                 |
| 7        | 0x1A0~0x1E0     | 908       | 715         |                 |
| 8        | 0x1E0~0x220     | 806       | 806         |                 |
| 9        | 0x220~0x260     | 715       | 908         |                 |
| 10       | 0x260~0x2A0     | 604       | 1K          |                 |
| 11       | 0x2A0~0x2E0     | 511       | 1.1K        |                 |
| 12       | 0x2E0~0x320     | 402       | 1.21K       |                 |
| 13       | 0x320~0x360     | 301       | 1.3K        |                 |
| 14       | 0x360~0x3A0     | 200       | 1.4K        |                 |
| 15       | 0x3A0~0x3E0     | 100       | 1.5K        |                 |
| 16       | 0x3E0~0x3FF     | 0         | NC          |                 |



#### Software Design

##### DDR and CMM Capacity Configuration

- ***build/projects/AX650_card_board.json*** defines DDR properties and CMM capacity configuration, format:

  ```json
  // Example: board id 4 & board id 5
  {
      "boards": [
          {
              "board_id": 4,
              "ddr_num": 2,
              "rank_num": 2,
              "lpdd4x": 1,
              "cmm": "anonymous,0,0x148000000,7040M"
          },
          {
              "board_id": 5,
              "ddr_num": 1,
              "rank_num": 2,
              "lpdd4x": 1,
              "cmm": "anonymous,0,0x148000000,3072M"
          }
      ]
  }
  ```

  | Field    | Default                       | Description                                    |
  | -------- | ----------------------------- | ---------------------------------------------- |
  | board_id |                               | Board ID, must be configured and unique        |
  | cmm      | anonymous,0,0x148000000,3072M | CMM configuration, refer to AX SYS API doc     |
  | ddr_num  | 2                             | Number of DDR chips                            |
  | lpddr4x  | 1                             | 1: LPDDR4X  0: LPDDR                           |
  | rank_num | 2                             | DDR rank count                                 |

- **SPL**: When building SPL, the ***boot/bl1/tools/load_board_config.py*** script parses ***build/projects/AX650_card_board.json*** to generate ***boot/bl1/driver/ddr/ddr_board_config.h***. After SPL runs, it reads the board id and configures DDR according to ***ddr_board_config.h***.

- **rootfs**: When building rootfs, the ***rootfs/card/scripts/load_board_config.py*** script parses ***build/projects/AX650_card_board.json*** to generate ***rootfs/card/soc/scripts/board_comm.conf***. At startup, it reads the board id and configures CMM according to ***board_comm.conf***.



##### UBOOT

Add board id:

- ***boot/uboot/u-boot-2020.04/arch/arm/include/asm/arch-axera/boot_mode.h***

  ```C
  typedef enum board_type {
      AX650A_Demo = 0,
      AX650N_Demo = 1,
      AX650A_EVB = 3,
      AX650N_EVB = 2,
      // Add or modify board id according to hardware design, Example:
      AX650N_CUSTOM_BOARD4 = 4,
      AX650N_CUSTOM_BOARD5 = 5,
      AX650N_PCIE = 7,
      AX650_BOARD_MAX = 8,
  } board_type_t;
  ```

- ***boot/uboot/u-boot-2020.04/arch/arm/mach-axera/ax650/board.c***

  ```C
  static const char * board_name[AX650_BOARD_MAX] = {
      [AX650A_Demo] = "AX650A_Demo",
      [AX650N_Demo] = "AX650N_Demo",
      [AX650A_EVB] = "AX650A_EVB",
      [AX650N_EVB] = "AX650N_EVB",
      // Add or modify board name, Example:
      [AX650N_CUSTOM_BOARD4] = "AX650N_CUSTOM_BOARD4",
      [AX650N_CUSTOM_BOARD5] = "AX650N_CUSTOM_BOARD5",
      [AX650N_PCIE] = "AX650N_PCIE",
  };
  ```

- ***boot/uboot/u-boot-2020.04/board/axera/ax650_card/pinmux.c***

  ```C

  struct pinmux ax650A_pinmux_tbl[] = {
      [AX650A_Demo] =
          {ax650A_demo_pinmux,
           sizeof(ax650A_demo_pinmux) / sizeof(unsigned int)},
      [AX650N_Demo] =
          {ax650N_demo_pinmux,
           sizeof(ax650N_demo_pinmux) / sizeof(unsigned int)},
      [AX650A_EVB] =
          {ax650A_evb_pinmux,
           sizeof(ax650A_evb_pinmux) / sizeof(unsigned int)},
      [AX650N_EVB] =
          {ax650N_evb_pinmux,
           sizeof(ax650N_evb_pinmux) / sizeof(unsigned int)},

      // Add pinmux table, reusing ax650N_demo_pinmux
      [AX650N_CUSTOM_BOARD4] =
         {ax650N_demo_pinmux,
          sizeof(ax650N_demo_pinmux) / sizeof(unsigned int)},
      [AX650N_CUSTOM_BOARD5] =
         {ax650N_demo_pinmux,
          sizeof(ax650N_demo_pinmux) / sizeof(unsigned int)},
  };

  int pinmux_init(void)
  {
      int index, i;
      int emmc_phy_flag = 0, sd_phy_flag = 0, sdio_phy_flag = 0, rtc_sys_flag = 0;
      index = get_board_id();
      /* If new board id is greater than 8, modify this accordingly */
      if (index < 0 || index > 8)
          return 0;
      ... ...
  }
  ```

- ***boot/uboot/u-boot-2020.04/arch/arm/mach-axera/ax650/ax650.c***

  ```C
  static int pcie_sata_mode_init(void)
  {
      u32 wdata, rdata;
      u32 sata0_spdmode = 0x2;
      u32 sata1_spdmode = 0x2;
      u32 sata2_spdmode = 0x2;
      u32 sata3_spdmode = 0x2;
      int ref_clk_en = 0;

  #ifdef PCIE_SATA_CFG_VIA_BOARDID
      u32 adc_data;
      board_type_t board_id = get_board_id();

      //for RD dailybuild test
      adc_read_boardid(2,&adc_data);
      switch (board_id) {
      case AX650N_Demo:
      case AX650N_CUSTOM_BOARD4:          // New board PCIe CTRL configuration
      case AX650N_CUSTOM_BOARD5:          // New board PCIe CTRL configuration
          pipe_pcs0_model_sel = 2;        // CTRL0 for PCIE EP
          pipe_pcs1_model_sel = 2;        // CTRL1 for PCIE RC
          sata0_spdmode = 0x1;
          sata1_spdmode = 0x1;
          sata2_spdmode = 0x1;
          sata3_spdmode = 0x1;
          break;
      ... ...
  }
  ```



##### KERNEL

- ***kernel/linux/linux-5.15.73/drivers/soc/axera/ax_hwinfo/ax_hwinfo.h***

  ```C
  typedef enum board_type {
      AX650A_Demo = 0,
      AX650N_Demo = 1,
      AX650A_EVB = 3,
      AX650N_EVB = 2,
      // Add or modify board id according to hardware design, Example:
      AX650N_CUSTOM_BOARD4 = 4,
      AX650N_CUSTOM_BOARD5 = 5,
      AX650N_PCIE = 7,
      AX650_BOARD_MAX = 8,
  } board_type_t;
  ```

- ***kernel/linux/linux-5.15.73/drivers/soc/axera/ax_hwinfo/ax_hwinfo.c***

  ```C
  static const char *ax650x_board_id_to_board_name(int id)
  {
      static const char *board_name[AX650_BOARD_MAX] = {
          [AX650A_Demo] = "AX650A_Demo",
          [AX650N_Demo] = "AX650N_Demo",
          [AX650A_EVB] = "AX650A_EVB",
          [AX650N_EVB] = "AX650N_EVB",
          // Add or modify board name, Example:
          [AX650N_CUSTOM_BOARD4] = "AX650N_CUSTOM_BOARD4",
          [AX650N_CUSTOM_BOARD5] = "AX650N_CUSTOM_BOARD5",
          [AX650N_PCIE] = "AX650N_PCIE",
      };

      if (AX650C_CHIP == ax_get_chip_type()) {
          board_name[AX650N_Demo] = "AX650C_Demo";
          board_name[AX650N_PCIE] = "AX650C_PCIE";
      }

      if (id >= 0 && id < AX650_BOARD_MAX) {
          return board_name[id];
      } else {
          return "Unknow board id";
      }
  }
  ```



## Hardware Design

### board id

[Software Compatibility for Different DDR Materials](#faq_hw_board_id)



### Speed-controlled Fan

Recommended: **PWM17** 1.8V/3.3V



### Heartbeat LED

Recommended: **GPIO1_A14**
