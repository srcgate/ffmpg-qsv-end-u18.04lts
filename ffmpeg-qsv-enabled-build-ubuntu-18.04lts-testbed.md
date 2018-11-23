**Build FFmpeg with Intel's QSV enablement on an Intel-based validation test-bed:**

Build platform: Ubuntu 18.04LTS

**Ensure the platform is up to date:**

`sudo apt update && sudo apt -y upgrade && sudo apt -y dist-upgrade`

**Install baseline dependencies first (inclusive of OpenCL headers+)**

`sudo apt-get -y install autoconf automake build-essential libass-dev libtool pkg-config texinfo zlib1g-dev libva-dev cmake mercurial libdrm-dev libvorbis-dev libogg-dev git libx11-dev libperl-dev libpciaccess-dev libpciaccess0 xorg-dev intel-gpu-tools opencl-headers libwayland-dev xutils-dev ocl-icd-*`

Then add the Oibaf PPA, needed to install the latest development headers for libva:

```
sudo add-apt-repository ppa:oibaf/graphics-drivers
sudo apt-get update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade
```

**To address linker problems down the line with Ubuntu 18.04LTS:**

Referring to this: https://forum.openframeworks.cc/t/ubuntu-unable-to-compile-missing-glx-mesa/29367/2

Create the following symlink as shown:

```
sudo ln -s /usr/lib/x86_64-linux-gnu/libGLX_mesa.so.0 /usr/lib/x86_64-linux-gnu/libGLX_mesa.so
```

**Build the latest libva and all drivers from source:**

**Setup build environment:**

Work space init:

```
mkdir -p ~/vaapi
mkdir -p ~/ffmpeg_build
mkdir -p ~/ffmpeg_sources
mkdir -p ~/bin
```

