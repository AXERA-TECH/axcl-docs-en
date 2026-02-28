(doc-axcl-smi)=
# AXCL-SMI

## Overview

AXCL-SMI (System Management Interface) is a tool for device information collection and device configuration. It supports the following device information collection:

- Hardware device model
- Firmware version
- Driver version
- Device utilization
- Memory usage
- Device chip junction temperature
- Other information



## Usage

### Quick Start

After correctly installing the AXCL driver package, AXCL-SMI is installed and ready to use. Simply run `axcl-smi` to display the following:

```bash
$ axcl-smi
+------------------------------------------------------------------------------------------------+
| AXCL-SMI  V2.18.0                                                              Driver  V2.18.0 |
+-----------------------------------------+--------------+---------------------------------------+
| Card  Name                     Firmware | Bus-Id       |                          Memory-Usage |
| Fan   Temp                Pwr:Usage/Cap | CPU      NPU |                             CMM-Usage |
|=========================================+==============+=======================================|
|    0  AX650N                    V2.18.0 | 0001:81:00.0 |                181 MiB /      954 MiB |
|   --   52C                      -- / -- | 3%        0% |                 22 MiB /     3072 MiB |
+-----------------------------------------+--------------+---------------------------------------+

+------------------------------------------------------------------------------------------------+
| Processes:                                                                                     |
| Card      PID  Process Name                                                   NPU Memory Usage |
|================================================================================================|
|    0      763  /opt/bin/axcl/axcl_run_model                                            160 KiB |
+------------------------------------------------------------------------------------------------+
```

**Field Description**

| Field            | Description                                              | Field        | Description                   |
| ---------------- | -------------------------------------------------------- | ------------ | ----------------------------- |
| Card             | Device index number (not the PCIe device number)         | Bus-Id       | Device Bus ID                 |
| Name             | Device name                                              | CPU          | Average CPU utilization        |
| Fan              | Fan speed ratio (not supported)                          | NPU          | Average NPU utilization        |
| Temp             | Device chip junction temperature Tj                      | Memory-Usage | System memory: Used/Total      |
| Firmware         | Device firmware version                                  | CMM-Usage    | Media memory: Used/Total       |
| Pwr: Usage/Cap   | Power consumption (not supported)                       |              |                               |
|                  |                                                          |              |                               |
| PID              | Host process PID                                         |              |                               |
| Process Name     | Host process name                                        |              |                               |
| NPU Memory Usage | CMM memory used by the device NPU                       |              |                               |

:::{note}

