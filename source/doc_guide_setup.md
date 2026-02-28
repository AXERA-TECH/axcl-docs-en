# Linux Driver Installation

## Environment Check

### IOMMU

On Intel, AMD, and other X86 CPU platforms, it is not recommended to enable IOMMU. If IOMMU is enabled, performance may be affected. It is recommended to disable it or set it to pass-through mode.

:::{note}

- SDK V3.4.0 (inclusive) and later versions support IOMMU. **For versions prior to SDK V3.4.0, IOMMU must be disabled or set to pass-through mode**.

- How to check if IOMMU is enabled?

  - Check if there are IOMMU groups under `ls /sys/kernel/iommu_groups`. If the directory is empty, IOMMU is not currently supported.
  - Check `dmesg | grep iommu` and `cat /proc/cmdline` for IOMMU configuration mode.

  ```bash
  # iommu enabled
  $ ls /sys/kernel/iommu_groups/
  0  1  10  11  12  13  14  15  16  17  18  19  2  20  21  22  23  24  25  3  4  5  6  7  8  9

  # iommu on
  $ cat /proc/cmdline
  BOOT_IMAGE=/vmlinuz-5.4.18-110-generic root=UUID=6ad21caf-7783-4b20-beab-8587a209f606 ro quiet splash iommu=on quiet splash loglevel=0 resume=UUID=c315bcbf-d4b5-4220-bbb6-9118476be003 security=kysec

  # Pass Through
  $ cat /proc/cmdline
  BOOT_IMAGE=/boot/vmlinuz-6.8.0-57-generic root=UUID=9f8f7b78-45bd-41c7-a2c4-cef9dfd6b6ca ro quiet splash iommu=pt quiet splash vt.handoff=7
  ```

- Configure IOMMU off or pass-through mode

  - Intel CPU

    ```bash
    $ sudo vi /etc/default/grub
    GRUB_CMDLINE_LINUX="quiet splash intel_iommu=off"

    $ sudo update-grub
    $ sudo reboot
    ```

  - AMD CPU

    ```bash
    $ sudo vi /etc/default/grub
    GRUB_CMDLINE_LINUX="quiet splash amd_iommu=off"

    $ sudo update-grub
    $ sudo reboot
    ```

  - Pass Through

    ```bash
    $ sudo vi /etc/default/grub
    GRUB_CMDLINE_LINUX="quiet splash iommu=pt"

    $ sudo update-grub
    $ sudo reboot
    ```

:::


### IRQBalance

The **IRQBalance** service distributes hardware interrupt signals evenly across multiple CPUs, solving the problem of single CPU overload. During sub-card operation, interrupts are reported to the host CPU via MSI. If multiple cards are running busy DMA copy tasks simultaneously on a server without **IRQBalance** enabled, interrupts will be concentrated on a single CPU, causing performance issues — the more cards, the worse the performance.

:::{note}

- Check if the irqbalance package is installed:

  ```bash
  $ dpkg -s irqbalance # debian
  $ rpm -qi irqbalance # redhat
  ```

- Check if irqbalance is enabled:

  ```bash
  $ systemctl status irqbalance
  ```

- Enable irqbalance:

  ```bash
  $ sudo systemctl start irqbalance     # Start the service
  $ sudo systemctl enable irqbalance    # Enable at boot
  ```

:::



### SecureBoot

Driver installation does not currently support secure boot mode. SecureBoot must be disabled in the UEFI BIOS.

```bash
$ sudo mokutil --sb-state
SecureBoot disabled
```



## Supported List

The following table shows the supported host controllers tested with the SDK:

