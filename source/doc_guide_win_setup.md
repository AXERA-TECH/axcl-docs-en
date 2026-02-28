# Windows Driver Installation (Beta)

## Environment Preparation

### Version Requirements

| Windows | Version |
| ------- | ------- |
| 10      | 22H2    |
| 11      | >= 23H2 |

:::{note}

- Press **Win+R**, type **winver** to check the current Windows version.
- Only 64-bit Windows 10 and 64-bit Windows 11 are supported.
- Press **Win+I** to open **Settings** > **Update & Security** > **Windows Update** to automatically update the Windows version. For Windows 10, it is recommended to use the [Microsoft Update Assistant](https://www.microsoft.com/en-us/software-download/windows10) to update to version 22H2.
- Disable system sleep: Press **Win+I** to open **Settings** > **System** > **Power & Sleep** > set **Sleep** to **Never**.

:::

### Environment Setup

1. It is recommended to disable antivirus software to avoid false positives.
2. Download the VS2022 runtime library **VC_redist.x64.exe** from the [Microsoft download link](https://aka.ms/vs/17/release/vc_redist.x64.exe), and install VC_redist.x64.exe with **administrator privileges**.
3. **Power off** the system before inserting the device into the motherboard. **Do not hot-plug**.
4. After booting, press **Win+R**, type **devmgmt.msc** to open **Device Manager**. Under **Other Devices**, you should see an unknown **Multimedia Video Controller**. Through **Properties**, you can confirm the device VENDOR is 1F4B.
   ![](../res/ax650_win64_drv_uninstall.png)



## Installation

> Both installation and uninstallation require administrator privileges. System sleep must be disabled.
>
> If an older version of AXCL is already installed, please uninstall it first!

1. The Windows release package `axcl_win64_setup_Vx.x.x_yyyymmdd_NOxxxx.exe` will automatically install the driver, dynamic link libraries (including import libraries), executables (such as axcl-smi.exe, etc.), and sample source code. Run the release package with **administrator privileges** and follow the prompts to install.
2. After installation, please **restart the system**.
3. After restarting, press **Win+R**, type **devmgmt.msc** to open **Device Manager**. Under **System Devices**, you should see the **Axera NPU Accelerator Device**, as shown below.
   ![](../res/ax650_win64_drv_installed.png)
4. Navigate to the installation path `AXCL\axcl\out\axcl_win_x64\bin` directory, and run **axcl-smi.exe** in Windows Terminal:
   ![](../res/ax650_win64_smi.png)
5. Done.

> [Installation Video](../res/ax650_win64_install.mp4)

## Uninstallation
  Double-click **uninst.exe** in the installation directory with **administrator privileges** and follow the prompts to uninstall.



## Building

### Dependencies

The AXCL Windows SDK uses the VS2022 C++17 toolchain for compilation. The required dependencies are:

- [Visual Studio Community 2022](https://visualstudio.microsoft.com/vs/), install following the [official documentation](https://learn.microsoft.com/en-us/visualstudio/install/install-visual-studio?view=vs-2022). Required component: **Desktop development with C++**.

  :::{note}

  - VS2022 installation requires a large amount of disk space. Please ensure sufficient disk space is available.
  - Version: v17.14.15

  :::

- [cmake](https://cmake.org/download/), version 3.20 or above

- [ninja](https://github.com/ninja-build/ninja/releases), v1.31.1 recommended. Add the path of ninja.exe to **System Environment Variables > Path**.

- [Python3](https://www.python.org/downloads/)

### Environment Check

```
PS D:\code> cd "C:\Program Files (x86)\Microsoft Visual Studio\Installer"
PS C:\Program Files (x86)\Microsoft Visual Studio\Installer> .\vswhere.exe -latest -property catalog_productDisplayVersion
17.14.15 (September 2025)

PS D:\code> ninja --version
1.13.1
PS D:\code> cmake --version
cmake version 4.1.1

CMake suite maintained and supported by Kitware (kitware.com/cmake).

PS D:\code> python --version
Python 3.11.5
```

### Building Samples

Run **axcl\scripts\build_win64.cmd** to build samples.

```
PS D:\code\axcl\axcl.win64\axcl\scripts> .\build_win64.cmd
Build started at: 16:48:05.96

Detected 12 physical CPU cores
Checking for Ninja build system...
Found Ninja version: 1.13.1
========================================
CMake Windows x64 Build Script
========================================
Parallel Jobs :  12
Build Type    :  Release
Build System  :  ninja
Project Root  :  D:\code\axcl\axcl.win64\axcl
CMake Source  :  D:\code\axcl\axcl.win64\axcl\cmake
Build Dir     :  D:\code\axcl\axcl.win64\axcl\build\out\axcl_win_x64
Install Dir   :  D:\code\axcl\axcl.win64\axcl\out\axcl_win_x64
========================================

Detecting Visual Studio / Build Tools...
Found: D:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsall.bat
Generator: Visual Studio 17 2022

Configuring Visual Studio build environment...
Visual Studio environment configured successfully.
Creating build directory...

Running CMake configuration...
Using Ninja build system
CMAKE command: cmake -G "Ninja" -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX="D:\code\axcl\axcl.win64\axcl\out\axcl_win_x64" "D:\code\axcl\axcl.win64\axcl\cmake" -DCONFIG_BUILD_VERSION="V3.9.0"

-- The C compiler identification is MSVC 19.44.35217.0
-- The CXX compiler identification is MSVC 19.44.35217.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: D:/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.44.35207/bin/Hostx64/x64/cl.exe - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: D:/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.44.35207/bin/Hostx64/x64/cl.exe - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Socket communication disabled on Windows (ZMQ not available)
-- COMPILE_USER : jingxiaoping
-- BUILD_VERSION: V3.9.0
-- BUILD_FFMPEG : OFF
-- BSP PATH: D:/code/axcl/axcl.win64/axcl/3rdparty/bsp
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD - Failed
-- Looking for pthread_create in pthreads
-- Looking for pthread_create in pthreads - not found
-- Looking for pthread_create in pthread
-- Looking for pthread_create in pthread - not found
-- Found Threads: TRUE
-- Configuring done (4.6s)
-- Generating done (0.2s)
-- Build files have been written to: D:/code/axcl/axcl.win64/axcl/build/out/axcl_win_x64

========================================
Building project (Release configuration)...
========================================
Using Ninja build system with 12 parallel jobs...
[96/96] Linking CXX executable out\axcl_ffmpeg_vdec.exe

========================================
Installing to D:\code\axcl\axcl.win64\axcl\out\axcl_win_x64...
========================================
[0/1] Install the project...-- Install configuration: "Release"
-- Installing: D:/code/axcl/axcl.win64/axcl/out/axcl_win_x64/lib/libcJSON.lib
...

========================================
Build completed successfully
========================================
Build started : 16:48:05.96
Build ended   : 16:48:33.25
Build elapsed : 00:00:27.29
Build Version : V3.9.0
Build Type    : Release
Build System  : ninja
Generator     : Visual Studio 17 2022
Build Dir     : D:\code\axcl\axcl.win64\axcl\build\out\axcl_win_x64
Install Dir   : D:\code\axcl\axcl.win64\axcl\out\axcl_win_x64

Binaries and libraries have been installed to:
  D:\code\axcl\axcl.win64\axcl\out\axcl_win_x64
PS D:\code\axcl\axcl.win64\axcl\scripts>
```



## Application Development

- It is recommended to use the **msvc17** (i.e., VS2022) build toolchain. MinGW is not recommended.
- C++ standard: 17
- Runtime library linking: /MD or /MDd. /MT and /MTd are not recommended.

### Directory Structure

#### Driver

- Driver axcl_pcie.sys: C:\Windows\System32\drivers
- Device firmware and naming path: C:\\Windows\\System32\\drivers\ax650_card.pac  # Fixed path and filename

#### AXCL

```
axcl
   |- 3rdparty                           # Third-party libraries
   |- build                              # Build output directory
   |- cmake                              # CMake files
   |- out                                # Build install directory
        |- axcl_win_x64
                     |- bin              # axcl-smi.exe, samples, and dynamic libraries
                     |- drv              # Driver files
                     |- include          # Header files
                     |- lib              # Dynamic library import libraries
   |- sample                             # Sample source code
   |- toolkit                            # Utility classes
```



## Auxiliary Tools

> [Windows Toolkit](https://hf-mirror.com/AXERA-TECH/testdata-boxdemo/resolve/main/windows%20develop%20kits.zip)
>
> It is recommended to right-click and copy the link address, then download in a new browser window.

1. Windows **lspci**: [pciutils-3.5.5-win64.zip](https://wiki.aixin-chip.com/download/attachments/220464657/pciutils-3.5.5-win64.zip?version=1&modificationDate=1758526243666&api=v2) (requires administrator privileges)

2. [RWEverything](https://rweverything.com/download/) (similar to Linux devmem)

   - [Solution for RWEverything not working on Win11](https://zhuanlan.zhihu.com/p/681795421): In **Settings**, type "**kernel**" in the search box, open **Core Isolation**, and disable both **Memory Integrity** and **Vulnerable Driver Blocklist**, then restart the computer.
   - If it still doesn't work, add the following to the registry `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\CI\Config`:
     Key: **VulnerableDriverBlocklistEnable**, **REG_DWORD**, Value: **1**

3. **md5**: `certutil.exe -hashfile yourfile MD5`

4. Disable driver signature verification (**for driver debugging only**)

   ```
   bcdedit /set testsigning on
   bcdedit /set nointegritychecks on
   ```



## FAQ

To be supplemented.