Build the dependency chain as shown, starting with installing the latest build of [libdrm](https://01.org/linuxgraphics/community/libdrm). This is needed to enable the [cl_intel_va_api_media_sharing](https://www.khronos.org/registry/OpenCL/extensions/intel/cl_intel_va_api_media_sharing.txt) extension, needed when deriving OpenCL device initialization interop in FFmpeg, as illustrated later on in the documentation:

    cd ~/vaapi
    git clone https://anongit.freedesktop.org/git/mesa/drm.git libdrm
    cd libdrm
    ./autogen.sh --prefix=/usr --enable-udev
    time make -j$(nproc) VERBOSE=1
    sudo make -j$(nproc) install
    sudo ldconfig -vvvv

Then proceed with libva:

**1. [Libva :](https://github.com/intel/libva)**

Libva is an implementation for VA-API (Video Acceleration API)

VA-API is an open-source library and API specification, which provides access to graphics hardware acceleration capabilities for video processing. It consists of a main library and driver-specific acceleration backends for each supported hardware vendor. It is a prerequisite for building the VAAPI driver components below.

```
cd ~/vaapi
git clone https://github.com/01org/libva
cd libva
./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu
time make -j$(nproc) VERBOSE=1
sudo make -j$(nproc) install
sudo ldconfig -vvvv
```

**2. [Gmmlib:](https://github.com/intel/gmmlib)**

The Intel(R) Graphics Memory Management Library provides device specific and buffer management for the Intel(R) Graphics Compute Runtime for OpenCL(TM) and the Intel(R) Media Driver for VAAPI.

The component is a prerequisite to the Intel Media driver build step below.

To build this, create a workspace directory within the vaapi sub directory and run the build:

```
mkdir -p ~/vaapi/workspace
cd ~/vaapi/workspace
git clone https://github.com/intel/gmmlib
mkdir -p build
cd build
cmake -DCMAKE_BUILD_TYPE= Release ../gmmlib
make -j$(nproc)
```

Then install the package:

```
sudo make -j$(nproc) install 
```
And proceed.


**3. [Intel Media driver:](https://github.com/intel/media-driver)**

The Intel(R) Media Driver for VAAPI is a new VA-API (Video Acceleration API) user mode driver supporting hardware accelerated decoding, encoding, and video post processing for GEN based graphics hardware, released under the MIT license.

```
cd ~/vaapi/workspace
git clone https://github.com/intel/media-driver
cd media-driver
git submodule init
git pull
mkdir -p ~/vaapi/workspace/build_media
cd ~/vaapi/workspace/build_media
```

Configure the project with cmake:

```
cmake ../media-driver \
-DMEDIA_VERSION="2.0.0" \
-DBS_DIR_GMMLIB=$PWD/../gmmlib/Source/GmmLib/ \
-DBS_DIR_COMMON=$PWD/../gmmlib/Source/Common/ \
-DBS_DIR_INC=$PWD/../gmmlib/Source/inc/ \
-DBS_DIR_MEDIA=$PWD/../media-driver \
-DCMAKE_INSTALL_PREFIX=/usr \
-DCMAKE_INSTALL_LIBDIR=/usr/lib/x86_64-linux-gnu \
-DINSTALL_DRIVER_SYSCONF=OFF \
-DLIBVA_DRIVERS_PATH=/usr/lib/x86_64-linux-gnu/dri
```

Then build the media driver:

```
time make -j$(nproc) VERBOSE=1
```

Then install the project:

```
sudo make -j$(nproc) install VERBOSE=1
```

Add yourself to the video group:

```
sudo usermod -a -G video $USER
```

Now, export environment variables as shown below:

```
LIBVA_DRIVERS_PATH=/usr/lib/x86_64-linux-gnu/dri
LIBVA_DRIVER_NAME=iHD
```

Put that in `/etc/environment`.

**And for the opensource driver (fallback, see notice below):**

Export environment variables as shown below:

```
LIBVA_DRIVERS_PATH=/usr/lib/x86_64-linux-gnu/dri
LIBVA_DRIVER_NAME=i965
```

Put that in `/etc/environment`.


**Notice:** You should ONLY use the `i965` driver for testing and validation only. For QSV-based deployments in production, ensure that `iHD` is the value set for the `LIBVA_DRIVER_NAME` variable, otherwise FFmpeg's QSV-based encoders will fail to initialize. Note that VAAPI is also supported by the `iHD` driver albeit to a limited feature-set, as explained in the last section.

**Fallback for the Intel Opensource VAAPI driver:**

 1. [cmrt](https://github.com/01org/cmrt):

This is the C for Media Runtime GPU Kernel Manager for Intel G45 & HD Graphics family. 
it's a prerequisite for building the [intel-hybrid-driver](https://github.com/01org/intel-hybrid-driver) package on supported platforms.

    cd ~/vaapi
    git clone https://github.com/01org/cmrt
    cd cmrt
    ./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu
    time make -j$(nproc) VERBOSE=1
    sudo make -j$(nproc) install
    

2. [intel-hybrid-driver](https://github.com/01org/intel-hybrid-driver):

This package provides support for WebM project VPx codecs. GPU acceleration
is provided via media kernels executed on Intel GEN GPUs.  The hybrid driver provides the CPU
bound entropy (e.g., CPBAC) decoding and manages the GEN GPU media kernel parameters and buffers.

This package grants access to the VPX-series hybrid decode capabilities on supported hardware configurations,, namely Haswell and Skylake. **Do not build this target on unsupported platforms.**

Related, see [this commit](https://github.com/intel/intel-vaapi-driver/commit/fbbd181c2d60affb3135bd3ad7e87524253d7e9a) regarding the hybrid driver initialization failure on platforms where its' not relevant.

    cd ~/vaapi
    git clone https://github.com/01org/intel-hybrid-driver
    cd intel-hybrid-driver
    ./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu
    time make -j$(nproc) VERBOSE=1
    sudo make -j$(nproc) install
    

3. [intel-vaapi-driver](https://github.com/01org/intel-vaapi-driver):

This package provides the VA-API (Video Acceleration API) user mode driver for Intel GEN Graphics family SKUs.
The current video driver back-end provides a bridge to the GEN GPUs through the packaging of buffers and commands to be sent to the i915 driver for exercising both hardware and shader functionality for video decode, encode, and processing.

it also provides a wrapper to the [intel-hybrid-driver](https://github.com/01org/intel-hybrid-driver) when called up to handle VP8/9 hybrid decode tasks on supported hardware (when configured with the `--enable-hybrid-codec` option as shown below:).

    cd ~/vaapi
    git clone https://github.com/01org/intel-vaapi-driver
    cd intel-vaapi-driver
    ./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu --enable-hybrid-codec
    time make -j$(nproc) VERBOSE=1
    sudo make -j$(nproc) install
    
However, on **Kabylake and newer**, omit this as shown since its' not needed:

```
cd ~/vaapi
git clone https://github.com/intel/intel-vaapi-driver
cd intel-vaapi-driver
./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu 
time make -j$(nproc) VERBOSE=1
sudo make -j$(nproc) install

```

Proceed:

**4. [libva-utils:](https://github.com/intel/libva-utils)**

This package provides a collection of tests for VA-API, such as `vainfo`, needed to validate a platform's supported features (encode, decode & postproc attributes on a per-codec basis by VAAPI entry points information).

```
cd ~/vaapi
git clone https://github.com/intel/libva-utils
cd libva-utils
./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu
time make -j$(nproc) VERBOSE=1
sudo make -j$(nproc) install
```

At this point, issue a reboot:

```
sudo systemctl reboot
```

Then on resume, proceed with the steps below, installing the Intel OpenCL platform (Neo):

**Before you proceed with the iMSDK:**

It is recommended that you build the Intel Neo OpenCL runtime:

**Justification:** This will allow for Intel's MediaSDK OpenCL inter-op back-end to be built. 

**Note:** There's also a Personal Package Archive (PPA) for this that you can add, allowing you to skip the manual build step, as shown:

```
sudo add-apt-repository ppa:intel-opencl/intel-opencl
sudo apt-get update
```

Then install the packages:

    sudo apt install intel-igc-* intel-opencl* intel-cloc intel-cmt-cat

Note that the PPA builds are a bit behind the upstream stack, and as such, these needing the latest version should use the build steps below.

**Install the dependencies for the OpenCL back-end:**

**Build dependencies:**

```
sudo apt-get install ccache flex bison clang-4.0 cmake g++ git patch zlib1g-dev autoconf xutils-dev libtool pkg-config libpciaccess-dev libz-dev
```

Create the project structure:
```
mkdir -p ~/intel-compute-runtime/workspace
```

Within this workspace directory, fetch the sources for the required dependencies:

```
cd ~/intel-compute-runtime/workspace

git clone -b release_40 https://github.com/llvm-mirror/clang clang_source

git clone https://github.com/intel/opencl-clang common_clang

git clone https://github.com/intel/llvm-patches llvm_patches

git clone -b release_40 https://github.com/llvm-mirror/llvm llvm_source

git clone https://github.com/intel/gmmlib gmmlib

git clone https://github.com/intel/intel-graphics-compiler igc

git clone https://github.com/KhronosGroup/OpenCL-Headers khronos

git clone https://github.com/intel/compute-runtime neo

ln -s khronos opencl_headers
```

Create a build directory for the Intel Graphics Compiler under the workspace:

```
mkdir -p ~/intel-compute-runtime/workspace/build_igc
```

Then build:

```
cd ~/intel-compute-runtime/workspace/build_igc

cmake -DIGC_OPTION__OUTPUT_DIR=../igc-install/Release \
    -DCMAKE_BUILD_TYPE=Release -DIGC_OPTION__ARCHITECTURE_TARGET=Linux64 \
    ../igc/IGC

time make -j$(nproc) VERBOSE=1
```

**Recommended:** Generate Debian archives for installation:

```
time make -j$(nproc) package VERBOSE=1
```

Install:

```
sudo dpkg -i *.deb
```

(Ran in the build directory) will suffice.


**To install directly without relying on the package manager:**

On whatever Linux distribution you're on, you can also run:

    sudo make -j$(nproc) install

If you prefer to skip the generated binary artifacts by cpack. This may solve package installation and dependency issues that some of you are encountering.

Then proceed.

**Add LLVM-4.x to system path:**

Append `:/usr/lib/llvm-4.0/bin:$HOME/bin` at the end of the PATH variable in `/etc/environment` and source the file:

```
source /etc/environment
```

**Important:** Either way, make sure that LLVM 4.0 is in your path, as its' required by Intel's OCL runtime.

Next, build and install the `compute runtime project`. Start by creating a separate build directory for it:

```
mkdir -p ~/intel-compute-runtime/workspace/build_icr

cd ~/intel-compute-runtime/workspace/build_icr

cmake -DBUILD_TYPE=Release -DCMAKE_BUILD_TYPE=Release -DSKIP_UNIT_TESTS=1 ../neo

time make -j$(nproc) package VERBOSE=1
```

Then install the deb archives:

```
  sudo dpkg -i *.deb 
```

From the build directory.

**To install directly without relying on the package manager:**

On whatever Linux distribution you're on, you can also run:

    sudo make -j$(nproc) install

If you prefer to skip the generated binary artifacts by cpack. This may solve package installation and dependency issues that some of you are encountering.

**Testing:**

Use clinfo and confirm that the ICD is detected.

Optionally, run [Luxmark](http://www.luxmark.info/) and confirm that Intel Neo's OpenCL platform is detected and usable.

Be aware that Luxmark, among others, require `freeglut`, which you can install by running:

```
sudo apt install freeglut3*
```


**4. Build [Intel's MSDK](https://github.com/Intel-Media-SDK/MediaSDK):**

This package provides an API to access hardware-accelerated video decode, encode and filtering on Intel® platforms with integrated graphics. See the supported platform details below:

**Supported platforms:**

Compared to the open source VAAPI driver, this project supports the following SKU generations:

```
BDW (Broadwell)

SKL (Skylake)

BXT (Broxton) / APL (Apollo Lake)

KBL (Kaby Lake)

CFL (Coffee Lake)

CNL (Cannonlake)

ICL (Ice Lake)

```

## Supported Codecs

| CODEC      | BDW  | SKL  | BXT/APL |   KBL   |   CFL   | CNL  |   ICL   |
|------------|------|------|---------|---------|---------|------|---------|
| H.264      | D/E1 | D/E1 | D/E1/E2 | D/E1/E2 | D/E1/E2 | D/E1 | D/E1/E2 |
| MPEG-2     | D/E1 | D/E1 | D       | D/E1    | D/E1    | D/E1 | D/E1    |
| VC-1       | D    | D    | D       | D       | D       | D    | D       |
| JPEG       | D    | D/E2 | D/E2    | D/E2    | D/E2    | D/E2 | D/E2    |
| VP8        | D    | D    | D       | D       | D       | D/E1 | D/E1    |
| HEVC       |      | D/E1 | D/E1    | D/E1    | D/E1    | D/E1 | D/E1/E2 |
| HEVC 10bit |      |      | D       | D       | D       | D/E1 | D/E1/E2 |
| VP9        |      |      | D       | D       | D       | D    | D/E2    |
| VP9 10bit  |      |      |         | D       | D       | D    | D/E2    |

D  - decoding

E1 - VME based encoding

E2 - Low power encoding

## Supported Video Processing

| Video Processing                             | BDW | SKL | BXT/APL | KBL | CFL | CNL | ICL |
|----------------------------------------------|-----|-----|---------|-----|-----|-----|-----|
| Blending                                     |  Y  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |
| CSC (Color Space Conversion)                 |  Y  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |
| De-interlace                                 |  Y  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |
| De-noise                                     |  Y  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |
| Luma Key                                     |  Y  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |
| Mirroring                                    |  Y  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |
| ProcAmp (brightness,contrast,hue,saturation) |  Y  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |
| Rotation                                     |  Y  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |
| Scaling                                      |  Y  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |
| Sharpening                                   |  Y  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |
| STD/E (Skin Tone Detect & Enhancement)       |  Y  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |
| TCC (Total Color Control)                    |  Y  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |
| Color fill                                   |  Y  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |
| Chroma Siting                                |  N  |  Y  |    Y    |  Y  |  Y  |  Y  |  Y  |

## Known Issues and Limitations

1. Intel(R) Media Driver for VAAPI is recommended to be built against gcc compiler v6.1
or later, which officially supported C++11.

2. SKL: Green or other incorrect color will be observed in output frames when using
YV12/I420 as input format for csc/scaling/blending/rotation, etc. on Ubuntu 16.04 stock
(with kernel 4.10). The issue can be addressed with the kernel patch:
WaEnableYV12BugFixInHalfSliceChicken7 [commit 0b71cea29fc29bbd8e9dd9c641fee6bd75f6827](https://cgit.freedesktop.org/drm-tip/commit/?id=0b71cea29fc29bbd8e9dd9c641fee6bd75f68274)

3. HuC firmware is needed for AVC low power encoding bitrate control, including CBR, VBR, etc. As of now, HuC firmware support is disabled in Linux kernels by default. Please, refer to i915 kernel mode driver documentation to learn how to enable it. Mind that HuC firmware support presents in the following kernels for the specified platforms:
   * APL, KBL: starting from kernel 4.11
   * CFL: starting from kernel 4.15

4. Restriction in implementation of vaGetImage: Source format (surface) should be same with destination format (image).

5. ICL encoding and decoding require special kernel mode driver ([issue#267](https://github.com/intel/media-driver/issues/267)/[PR#271](https://github.com/intel/media-driver/pull/271)), please refer to i915 kernel mode driver documentation for supporting status.

Unless explicitly listed, any platform not in that list should be considered as unsupported. For these platforms, stick to upstream VAAPI. That list will be updated over time. 

**Build steps:**

(a). Fetch the sources into the working directory `~/vaapi`:

```
cd ~/vaapi
git clone https://github.com/Intel-Media-SDK/MediaSDK msdk
cd msdk
git submodule init
git pull
```

(b). Configure the build with GCC:

There are two options here, namely, using Intel's Perl builder configurator, or better still, the Cmake route:

(i). For Apollo Lake:

```
perl tools/builder/build_mfx.pl --cmake=intel64.make.release --target=BXT
```

(ii). For SKL:

```
perl tools/builder/build_mfx.pl --cmake=intel64.make.release 
```

This will build MSDK binaries and MSDK samples.

(c ). Run the build:

```
make -j$(nproc) -C __cmake/intel64.make.release
```

(d). And install:

`cd __cmake/intel64.make.release`

`sudo make -j$(nproc) install`

**Recommended:** With the Cmake route:

If you encounter errors above (with Intel's Perl configurator), use the CMake route below:

```
mkdir -p ~/vaapi/build_msdk
cd ~/vaapi/build_msdk
cmake -DCMAKE_BUILD_TYPE=Release -DENABLE_WAYLAND=ON -DENABLE_X11_DRI3=ON -DENABLE_OPENCL=ON  ../msdk
time make -j$(nproc) VERBOSE=1
sudo make install -j$(nproc) VERBOSE=1
```

CMake will automatically detect the platform you're on and enable the platform-specific hooks needed for a working build.

**Obsolete:** Apply this workaround (no longer needed as its' the default upstream):

```
sudo mkdir -p /opt/intel/mediasdk/include/mfx
sudo cp -vr /opt/intel/mediasdk/include/*.h  /opt/intel/mediasdk/include/mfx
```

Create a library config file for the iMSDK:

```
sudo nano /etc/ld.so.conf.d/imsdk.conf
```

Content:

```
/opt/intel/mediasdk/lib
/opt/intel/mediasdk/plugins
```

Then run:

```
sudo ldconfig -vvvv
```

To proceed.

Confirm that `/opt/intel/mediasdk/lib/pkgconfig/libmfx.pc` and `/opt/mediasdk/lib/pkgconfig/mfx.pc` are populated correctly:

```
Name: mfx
Description: Intel(R) Media SDK Dispatcher
Version: 1.26  ((MFX_VERSION) % 1000)

prefix=/opt/intel/mediasdk
libdir=/opt/intel/mediasdk/lib
includedir=/opt/intel/mediasdk/include
Libs: -L${libdir} -lmfx -lstdc++ -ldl -lva -lstdc++ -ldl -lva-drm -ldrm
Cflags: -I${includedir} -I/usr/include/libdrm

```

When done, issue a reboot:

```
 sudo systemctl reboot

```

**Known Issues and Limitations:**

For validation, consider the following known issues:

1.  SKL: Green or other incorrect color will be observed in output frames when using YV12/I420 as input format for csc/scaling/blending/rotation, etc. on Ubuntu 16.04 stock (with kernel 4.10). The issue can be addressed with the kernel patch: [WaEnableYV12BugFixInHalfSliceChicken7 commit](https://cgit.freedesktop.org/drm-tip/commit/?id=0b71cea29fc29bbd8e9dd9c641fee6bd75f68274).
    
2.  CNL: Functionalities requiring HuC including AVC BRC for low power encoding, HEVC low power encoding, and VP9 low power encoding are pending on the kernel patch for GuC support which is expected in Q1’2018.
    
3.  BXT/APL: Limited validation was performed; product quality expected in Q1’2018.
    

**Build a usable FFmpeg binary with the iMSDK:**

Include extra components as needed:

**(a). Build and deploy nasm:** [Nasm](https://www.nasm.us/) is an assembler for x86 optimizations used by x264 and FFmpeg. Highly recommended or your resulting build may be very slow.

Note that we've now switched away from Yasm to nasm, as this is the current assembler that x265,x264, among others, are adopting.

```
cd ~/ffmpeg_sources
wget wget http://www.nasm.us/pub/nasm/releasebuilds/2.14rc0/nasm-2.14rc0.tar.gz
tar xzvf nasm-2.14rc0.tar.gz
cd nasm-2.14rc0
./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin" 
make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean

```

**(b). Build and deploy libx264 statically:** This library provides a H.264 video encoder. See the [H.264 Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/H.264) for more information and usage examples. This requires ffmpeg to be configured with `--enable-gpl --enable-libx264`.

```
cd ~/ffmpeg_sources
git clone http://git.videolan.org/git/x264.git -b stable
cd x264/
PATH="$HOME/bin:$PATH" ./configure --prefix="$HOME/ffmpeg_build" --enable-static --enable-shared
PATH="$HOME/bin:$PATH" make -j$(nproc) VERBOSE=1
make -j$(nproc) install VERBOSE=1
make -j$(nproc) distclean
```

**(c ). Build and configure libx265:** This library provides a H.265/HEVC video encoder. See the [H.265 Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/H.265) for more information and usage examples.

```
cd ~/ffmpeg_sources
hg clone https://bitbucket.org/multicoreware/x265
cd ~/ffmpeg_sources/x265/build/linux
PATH="$HOME/bin:$PATH" cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="$HOME/ffmpeg_build" -DENABLE_SHARED:bool=off ../../source
make -j$(nproc) VERBOSE=1
make -j$(nproc) install VERBOSE=1
make -j$(nproc) clean VERBOSE=1

```

**(d). Build and deploy the libfdk-aac library:** This provides an AAC audio encoder. See the [AAC Audio Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/AAC) for more information and usage examples. This requires ffmpeg to be configured with `--enable-libfdk-aac` (and `--enable-nonfree` if you also included `--enable-gpl`).

```
cd ~/ffmpeg_sources
wget -O fdk-aac.tar.gz https://github.com/mstorsjo/fdk-aac/tarball/master
tar xzvf fdk-aac.tar.gz
cd mstorsjo-fdk-aac*
autoreconf -fiv
./configure --prefix="$HOME/ffmpeg_build" --disable-shared
make -j$(nproc)
make -j$(nproc) install
make -j$(nproc) distclean

```

**(e). Build and configure libvpx:**

```
cd ~/ffmpeg_sources
git clone https://github.com/webmproject/libvpx
cd libvpx
./configure --prefix="$HOME/ffmpeg_build" --enable-runtime-cpu-detect --enable-vp9 --enable-vp8 \
--enable-postproc --enable-vp9-postproc --enable-multi-res-encoding --enable-webm-io --enable-better-hw-compatibility --enable-vp9-highbitdepth --enable-onthefly-bitpacking --enable-realtime-only --cpu=native --as=nasm 
time make -j$(nproc)
time make -j$(nproc) install
time make clean -j$(nproc)
time make distclean
```

**(f). Build LibVorbis:**

```
cd ~/ffmpeg_sources
wget -c -v http://downloads.xiph.org/releases/vorbis/libvorbis-1.3.6.tar.xz
tar -xvf libvorbis-1.3.6.tar.xz
cd libvorbis-1.3.6
./configure --enable-static --prefix="$HOME/ffmpeg_build"
time make -j$(nproc)
time make -j$(nproc) install
time make clean -j$(nproc)
time make distclean
```

**(g). Build FFmpeg (with OpenCL enabled):**

Notes on API support:

The hardware can be accessed through a number of different APIs:

i.libmfx on Linux:

> This is a library from Intel which can be installed as part of the Intel Media SDK, and supports a subset of encode and decode cases.

ii.vaapi on Linux:

> A fully opensource stack, dependent on libva and an appropriate VAAPI driver, that can be configured at runtime via the LIBVA-related environment variables (as shown above).

In our use case, that is the backend that we will be using throughout this validation with this FFmpeg build:

```
cd ~/ffmpeg_sources
git clone https://github.com/FFmpeg/FFmpeg -b master
cd FFmpeg
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig:/opt/intel/mediasdk/lib/pkgconfig" ./configure \
  --pkg-config-flags="--static" \
  --prefix="$HOME/bin" \
  --bindir="$HOME/bin" \
  --extra-cflags="-I$HOME/bin/include" \
  --extra-ldflags="-L$HOME/bin/lib" \
  --extra-cflags="-I/opt/intel/mediasdk/include" \
  --extra-ldflags="-L/opt/intel/mediasdk/lib" \
  --extra-ldflags="-L/opt/intel/mediasdk/plugins" \
  --enable-libmfx \
  --enable-vaapi \
  --enable-opencl \
  --disable-debug \
  --enable-libvorbis \
  --enable-libvpx \
  --enable-libdrm \
  --enable-gpl \
  --cpu=native \
  --enable-libfdk-aac \
  --enable-libx264 \
  --enable-libx265 \
  --extra-libs=-lpthread \
  --enable-nonfree 
PATH="$HOME/bin:$PATH" make -j$(nproc) 
make -j$(nproc) install 
make -j$(nproc) distclean 
hash -r
```

**Note:** To get debug builds, add the `--enable-debug=3` configuration flag and omit the `distclean` step and you'll find the `ffmpeg_g` binary under the sources subdirectory.

We only want the debug builds when an issue crops up and a gdb trace may be required for debugging purposes. Otherwise, leave this omitted for production environments.

**Sample snippets to test the new encoders:**

Confirm that the VAAPI & QSV-based encoders have been built successfully:

(a). VAAPI:

```
ffmpeg  -hide_banner -encoders | grep vaapi 

 V..... h264_vaapi           H.264/AVC (VAAPI) (codec h264)
 V..... hevc_vaapi           H.265/HEVC (VAAPI) (codec hevc)
 V..... mjpeg_vaapi          MJPEG (VAAPI) (codec mjpeg)
 V..... mpeg2_vaapi          MPEG-2 (VAAPI) (codec mpeg2video)
 V..... vp8_vaapi            VP8 (VAAPI) (codec vp8)
 V..... vp9_vaapi            VP9 (VAAPI) (codec vp9)


```

(b). QSV:

```
 V..... h264_qsv             H.264 / AVC / MPEG-4 AVC / MPEG-4 part 10 (Intel Quick Sync Video acceleration) (codec h264)
 V..... hevc_qsv             HEVC (Intel Quick Sync Video acceleration) (codec hevc)
 V..... mjpeg_qsv            MJPEG (Intel Quick Sync Video acceleration) (codec mjpeg)
 V..... mpeg2_qsv            MPEG-2 video (Intel Quick Sync Video acceleration) (codec mpeg2video)
```

See the help documentation for each encoder in question:

```
ffmpeg -hide_banner -h encoder='encoder name'
```

**Test the encoders;**

Using GNU parallel, we will encode some mp4 files (4k H.264 test samples, 40 minutes each, AAC 6-channel audio) on the ~/src path on the system to VP8 and HEVC respectively using the examples below. Note that I've tuned the encoders to suit my use-cases, and re-scaling to 1080p is enabled. Adjust as necessary (and compensate as needed if `~/bin` is not on the system path).

**To VP8, launching 10 encode jobs simultaneously:**

```
parallel -j 10 --verbose 'ffmpeg -loglevel debug -threads 4 -hwaccel vaapi -i "{}"  -vaapi_device /dev/dri/renderD129 -c:v vp8_vaapi -loop_filter_level:v 63 -loop_filter_sharpness:v 15 -b:v 4500k -maxrate:v 7500k -vf 'format=nv12,hwupload,scale_vaapi=w=1920:h=1080' -c:a libvorbis -b:a 384k -ac 6 -f webm "{.}.webm"' ::: $(find . -type f -name '*.mp4')
```

**To HEVC with GNU Parallel:**

To HEVC Main Profile, launching 10 encode jobs simultaneously:

```
parallel -j 4 --verbose 'ffmpeg -loglevel debug -threads 4 -hwaccel vaapi -i "{}"  -vaapi_device /dev/dri/renderD129 -c:v hevc_vaapi -qp:v 19 -b:v 2100k -maxrate:v 3500k -vf 'format=nv12,hwupload,scale_vaapi=w=1920:h=1080' -c:a libvorbis -b:a 384k -ac 6 -f matroska "{.}.mkv"' ::: $(find . -type f -name '*.mp4')
```

**FFmpeg QSV encoder usage notes:**

I provide an example below that demonstrates the use of a complex filter chain with the QSV encoders in place for livestreaming purposes. Adapt as per your needs.

**Complex Filter chain usage for variant stream encoding with Intel's QSV encoders:**

Take the example snippet below, which takes the safest (and not necessarily the fastest route), utilizing a hybrid encoder approach (partial hwaccel with significant processor load):

```
ffmpeg -re -stream_loop -1 -threads n -loglevel debug -filter_complex_threads n \
-init_hw_device qsv=qsv:MFX_IMPL_hw_any -hwaccel qsv -filter_hw_device qsv \
-i 'udp://$stream_url:$port?fifo_size=9000000' \
-filter_complex "[0:v]split=6[s0][s1][s2][s3][s4][s5]; \
[s0]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=1920:1080:format=nv12[v0]; \
[s1]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=1280:720:format=nv12[v1];
[s2]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=960:540:format=nv12[v2];
[s3]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=842:480:format=nv12[v3];
[s4]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=480:360:format=nv12[v4];
[s5]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=426:240:format=nv12[v5]" \
-b:v:0 2250k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
-b:v:1 1750k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
-b:v:2 1000k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
-b:v:3 875k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
-b:v:4 750k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
-b:v:5 640k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
-c:a aac -b:a 128k -ar 48000 -ac 2 \
-flags -global_header -f tee -use_fifo 1 \
-map "[v0]" -map "[v1]" -map "[v2]" -map "[v3]" -map "[v4]" -map "[v5]" -map 0:a:0 -map 0:a:1 \
"[select=\'v:0,a\':f=mpegts]udp:$stream_url_out:$port_out| \
 [select=\'v:0,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:0,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:1,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:1,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:1,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:2,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:2,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:2,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:3,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:3,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:3,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:4,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:4,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:4,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:5,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:5,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:5,a\':f=mpegts]udp://$stream_url_out:$port_out"
```

**Template breakdown:**

The ffmpeg snippet assumes the following:

1.  The UDP ingest stream specifier has one video stream and two separate audio streams, as shown in the explicit mapping above: `-map "[v0]" -map "[v1]" -map "[v2]" -map "[v3]" -map "[v4]" -map "[v5]" -map 0:a:0 -map 0:a:1`

The mapping is needed because the [tee muxer](https://www.ffmpeg.org/ffmpeg-formats.html#tee-1) (`-f tee`) makes no assumptions about the capabilities of the underlying muxer launched beneath the fifo process.

2.  We encode audio only once. Consider audio as a [blocking encoder and minimize unnecessary encoder duplication](https://blog.twitch.tv/live-video-transmuxing-transcoding-ffmpeg-vs-twitchtranscoder-part-ii-4973f475f8a3) to save on CPU cycles.
    
3.  We split the incoming streams into six, and in so doing:
    

(a). Allocate a single thread to each filter complex chain. Thus the value `n` set for `-filter_complex_threads` should match the number of `split=n` value.

(b). Allocate the total thread count for FFmpeg to the value specified above. This ensures that each encoder is fed only through a single thread, the optimal value for hardware-accelerated encoding.

The following notes about thread allocation in FFmpeg applies doubly so:

More encoder threads beyond a certain threshold increases latency and will have a higher encoding memory footprint. Quality degradation is more prominent with higher thread counts in constant bitrate modes and near-constant bitrate mode called VBV ([video buffer verifier](https://en.wikipedia.org/wiki/Video_buffering_verifier)), due to increased encode delay. Keyframes need more data then other frame types to avoid pulsing poor quality keyframes.

Zero-delay or sliced thread mode (on supported encoders) has no delay, but this option farther worsens multi-threads quality in supported encoders.

It's therefore wise to limit thread counts on encodes where latency matters, as the perceived encoder throughput increase offsets any advantages it may bring in the long term.

4.  Through the tee muxer, we then allocate each encoded stream variant to an output, through the select statement in the bracketed clauses above. This allows us to generate exponentially more elementary streams than there are encoders.
    
5.  On the tuning options passed to the `h264_qsv` encoder:
    

(a). The `hwupload` filter must be appended with the `extra_hw_frames=10` option, because the QSV encoder expects a fixed initial pool size.  
Conceptually, this should happen automatically, but the problem is that the mfx plugins lack sufficient negotiation with the encoder to know how big the pool should be - if it's feeding into a scaler (such as the complex scale filter chain above), then probably 2 or 3 frames are sufficient, but if it's feeding into an encoder with look-ahead enabled then you might need over 100. As such, it's currently under user control and must be applied to ensure proper encoder initialization.

Without this option, the encoder will fail as shown below:

```
[AVHWFramesContext @ 0x3e26ec0] QSV requires a fixed frame pool size
[AVHWFramesContext @ 0x3e26ec0] Error creating an internal frame pool
[Parsed_hwupload_1 @ 0x3e26880] Failed to configure output pad on Parsed_hwupload_1
Error reinitializing filters!
Failed to inject frame into filter network: Invalid argument
Error while processing the decoded data for stream #0:0
[AVIOContext @ 0x3a68f80] Statistics: 0 seeks, 0 writeouts
[AVIOContext @ 0x3a6cd40] Statistics: 1022463 bytes read, 0 seeks
Conversion failed!
```

(b). When initializing the encoder, the hardware device node for [libmfx](https://trac.ffmpeg.org/wiki/Hardware/QuickSync) _must_ be initialized as shown:

```
-init_hw_device qsv=qsv:MFX_IMPL_hw_any -hwaccel qsv -filter_hw_device qsv
```

That ensures that the proper hardware accelerator node (`qsv`) is initialized with the proper device context (`-init_hw_device qsv=qsv:MFX_IMPL_hw_any`) with device nodes for a hardware accelerator implementation being inherited `(-hwaccel qsv`) with an appropriate filter device (`-filter_hw_device qsv`) are initialized for resource allocation by the `hwupload` filter, the `vpp_qsv` post-processor (needed for advanced deinterlacing) and the `scale_vpp` filter (needed for texture format conversion to nv12, otherwise the encoder will fail).

If hardware scaling is undesired, the filter chain can be modified from:

```
[sn]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=W:H:format=nv12[vn]
```

To:

```
[sn]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=format=nv12[vn]
```

Where `n` is the stream specifier id inherited from the complex filter chain. Note that the texture conversion is mandatory, and cannot be skipped.

The other arguments passed to the encoder are optimal for smooth streaming, enabling automatic detection and use of closed captions, an advanced [rate distortion algorithm](https://en.wikipedia.org/wiki/Rate%E2%80%93distortion_optimization) and sensible bitrates and profile limits per encoder variant.

**Special notes concerning performance:**

If you're after a **full hardware-accelerated transcode pipeline** (use with caution as it may not work with all input formats), see the snippet below:

```
ffmpeg -re -stream_loop -1 -threads n -loglevel debug -filter_complex_threads n \
-c:v h264_qsv -hwaccel qsv \
-i 'udp://$stream_url:$port?fifo_size=9000000' \
-filter_complex "[0:v]split=6[s0][s1][s2][s3][s4][s5]; \
[s0]vpp_qsv=deinterlace=2,scale_qsv=1920:1080:format=nv12[v0]; \
[s1]vpp_qsv=deinterlace=2,scale_qsv=1280:720:format=nv12[v1];
[s2]vpp_qsv=deinterlace=2,scale_qsv=960:540:format=nv12[v2];
[s3]vpp_qsv=deinterlace=2,scale_qsv=842:480:format=nv12[v3];
[s4]vpp_qsv=deinterlace=2,scale_qsv=480:360:format=nv12[v4];
[s5]vpp_qsv=deinterlace=2,scale_qsv=426:240:format=nv12[v5]" \
-b:v:0 2250k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
-b:v:1 1750k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
-b:v:2 1000k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
-b:v:3 875k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
-b:v:4 750k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
-b:v:5 640k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
-c:a aac -b:a 128k -ar 48000 -ac 2 \
-flags -global_header -f tee -use_fifo 1 \
-map "[v0]" -map "[v1]" -map "[v2]" -map "[v3]" -map "[v4]" -map "[v5]" -map 0:a:0 -map 0:a:1 \
"[select=\'v:0,a\':f=mpegts]udp:$stream_url_out:$port_out| \
 [select=\'v:0,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:0,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:1,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:1,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:1,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:2,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:2,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:2,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:3,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:3,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:3,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:4,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:4,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:4,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:5,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:5,a\':f=mpegts]udp://$stream_url_out:$port_out| \
 [select=\'v:5,a\':f=mpegts]udp://$stream_url_out:$port_out"
```

So, what has changed here? For one:

(a). We have selected an appropriate QSV-based decoder based on the video codec type in the ingest feed (h264), assigned as `-c:v h264_qsv` and the `-hwaccel qsv` as the hwaccel before declaring the input (`-i`). If the ingest feed is MPEG-2, select the MPEG decoder (`-c:v mpeg2_qsv`) instead.

(b). We have dropped the manual H/W init (`-init_hw_device qsv=qsv:MFX_IMPL_hw_any -hwaccel qsv -filter_hw_device qsv`) and the hwupload video filter.

4.  The open source iMSDK has a frame encoder limit of 1000 for the HEVC-based encoder, and as such, the HEVC encoder components should only be used for evaluation purposes. These that require these functions should consult the [proprietary licensed SDK](https://software.intel.com/en-us/media-sdk). To specify the frame limit in ffmpeg, use the `-vframes n` option, where `n` is an integer.
    
5.  The `iHD` libva driver also provides similar VAAPI functionality as the opensource `i965` driver, with a few discrepancies:
    

(a). It does not offer encode entry points for VP8 and VP9 codecs (yet). (b). As mentioned above, HEVC encoding is for evaluation purposes only and will limit the encode to a mere 1000 frames.

See the VAAPI features enabled with this iHD driver:

```
vainfo 
libva info: VA-API version 1.2.0
libva info: va_getDriverName() returns 0
libva info: User requested driver 'iHD'
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_2
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.2 (libva 2.2.1.pre1)
vainfo: Driver version: Intel iHD driver - 2.0.0
vainfo: Supported profile and entrypoints
      VAProfileNone                   :	VAEntrypointVideoProc
      VAProfileNone                   :	VAEntrypointStats
      VAProfileMPEG2Simple            :	VAEntrypointVLD
      VAProfileMPEG2Simple            :	VAEntrypointEncSlice
      VAProfileMPEG2Main              :	VAEntrypointVLD
      VAProfileMPEG2Main              :	VAEntrypointEncSlice
      VAProfileH264Main               :	VAEntrypointVLD
      VAProfileH264Main               :	VAEntrypointEncSlice
      VAProfileH264Main               :	VAEntrypointFEI
      VAProfileH264Main               :	VAEntrypointEncSliceLP
      VAProfileH264High               :	VAEntrypointVLD
      VAProfileH264High               :	VAEntrypointEncSlice
      VAProfileH264High               :	VAEntrypointFEI
      VAProfileH264High               :	VAEntrypointEncSliceLP
      VAProfileVC1Simple              :	VAEntrypointVLD
      VAProfileVC1Main                :	VAEntrypointVLD
      VAProfileVC1Advanced            :	VAEntrypointVLD
      VAProfileJPEGBaseline           :	VAEntrypointVLD
      VAProfileJPEGBaseline           :	VAEntrypointEncPicture
      VAProfileH264ConstrainedBaseline:	VAEntrypointVLD
      VAProfileH264ConstrainedBaseline:	VAEntrypointEncSlice
      VAProfileH264ConstrainedBaseline:	VAEntrypointFEI
      VAProfileH264ConstrainedBaseline:	VAEntrypointEncSliceLP
      VAProfileVP8Version0_3          :	VAEntrypointVLD
      VAProfileHEVCMain               :	VAEntrypointVLD
      VAProfileHEVCMain               :	VAEntrypointEncSlice
      VAProfileHEVCMain               :	VAEntrypointFEI
      VAProfileHEVCMain10             :	VAEntrypointVLD
      VAProfileHEVCMain10             :	VAEntrypointEncSlice
      VAProfileVP9Profile0            :	VAEntrypointVLD
      VAProfileVP9Profile2            :	VAEntrypointVLD
```

**Note:** OpenCL enablement in both libx264 and FFmpeg are dependent on the following conditions:

(a). The flag `--disable-opencl` is removed from libx264's configuration.

(b). The flag `--enable-opencl` is present in FFmpeg's configure options.

(c ). The prerequisite packages for OpenCL development are present:

With OpenCL, the installable client drivers (ICDs) are normally issued with the accelerator's device drivers, namely:

```
1.The NVIDIA CUDA toolkit (and the device driver) for NVIDIA GPUs.
2. AMD's RoCM for GCN-class AMD hardware.
3. Intel's beignet and the newer Neo compute runtime, as on OUR platform.
```

The purpose of the installable client driver model is to allow multiple OpenCL platforms to coexist on the same platform. That way, multiple OpenCL accelerators, be they discrete GPUs paired with a combination of FPGAs and integrated GPUs can all coexist.

However, for linkage purposes, you'll require the ocl-icd package (which we installed earlier), which can be installed by:

```
sudo apt install ocl-icd-* 
```

Why ocl-icd? Simple: Whereas other ICDs may permit you to link against them directly, it is discouraged so as to limit the risk of unexpected runtime behavior. Assume ocl-icd to be the gold link target if your goal is to be platform-neutral as possible.

**OpenCL in FFmpeg:**

OpenCL's enablement in FFmpeg comes in two ways:

(a):. Some encoders, such as `libx264`, if built with OpenCL enablement, can utilize these capabilities for accelerated lookahead functions. The performance impact for this enablement will vary with the GPU on the platform, and with older GPUs, may slow down the encoder. Lower power platforms such as specific AMD APUs and their SoCs may see modest performance improvements at best, but on modern, high performance GPUs, your mileage may vary. Expect no miracles. The reason OpenCL lookahead is available for this library in particular is that the lookahead algorithms for OpenCL are easily parallelized.

For instance, you can combine the `-hwaccel auto` option which allows you to select the hardware-based accelerated decoding to use for the encode session with `libx264`. You can add this parameter with "auto" before input (if your x264 is compiled with OpenCL support you can try to add `-x264opts` param), for example:

```
ffmpeg -hwaccel auto -i input -vcodec libx264 -x264opts opencl output
```

(b):. FFmpeg, in particular, can utilize OpenCL with some filters, namely `program_opencl` and `opencl_src` as documented in the filters documentation, among others.

See the sample command below:

```
ffmpeg -hide_banner -v verbose -init_hw_device 
opencl=ocl:1.0 -filter_hw_device ocl -i 
"cheeks.mkv" -an -map_metadata -1 -sws_flags 
lanczos+accurate_rnd+full_chroma_int+full_chroma_inp -filter_complex 
"[0:v]yadif=0:0:0,hwupload,unsharp_opencl=lx=3:ly=3:la=0.5:cx=3:cy=3:ca=0.5,hwdownload,setdar=dar=16/9" 
 -r 25 -c:v h264_nvenc -preset:v llhq -bf 2 -g 50 -refs 3 -rc:v 
vbr_hq -rc-lookahead:v 32 -coder:v cabac -movflags 
+faststart -profile:v high -level 4.1 -pixel_format yuv420p -y 
"crunchy_cheeks.mp4"
```

**List OpenCL platform devices:***

```
ffmpeg -hide_banner -v verbose -init_hw_device list
ffmpeg -hide_banner -v verbose -init_hw_device opencl
ffmpeg -hide_banner -v verbose -init_hw_device opencl:1.0 
```

For the filter, see:

```
ffmpeg -hide_banner -v verbose -h filter=unsharp_opencl 
```

**Example:**

On the test-bed:

```
ffmpeg -hide_banner -v verbose -init_hw_device list
Supported hardware device types:
vaapi
qsv
drm
opencl
```

And based on these platforms:

(a). QSV:

```
ffmpeg -hide_banner -v verbose -init_hw_device qsv
[AVHWDeviceContext @ 0x559f67501440] Opened VA display via X11 display :1.
[AVHWDeviceContext @ 0x559f67501440] libva: VA-API version 1.2.0
[AVHWDeviceContext @ 0x559f67501440] libva: va_getDriverName() returns 0
[AVHWDeviceContext @ 0x559f67501440] libva: User requested driver 'iHD'
[AVHWDeviceContext @ 0x559f67501440] libva: Trying to open /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so
[AVHWDeviceContext @ 0x559f67501440] libva: Found init function __vaDriverInit_1_2
[AVHWDeviceContext @ 0x559f67501440] libva: va_openDriver() returns 0
[AVHWDeviceContext @ 0x559f67501440] Initialised VAAPI connection: version 1.2
[AVHWDeviceContext @ 0x559f67501440] Unknown driver "Intel iHD driver - 2.0.0", assuming standard behaviour.
[AVHWDeviceContext @ 0x559f67501040] Initialize MFX session: API version is 1.27, implementation version is 1.27
[AVHWDeviceContext @ 0x559f67501040] MFX compile/runtime API: 1.27/1.27
Hyper fast Audio and Video encoder

```

(b). VAAPI:

```
ffmpeg -hide_banner -v verbose -init_hw_device vaapi
[AVHWDeviceContext @ 0x56362b64d040] Opened VA display via X11 display :1.
[AVHWDeviceContext @ 0x56362b64d040] libva: VA-API version 1.2.0
[AVHWDeviceContext @ 0x56362b64d040] libva: va_getDriverName() returns 0
[AVHWDeviceContext @ 0x56362b64d040] libva: User requested driver 'iHD'
[AVHWDeviceContext @ 0x56362b64d040] libva: Trying to open /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so
[AVHWDeviceContext @ 0x56362b64d040] libva: Found init function __vaDriverInit_1_2
[AVHWDeviceContext @ 0x56362b64d040] libva: va_openDriver() returns 0
[AVHWDeviceContext @ 0x56362b64d040] Initialised VAAPI connection: version 1.2
[AVHWDeviceContext @ 0x56362b64d040] Unknown driver "Intel iHD driver - 2.0.0", assuming standard behaviour.

```

(c). OpenCL:

```
ffmpeg -hide_banner -v verbose -init_hw_device opencl

[AVHWDeviceContext @ 0x55dee05a2040] 0.0: NVIDIA CUDA / GeForce GTX 1070 with Max-Q Design
[AVHWDeviceContext @ 0x55dee05a2040] 1.0: Intel(R) OpenCL HD Graphics / Intel(R) Gen9 HD Graphics NEO
[AVHWDeviceContext @ 0x55dee05a2040] More than one matching device found.
Device creation failed: -19.
Failed to set value 'opencl' for option 'init_hw_device': No such device
Error parsing global options: No such device
```

Now, you'll notice a platform init error for OpenCL, and that is because we did not pick up a specific device. When done properly, for both devices, this is the output you should get:

i. First OpenCL device:

```
ffmpeg -hide_banner -v verbose -init_hw_device opencl:1.0

[AVHWDeviceContext @ 0x562f64b66040] 1.0: Intel(R) OpenCL HD Graphics / Intel(R) Gen9 HD Graphics NEO
Hyper fast Audio and Video encoder
```

ii. Second OpenCL device:

```
ffmpeg -hide_banner -v verbose -init_hw_device opencl:0.1

ffmpeg -hide_banner -v verbose -init_hw_device opencl:0.0
[AVHWDeviceContext @ 0x55e7524fb040] 0.0: NVIDIA CUDA / GeForce GTX 1070 with Max-Q Design
```

Take note of the syntax used. On a platform with more than one OpenCL platform, the device ordinal must be selected from the OpenCL platform its' on, followed by the device index number.

Using the example above, you can see that this device has two OpenCL platforms, the Intel Neo stack and the NVIDIA CUDA stack. These platforms are `opencl:1` and `opencl:0` respectively. The devices are `opencl:1:1` and `opencl:0:0` respectively, where the first device ordinal is always zero (0).

**An OpenCL-based tonemap filter example:**

You can use an OpenCL-based tone map filters, allowing for HDR(HDR10/HLG) to SDR conversion with tone-mapping, with the example shown below:

```
ffmpeg -init_hw_device vaapi=va:/dev/dri/renderD128 -init_hw_device \
opencl=ocl@va -hwaccel vaapi -hwaccel_device va -hwaccel_output_format \
vaapi -i INPUT -filter_hw_device ocl -filter_complex \
'[0:v]hwmap,tonemap_opencl=t=bt2020:tonemap=linear:format=p010[x1]; \
[x1]hwmap=derive_device=vaapi:reverse=1' -c:v hevc_vaapi -profile 2 OUTPUT
```