| Host Controller                   | Chip                                | Package                                                                       |
| --------------------------------- | ----------------------------------- | ----------------------------------------------------------------------------- |
| CentOS 9 stream                   | Intel, AMD64                        | axcl_host_x86_64_Vxxx.rpm                                                     |
| Kylin V10 SP1                     | Intel, AMD64, Phytium (ARM64)       | **x86_64**: axcl_host_x86_64_Vxxx.deb<br />**arm64**: axcl_host_aarch64_Vxxx.deb |
| [UOS](#setup_uos)                 | Intel, AMD64                        | axcl_host_x86_64_Vxxx.deb                                                     |
| RK3588                            | ARM64                               | axcl_host_aarch64_Vxxx.deb                                                     |
| [RaspberryPi5](#setup_raspberrypi5) | ARM64                             | axcl_host_aarch64_Vxxx.deb                                                     |
| OpenEuler                         | Intel, AMD64                        | axcl_host_x86_64_Vxxx.rpm                                                     |
| ubuntu 18.04/22.04                | Intel, AMD64                        | axcl_host_x86_64_Vxxx.deb                                                     |
| loongarch64                       | Loongson-3A6000 + debian14 forky    | axcl_host_loongarch64_Vxxx.deb                                                 |

:::{note}

loongarch64 cross-compilation toolchain: https://github.com/loongson/build-tools/releases/tag/2024.11.01

:::

## Install Dependencies

### PCIe Device

- PCIe devices (such as the [Xinjian M.2](#m2_card_xinjian) or boards) have a built-in FLASH (usually NOR). The FLASH must have the [SPL image pre-flashed](#spl_download), which is used for PCIe boot.
- After inserting the PCIe device into the host, use **lspci** to confirm if the device is recognized. Refer to [FAQ: How to Query Devices](#faq_lspci_description).



### make

Check if make is installed with `make -v`:

```bash
$ make -v
GNU Make 4.3
```

### gcc

- Check if gcc is installed with `gcc -v`.
- AXCL runtime is developed based on the C++17 standard, so gcc version **9.4.0** or higher is required. For instructions on upgrading gcc, [refer to the FAQ](#gccupdate).

```bash
$ gcc -v
gcc version 12.2.0 (Debian 12.2.0-14)
```




## Driver Installation

### Packages

| Package Keyword | Architecture | Description                                     |
| --------------- | ------------ | ----------------------------------------------- |
| x86_64          | x86_64       | Intel, AMD                                      |
| aarch64         | ARM64        | Raspberry Pi, RK3588, Phytium (D2000), etc.     |

:::{important}

  **If AXCL driver package has been previously installed, please [uninstall the driver](#uninstalldriver) first.**

:::



### Ubuntu and Other Debian-based Systems

#### Step 1: Hardware Connection

Insert the M.2 or PCIe board into the physical machine.

#### Step 2: System Update

```bash
$ sudo apt update && sudo apt upgrade -y
$ sudo reboot
```

:::{note}

- System updates may take a long time. It is recommended to switch to a local mirror source (e.g., [Tsinghua Mirror](https://mirror.tuna.tsinghua.edu.cn/help/ubuntu/)).

- On Ubuntu 22.04, the kernel gcc version differs from the pre-installed gcc. Install gcc-12:

  ```bash
  $ sudo apt install gcc-12
  $ sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 12
  ```

:::

#### Step 3: Install Linux Headers

```bash
$ sudo apt install -y linux-headers-$(uname -r)
```

#### Step 4: Install AXCL

```bash
$ sudo apt install -y ./xxx.deb
```

:::{important}

For SDK versions prior to V3.4.0, only dpkg installation is supported. Please run `sudo dpkg -i ./xxx.deb`

:::

#### Step 5: Verify Installation

Check package information:

```bash
$ apt show axclhost
```

Run `axcl-smi` to confirm the device is correctly detected:

```bash
$ axcl-smi
+------------------------------------------------------------------------------------------------+
| AXCL-SMI  V3.4.0_20250418095028                                  Driver  V3.4.0_20250418095028 |
+-----------------------------------------+--------------+---------------------------------------+
| Card  Name                     Firmware | Bus-Id       |                          Memory-Usage |
| Fan   Temp                Pwr:Usage/Cap | CPU      NPU |                             CMM-Usage |
|=========================================+==============+=======================================|
|    0  AX650N                     V3.4.0 | 0000:03:00.0 |                153 MiB /      945 MiB |
|   --   38C                      -- / -- | 1%        0% |                 18 MiB /     7040 MiB |
+-----------------------------------------+--------------+---------------------------------------+

+------------------------------------------------------------------------------------------------+
| Processes:                                                                                     |
| Card      PID  Process Name                                                   NPU Memory Usage |
|================================================================================================|
```



### CentOS and Other RedHat-based Systems

#### Step 1: Hardware Connection

Insert the M.2 or PCIe board into the physical machine.

#### Step 2: System Update

```bash
$ sudo yum update -y
$ sudo reboot
```

:::{note}

- System updates may take a long time. It is recommended to switch to a local mirror source (e.g., [Tsinghua Mirror](https://mirror.tuna.tsinghua.edu.cn/help/centos-stream/)).

:::

#### Step 3: Install Linux Headers & Devel

```bash
$ sudo yum install -y kernel-headers-$(uname -r) kernel-devel-$(uname -r)
```

#### Step 4: Install AXCL

```bash
$ sudo yum install -y ./xxx.rpm
```

:::{important}

For SDK versions prior to V3.4.0, only rpm installation is supported. Please run `sudo rpm -Uvh --nodeps xxx.rpm`

:::

#### Step 5: Verify Installation

Check package information:

```bash
$ yum info axcl_host
```

Run `axcl-smi` to confirm the device is correctly detected:

```bash
$ axcl-smi
+------------------------------------------------------------------------------------------------+
| AXCL-SMI  V3.4.0_20250418095028                                  Driver  V3.4.0_20250418095028 |
+-----------------------------------------+--------------+---------------------------------------+
| Card  Name                     Firmware | Bus-Id       |                          Memory-Usage |
| Fan   Temp                Pwr:Usage/Cap | CPU      NPU |                             CMM-Usage |
|=========================================+==============+=======================================|
|    0  AX650N                     V3.4.0 | 0000:03:00.0 |                153 MiB /      945 MiB |
|   --   38C                      -- / -- | 1%        0% |                 18 MiB /     7040 MiB |
+-----------------------------------------+--------------+---------------------------------------+

+------------------------------------------------------------------------------------------------+
| Processes:                                                                                     |
| Card      PID  Process Name                                                   NPU Memory Usage |
|================================================================================================|
```



(setup_raspberrypi5)=

### Raspberry Pi 5

#### Preparation

When installing the AX650 accelerator card on Raspberry Pi 5, you first need to prepare an M.2 HAT+ expansion board. Refer to the [official link](https://www.raspberrypi.com/news/using-m-2-hat-with-raspberry-pi-5/). The official M.2 HAT+ only supports 2230 and 2242 M.2 M Key cards. The AX650 accelerator card is typically 2280, so you may need to consider purchasing a third-party M.2 HAT+ expansion board that supports the 2280 form factor.

:::{Warning}
Depending on the Raspberry Pi hardware batch, you may need to update the Raspberry Pi EEPROM settings. The specific steps are as follows:
:::

Like the BIOS on a PC, EEPROM settings are independent of the OS on the TF card. Flashing the latest Raspberry Pi image or switching image versions does not automatically update the EEPROM settings. First, run update:

```bash
$ sudo apt update && sudo apt full-upgrade
```

Then check the EEPROM version:

```bash
$ sudo rpi-eeprom-update
```

If the date shown is earlier than `December 6, 2023`, run the following command to open the Raspberry Pi Configuration CLI:

```bash
$ sudo raspi-config
```

Under Advanced Options > Bootloader Version, select Latest. Then exit raspi-config using Finish or the ESC key.

Run the following command to update the firmware to the latest version:

```bash
$ sudo rpi-eeprom-update -a
```

Finally, reboot with `sudo reboot`. After rebooting, the EEPROM firmware update is complete.

:::{Warning}
Depending on the Raspberry Pi kernel state, the current instructions are based on a freshly flashed Raspberry Pi system as of November 18, 2024. Users need to determine whether this step is necessary based on their Raspberry Pi system update status.
:::

With the current Raspberry Pi kernel and M.2 HAT+ combination, you may encounter the following limitations:

> - PCIe Device cannot be recognized
> - PCIe MSI IRQ cannot request multiple interrupts

These issues will cause installation failure or the sub-card failing to boot. You need to check and modify the Raspberry Pi 5 `/boot/firmware/config.txt` file.

For third-party compatible M.2 HAT+ products, pay attention to power supply issues. Add the following to config.txt:

```bash
dtparam=pciex1
```

This enables the PCIe function by default. Then add the PCIe device description:

```bash
[all]
dtoverlay=pciex1-compat-pi5,no-mip
```

```bash
$ screenfetch
         _,met$$$$$gg.
      ,g$$$$$$$$$$$$$$$P.        OS: Debian 12 bookworm
    ,g$$P""       """Y$$.".      Kernel: aarch64 Linux 6.6.74+rpt-rpi-2712
   ,$$P'              `$$$.      Uptime: 3h 4m
  ',$$P       ,ggs.     `$$b:    Packages: 1609
  `d$$'     ,$P"'   .    $$$     Shell: bash 5.2.15
   $$P      d$'     ,    $$P     Disk: 13G / 922G (2%)
   $$:      $$.   -    ,d$$'     CPU: ARM Cortex-A76 @ 4x 2.4GHz
   $$\;      Y$b._   _,d$P'      GPU:
   Y$$.    `.`"Y$$$$P"'          RAM: 631MiB / 8064MiB
   `$$b      "-.__
    `Y$$
     `Y$$.
       `$$b.
         `Y$$b.
            `"Y$b._
                `""""

$ cat /boot/firmware/config.txt
# For more options and information see
# http://rptl.io/configtxt
# Some settings may impact device functionality. See link above for details

# Uncomment some or all of these to enable the optional hardware interfaces
#dtparam=i2c_arm=on
#dtparam=i2s=on
#dtparam=spi=on

# Enable audio (loads snd_bcm2835)
dtparam=audio=on
dtparam=pciex1

# Additional overlays and parameters are documented
# /boot/firmware/overlays/README

# Automatically load overlays for detected cameras
camera_auto_detect=1

# Automatically load overlays for detected DSI displays
display_auto_detect=1

# Automatically load initramfs files, if found
auto_initramfs=1

# Enable DRM VC4 V3D driver
dtoverlay=vc4-kms-v3d
max_framebuffers=2

# Don't have the firmware create an initial video= setting in cmdline.txt.
# Use the kernel's default instead.
disable_fw_kms_setup=1

# Run in 64-bit mode
arm_64bit=1

# Disable compensation for displays with overscan
disable_overscan=1

# Run as fast as firmware / board allows
arm_boost=1

[cm4]
# Enable host mode on the 2711 built-in XHCI USB controller.
# This line should be removed if the legacy DWC2 controller is required
# (e.g. for USB device mode) or if USB support is not required.
otg_mode=1

[cm5]
dtoverlay=dwc2,dr_mode=host

[all]
dtoverlay=pciex1-compat-pi5,no-mip
```



After completing the modifications and rebooting, use the `lspci` command to check if the accelerator card is correctly recognized:

```bash
$ lspci
0000:01:00.0 Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)
```

Where `Multimedia video controller: Axera Semiconductor Co., Ltd Device 0650 (rev 01)` is the AX650 accelerator card.

#### Driver Installation

:::{Warning}
The development board requires build support and depends on the gcc, make, patch, and linux-header-$(uname -r) packages. These must be installed beforehand, or network access must be available during installation.
:::

Copy the aarch64 deb package to the Raspberry Pi development board and run the installation command:

```bash
$ sudo apt install axcl_host_aarch64_Vxxx.deb
```

:::{important}

For SDK versions prior to V3.4.0, only dpkg installation is supported. Please run `sudo dpkg -i ./xxx.deb`

:::

Installation will complete quickly. During installation, environment variables are automatically added so that the installed .so libraries and executables are available. Note that if you need the executables to be immediately available, you also need to update the bash terminal environment:

```bash
$ source /etc/profile
```

If you are remotely connected to the board via SSH, you can also reconnect SSH to automatically update (or open a new terminal for local login).

#### Start Sub-card

When the host starts, the sub-card will be started automatically.

#### Uninstall

```bash
$ sudo apt remove axclhost
```

:::{important}

For versions prior to SDK V3.4.0, please run `dpkg -r axclhost` to uninstall.

:::



(setup_uos)=

### UOS

:::{note}
The following example uses UOS Desktop Professional Edition AMD64 [1070 HWE](https://cdimage-download.chinauos.com/hwe/1072/release/uos-desktop-20-professional-hwe-1070-amd64-202412.iso?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NDY3MDc0NDMsImZpbGVfbmFtZSI6Imh3ZS8xMDcyL3JlbGVhc2UvdW9zLWRlc2t0b3AtMjAtcHJvZmVzc2lvbmFsLWh3ZS0xMDcwLWFtZDY0LTIwMjQxMi5pc28ifQ.DUFV5zuXEkzlsRdjp6DysdahDza7Nrh5b_8aHEo0sTY) version.
:::

#### Step 1: Disable Signature Verification

- Go to **Power Management** - **Using Power** - Change **Computer enters standby mode** to **Never**. After entering standby, the PCIe link will be disconnected and cannot be recovered.
- It is recommended to configure High Performance mode.
- Go to **Developer Options** in the <u>**Security Center**</u>, and disable application signature verification.

![](../res/uos_develop_mode.png)
![](../res/uos_off_sign.png)



#### Step 2: System Update

```bash
$ sudo apt update && sudo apt upgrade -y
$ sudo reboot
```

#### Step 3: Install Linux Headers

```bash
$ sudo apt install -y linux-headers-$(uname -r)
```

#### Step 4: Upgrade GCC
UOS 1070 HWE does not have g++ installed by default, and the gcc version is 8.3.0. Since the AXCL runtime requires C++17, the minimum requirement is GCC 9.4.0. Please refer to the [`gcc update`](#gccupdate) section to update GCC to 9.4.0.

#### Step 5: Install AXCL

```bash
$ sudo apt install -y ./xxx.deb
```

#### Step 6: Verify Installation

Check package information:

```bash
$ apt show axclhost
```

Run `axcl-smi` to confirm the device is correctly detected:

```bash
$ axcl-smi
+------------------------------------------------------------------------------------------------+
| AXCL-SMI  V3.4.0_20250418095028                                  Driver  V3.4.0_20250418095028 |
+-----------------------------------------+--------------+---------------------------------------+
| Card  Name                     Firmware | Bus-Id       |                          Memory-Usage |
| Fan   Temp                Pwr:Usage/Cap | CPU      NPU |                             CMM-Usage |
|=========================================+==============+=======================================|
|    0  AX650N                     V3.4.0 | 0000:03:00.0 |                153 MiB /      945 MiB |
|   --   38C                      -- / -- | 1%        0% |                 18 MiB /     7040 MiB |
+-----------------------------------------+--------------+---------------------------------------+

+------------------------------------------------------------------------------------------------+
| Processes:                                                                                     |
| Card      PID  Process Name                                                   NPU Memory Usage |
|================================================================================================|
```



(uninstalldriver)=
## Uninstallation

### Ubuntu and Other Debian-based Systems

```bash
$ sudo apt remove axclhost
```

:::{important}

For versions prior to SDK V3.4.0, please run `dpkg -r axclhost` to uninstall.

:::

### CentOS and Other RedHat-based Systems

```bash
$ sudo yum remove axcl_host
```

:::{important}

For versions prior to SDK V3.4.0, please run `sudo rpm -e axcl_host` to uninstall.

:::



## Driver Upgrade

:::{important}

  **Please [uninstall](#uninstalldriver) the old driver first, then reinstall the new driver.**

:::
