# FFmpeg

## Environment Preparation

AXCL_FFMPEG dynamic libraries are stored in `/usr/lib/axcl/ffmpeg`, and AXCL_FFMPEG executables are stored in `/usr/bin/axcl/ffmpeg`.

Before running ffmpeg, you need to set the dynamic library search path with the following environment variable:
```sh
export LD_LIBRARY_PATH="/usr/lib/axcl/ffmpeg:$LD_LIBRARY_PATH";
```



## How to Rebuild FFmpeg?

The SDK FFmpeg is developed based on version 7.1 and provides pre-built .so and ffmpeg binaries that can be directly linked and run.

If you need to rebuild FFmpeg, follow these steps:

1.  Download FFmpeg-n7.1.tar.gz from GitHub, [click here to download](https://github.com/FFmpeg/FFmpeg/releases/tag/n7.1). Copy FFmpeg-n7.1.tar.gz to the `axcl/3rdparty/ffmpeg` directory.

2.  Extract

   ```bash
   tar -zxvf FFmpeg-n7.1.tar.gz
   ```

3.  Patch

   ```bash
   patch -p3 < FFmpeg-n7.1.patch
   ```

4. Build

   - arm64

     ```bash
     cd axcl/3rdparty/ffmpeg
     make host=arm64 clean all install
     ```

     Output file paths:

     ```
     lib: axcl/out/axcl_linux_arm64/lib/ffmpeg
     bin: axcl/out/axcl_linux_arm64/bin/ffmpeg
     ```

   - x86

     ```bash
     cd axcl/3rdparty/ffmpeg
     make host=x86 clean all install
     ```

     Output file paths:

     ```
     lib: axcl/out/axcl_linux_x86/lib/ffmpeg
     bin: axcl/out/axcl_linux_x86/bin/ffmpeg
     ```
