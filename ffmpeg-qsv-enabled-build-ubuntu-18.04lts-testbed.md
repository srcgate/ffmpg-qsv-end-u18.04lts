**Build FFmpeg with Intel's QSV enablement on an Intel-based validation test-bed:**

Build platform: Ubuntu 18.04LTS

**Ensure the platform is up to date:**

`sudo apt update && sudo apt -y upgrade && sudo apt -y dist-upgrade`

**Install baseline dependencies first (inclusive of OpenCL headers+)**

`sudo apt-get -y install autoconf automake build-essential libass-dev libtool pkg-config texinfo zlib1g-dev libva-dev cmake mercurial libdrm-dev libvorbis-dev libogg-dev git libx11-dev libperl-dev libpciaccess-dev libpciaccess0 xorg-dev intel-gpu-tools opencl-headers libwayland-dev xutils-dev ocl-icd-* libssl-dev`

Then add the Oibaf PPA, needed to install the latest development headers for libva:

```
sudo add-apt-repository ppa:oibaf/graphics-drivers
sudo apt-get update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade
```

**Configure git:**

This will be needed by some projects below, such as when building `opencl-clang`. Use your credentials, as you see fit:

```sh
git config --global user.name "FirstName LastName"
git config --global user.email "your@email.com"
```

Then proceed.

**To address linker problems down the line with Ubuntu 18.04LTS:**

**Note:** This can be skipped as the bug has been fixed upstream.

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

**Again, this can be skipped safely as the PPA has the latest code:**

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

It is recommended that you install the Intel Neo OpenCL runtime:

**Justification:** This will allow for Intel's MediaSDK OpenCL inter-op back-end to be built. 

**Note:** There's also a Personal Package Archive (PPA) for this that you can add, allowing you to skip the manual build step, as shown:

```
sudo add-apt-repository ppa:intel-opencl/intel-opencl
sudo apt-get update
```

Then install the packages:

    sudo apt install intel-*

Then proceed.

**Install the dependencies for the OpenCL back-end:**

**Build dependencies:**

```
sudo apt-get install ccache flex bison cmake g++ git patch zlib1g-dev autoconf xutils-dev libtool pkg-config libpciaccess-dev libz-dev clinfo
```


**Testing:**

Use `clinfo` and confirm that the ICD is detected.

Optionally, run [Luxmark](http://www.luxmark.info/) and confirm that Intel Neo's OpenCL platform is detected and usable.

Be aware that Luxmark, among others, require `freeglut`, which you can install by running:

```
sudo apt install freeglut3*
```


**4. Build [Intel's MSDK](https://github.com/Intel-Media-SDK/MediaSDK):**

This package provides an API to access hardware-accelerated video decode, encode and filtering on Intel® platforms with integrated graphics. It is supported on platforms that the intel-media-driver is targeted for. 

For supported features per generation, see [this](https://github.com/intel/media-driver/blob/master/README.md).
   

**Build steps:**

(a). Fetch the sources into the working directory `~/vaapi`:

```
cd ~/vaapi
git clone https://github.com/Intel-Media-SDK/MediaSDK msdk
cd msdk
git submodule init
git pull
```

(b). Configure the build:

```
mkdir -p ~/vaapi/build_msdk
cd ~/vaapi/build_msdk
cmake -DCMAKE_BUILD_TYPE=Release -DENABLE_WAYLAND=ON -DENABLE_X11_DRI3=ON -DENABLE_OPENCL=ON  ../msdk
time make -j$(nproc) VERBOSE=1
sudo make install -j$(nproc) VERBOSE=1
```

CMake will automatically detect the platform you're on and enable the platform-specific hooks needed for a working build.

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


When done, issue a reboot:

```
 sudo systemctl reboot
```


**Build a usable FFmpeg binary with the iMSDK:**

Include extra components as needed:

**(a). Build and deploy nasm:** [Nasm](https://www.nasm.us/) is an assembler for x86 optimizations used by x264 and FFmpeg. Highly recommended or your resulting build may be very slow.

Note that we've now switched away from Yasm to nasm, as this is the current assembler that x265,x264, among others, are adopting.

```
cd ~/ffmpeg_sources
wget wget http://www.nasm.us/pub/nasm/releasebuilds/2.14.02/nasm-2.14.02.tar.gz
tar xzvf nasm-2.14.02.tar.gz
cd nasm-2.14.02
./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin" 
make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean

```

**(b). Build and deploy libx264 statically:** This library provides a H.264 video encoder. See the [H.264 Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/H.264) for more information and usage examples. This requires ffmpeg to be configured with `--enable-gpl --enable-libx264`.

```
cd ~/ffmpeg_sources
git clone http://git.videolan.org/git/x264.git 
cd x264/
PATH="$HOME/bin:$PATH" ./configure --prefix="$HOME/ffmpeg_build" --enable-static --enable-pic --bit-depth=all
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
git clone https://github.com/mstorsjo/fdk-aac
cd fdk-aac
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
./configure --prefix="$HOME/ffmpeg_build" --enable-runtime-cpu-detect --cpu=native --as=nasm --enable-vp8 --enable-vp9 \
--enable-postproc-visualizer --disable-examples --disable-unit-tests --enable-static --disable-shared \
--enable-multi-res-encoding --enable-postproc --enable-vp9-postproc \
--enable-vp9-highbitdepth --enable-pic --enable-webm-io --enable-libyuv 
time make -j$(nproc)
time make -j$(nproc) install
time make clean -j$(nproc)
time make distclean
```

**(f). Build LibVorbis:**

```
cd ~/ffmpeg_sources
git clone https://git.xiph.org/vorbis.git
cd vorbis
autoreconf -ivf
./configure --enable-static --prefix="$HOME/ffmpeg_build"
time make -j$(nproc)
time make -j$(nproc) install
time make clean -j$(nproc)
time make distclean
```

**(g). Build SDL**

```sh
cd ~/ffmpeg_sources
hg clone hg clone http://hg.libsdl.org/SDL
cd ~/ffmpeg_sources/SDL
./autogen.sh -ivf
PATH="$HOME/bin:$PATH" ./configure --prefix="$HOME/ffmpeg_build" --with-x --with-pic=yes \
--disable-alsatest --enable-pthreads --enable-static=yes --enable-shared=no
make -j$(nproc) VERBOSE=1
make -j$(nproc) install VERBOSE=1
make -j$(nproc) clean VERBOSE=1
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
  --enable-static --disable-shared \
  --prefix="$HOME/ffmpeg_build" \
  --bindir="$HOME/bin" \
  --extra-cflags="-I$HOME/ffmpeg_build/include" \
  --extra-ldflags="-L$HOME/ffmpeg_build/lib" \
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
  --enable-runtime-cpudetect \
  --enable-libfdk-aac \
  --enable-libx264 \
  --enable-libx265 \
  --enable-openssl \
  --enable-pic \
  --extra-libs="-lpthread -lm -lz -ldl" \
  --enable-nonfree 
PATH="$HOME/bin:$PATH" make -j$(nproc) 
make -j$(nproc) install 
make -j$(nproc) distclean 
hash -r
```

**Note:** To get debug builds, add the `--enable-debug=3` configuration flag and omit the `distclean` step and you'll find the `ffmpeg_g` binary under the sources subdirectory.

We only want the debug builds when an issue crops up and a gdb trace may be required for debugging purposes. Otherwise, leave this omitted for production environments.