Refer to the FAQ sections on [DDR Bandwidth](#faq_device_ddr_bw) and [NPU Utilization](#faq_device_npu_tilization) for detailed DDR and NPU utilization information.

:::



### Help (-h) and Version (-v)

`axcl-smi -h` to query help information

```bash
$ axcl-smi -h
usage: axcl-smi [<command> [<args>]] [--device] [--version] [--help]

axcl-smi System Management Interface V2.18.1

Commands
    info                                    Show device information
        --temp                                  Show SoC temperature
        --mem                                   Show memory usage
        --cmm                                   Show CMM usage
        --cpu                                   Show CPU usage
        --npu                                   Show NPU usage
    proc                                    cat device proc
        --vdec                                  cat /proc/ax_proc/vdec
        --venc                                  cat /proc/ax_proc/venc
        --jenc                                  cat /proc/ax_proc/jenc
        --ivps                                  cat /proc/ax_proc/ivps
        --rgn                                   cat /proc/ax_proc/rgn
        --ive                                   cat /proc/ax_proc/ive
        --pool                                  cat /proc/ax_proc/pool
        --link                                  cat /proc/ax_proc/link_table
        --cmm                                   cat /proc/ax_proc/mem_cmm_info
    set                                     Set
        -f[MHz], --freq=[MHz]                   Set CPU frequency in MHz. One of: 1200000, 1400000, 1700000
    log                                     Dump logs from device
        -t[mask], --type=[mask]                 Specifies which logs to dump by a combination (bitwise OR) value of blow:
                                                  -1: all (default) 0x01: daemon 0x02: worker 0x10: syslog 0x20: kernel
        -o[path], --output=[path]               Specifies the path to save dump logs (default: ./)
    sh                                      Execute a shell command
        cmd                                     Shell command
        args...                                 Shell command arguments
-d, --device                            Card index [0, connected cards number - 1]
-v, --version                           Show axcl-smi version
-h, --help                              Show this help menu
```

`axcl-smi -v` to query the AXCL-SMI tool version

```bash
$ axcl-smi -v
AXCL-SMI V2.26.1 BUILD: Feb 13 2025 11:08:47
```



### Options

#### Device ID (-d, --device)

```bash
-d, --device                             Card index [0, connected cards number - 1]
```

`[-d, --device]` specifies the device index, range: [0, number of connected devices - 1], **default is device 0**.

:::{Important}

- SDK V2.25.0 (inclusive) and earlier versions: the device ID refers to the PCIe bus number.
- SDK V2.26.0 (inclusive) and later versions: the device ID refers to the device index number.

:::



### Information Query (info)

`axcl-smi info` displays detailed device information. Supported subcommands are:

| Subcommand | Description                                                                                                                                                           |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| --temp     | Display device chip junction temperature, in millidegrees Celsius (°C x 1000).                                                                                        |
| --mem      | Display detailed device system memory usage.                                                                                                                           |
| --cmm      | Display device media memory usage. For more detailed media memory info, run `axcl-smi sh cat /proc/ax_proc/mem_cmm_info -d xx` (xx is the PCIe device number).         |
| --cpu      | Display device CPU utilization.                                                                                                                                        |
| --npu      | Display device NPU utilization.                                                                                                                                        |

**Example**: Query the media memory usage of device index 0:

```bash
$ axcl-smi info --cmm -d 0
Device ID           : 129 (0x81)
CMM Total           :  3145728 KiB
CMM Used            :    18876 KiB
CMM Remain          :  3126852 kiB
```



### PROC Query (proc)

`axcl-smi proc` queries device module proc information. Supported subcommands are:

| Subcommand | Description                                                |
| ---------- | ---------------------------------------------------------- |
| --vdec     | Query VDEC module proc (`cat /proc/ax_proc/vdec`)          |
| --venc     | Query VENC module proc (`cat /proc/ax_proc/venc`)          |
| --jenc     | Query JENC module proc (`cat /proc/ax_proc/jenc`)          |
| --ivps     | Query IVPS module proc (`cat /proc/ax_proc/ivps`)          |
| --rgn      | Query RGN module proc (`cat /proc/ax_proc/rgn`)            |
| --ive      | Query IVE module proc (`cat /proc/ax_proc/ive`)            |
| --pool     | Query POOL module proc (`cat /proc/ax_proc/pool`)          |
| --link     | Query LINK module proc (`cat /proc/ax_proc/link_table`)    |
| --cmm      | Query CMM module proc (`cat /proc/ax_proc/mem_cmm_info`)   |

**Example**: Query the VDEC proc information of device 0

```bash
$ axcl-smi proc --vdec -d 0
```



### Parameter Settings (set)

`axcl-smi set` configures device information. Supported subcommands are:

| Subcommand            | Description                                                                    |
| --------------------- | ------------------------------------------------------------------------------ |
| -f[MHz], --freq=[MHz] | Set the device CPU frequency. Only supports: 1200000, 1400000, 1700000         |

**Example**: Set the CPU frequency of device index 0 to 1200MHz

```bash
$ axcl-smi set -f 1200000 -d 0
set cpu frequency 1200000 to device 129 succeed.
```



### Download Logs (log)

`axcl-smi log` downloads device log files to the host side. Supported parameters are:

| Parameter                 | Description                                                                                                                                                                                                      |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| -t[mask], --type=[mask]   | Specify the log categories to download. Device-side log categories:<br />-1: All logs<br />0x01: Daemon process<br />0x02: Worker process<br />0x10: syslog<br />0x20: Kernel log<br />Recommended: -1 for all logs |
| -o[path], --output=[path] | Specify the log save path. Supports absolute and relative paths. Default is the current directory. Note: the directory must have write permissions.                                                                |

**Example**: Download all logs from device index 0 and save to the current directory

```bash
$ axcl-smi log -d 0
[2025-02-13 21:07:40.302][1265][C][log][dump][73]: log dump finished: ./dev129_log_20250213210740.tar.gz
```



### Shell Command (sh)

`axcl-smi sh` supports shell commands to query device information, typically used to query device-side module runtime proc information. **Example**: Query CMM information of device index 0

```bash
$ axcl-smi sh cat /proc/ax_proc/mem_cmm_info -d 0
--------------------SDK VERSION-------------------
[Axera version]: ax_cmm V2.26.0_20250211193319 Feb 11 2025 19:52:13 JK
+---PARTITION: Phys(0x180000000, 0x23FFFFFFF), Size=3145728KB(3072MB),    NAME="anonymous"
 nBlock(Max=0, Cur=23, New=0, Free=0)  nbytes(Max=0B(0KB,0MB), Cur=19329024B(18876KB,18MB), New=0B(0KB,0MB), Free=0B(0KB,0MB))  Block(Max=0B(0KB,0MB), Min=0B(0KB,0MB), Avg=0B(0KB,0MB))
   |-Block: phys(0x180000000, 0x180013FFF), cache =non-cacheable, length=80KB(0MB),    name="TDP_DEV"
   |-Block: phys(0x180014000, 0x180014FFF), cache =non-cacheable, length=4KB(0MB),    name="TDP_CMODE3"
   |-Block: phys(0x180015000, 0x180015FFF), cache =non-cacheable, length=4KB(0MB),    name="TDP_CMODE3_CPU"
   |-Block: phys(0x180016000, 0x180029FFF), cache =non-cacheable, length=80KB(0MB),    name="TDP_DEV"
   |-Block: phys(0x18002A000, 0x18002AFFF), cache =non-cacheable, length=4KB(0MB),    name="TDP_CMODE3"
   |-Block: phys(0x18002B000, 0x18002BFFF), cache =non-cacheable, length=4KB(0MB),    name="TDP_CMODE3_CPU"
   |-Block: phys(0x18002C000, 0x180047FFF), cache =non-cacheable, length=112KB(0MB),    name="VGP_DEV"
   |-Block: phys(0x180048000, 0x180048FFF), cache =non-cacheable, length=4KB(0MB),    name="VGP_CMODE3"
   |-Block: phys(0x180049000, 0x180049FFF), cache =non-cacheable, length=4KB(0MB),    name="VGP_CMODE3_CPU"
   |-Block: phys(0x18004A000, 0x1801C9FFF), cache =non-cacheable, length=1536KB(1MB),    name="h26x_ko"
   |-Block: phys(0x1801CA000, 0x180349FFF), cache =non-cacheable, length=1536KB(1MB),    name="h26x_ko"
   |-Block: phys(0x18034A000, 0x18034AFFF), cache =non-cacheable, length=4KB(0MB),    name="h26x_ko"
   |-Block: phys(0x18034B000, 0x18094AFFF), cache =non-cacheable, length=6144KB(6MB),    name="vdec_ko"
   |-Block: phys(0x18094B000, 0x180ACAFFF), cache =non-cacheable, length=1536KB(1MB),    name="jenc_ko"
   |-Block: phys(0x180ACB000, 0x180C4AFFF), cache =non-cacheable, length=1536KB(1MB),    name="jenc_ko"
   |-Block: phys(0x180C4B000, 0x180C4BFFF), cache =non-cacheable, length=4KB(0MB),    name="jenc_ko"
   |-Block: phys(0x180C4C000, 0x180C67FFF), cache =non-cacheable, length=112KB(0MB),    name="VPP_DEV"
   |-Block: phys(0x180C68000, 0x180C68FFF), cache =non-cacheable, length=4KB(0MB),    name="VPP_CMODE3"
   |-Block: phys(0x180C69000, 0x180C69FFF), cache =non-cacheable, length=4KB(0MB),    name="VPP_CMODE3_CPU"
   |-Block: phys(0x180C6A000, 0x181269FFF), cache =non-cacheable, length=6144KB(6MB),    name="vdec_ko"
   |-Block: phys(0x18126A000, 0x18126AFFF), cache =non-cacheable, length=4KB(0MB),    name="GDC_CMDA3"
   |-Block: phys(0x18126B000, 0x18126BFFF), cache =non-cacheable, length=4KB(0MB),    name="GDC_CMDA3_CPU"
   |-Block: phys(0x18126C000, 0x18126EFFF), cache =non-cacheable, length=12KB(0MB),    name="GDC_CMD"

---CMM_USE_INFO:
 total size=3145728KB(3072MB),used=18876KB(18MB + 444KB),remain=3126852KB(3053MB + 580KB),partition_number=1,block_number=23
```

:::{Important}

- If shell command parameters contain `-`, `--`, `>`, etc., use double quotes `"-l"` to wrap the command and parameters as a single string, e.g., `axcl-smi sh "ls -l" -d 0`
- Use shell commands to configure devices with caution

:::



### Reboot

`axcl-smi reboot` resets the specified device and then automatically loads the firmware. Example:

```bash
$ axcl-smi reboot -d 0
Do you want to reboot device 0 ? (y/n): y
```
