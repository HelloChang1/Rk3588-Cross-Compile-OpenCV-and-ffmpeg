# opencv与ffmpeg源码交叉编译
## step1:环境准备

linux系统：unbuntu18.04 X86_64
rk3588软件包：rknn-toolkit2:1.5.2-cp36
交叉编译工具：gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu
硬件平台：瑞芯微RK3588

### 1. 创建docker虚拟环境

```bash
    docker run -it --privileged --name NewF3588 -p 35880:22 -v /home/user/rk3588:/workspace/user/rk3588  rknn-toolkit2:1.5.2-cp36  /bin/bash
``` 
### 2. 搭建交叉编译环境
1. 将交叉编译工具放置在docker虚拟环境中，比如/opt目录下，并改名gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu为aarch64-rockchip-linux-gnu
2. 设置交叉编译环境变量：
```bash
    vim ~/.bashrc
```
在文件最后添加：
```bash
    export LD_LIBRARY_PATH=/opt/aarch64-rockchip-linux-gnu/lib:/opt/aarch64-rockchip-linux-gnu/lib64:/opt/arm/fffmpeginstall/lib:$LD_LIBRARY_PATH
    export PATH=$PATH:/opt/aarch64-rockchip-linux-gnu/bin
```
添加后保存并:
```bash
    source ~/.bashrc
```
3. 验证交叉编译环境是否搭建成功，在任意目录执行命令：
```bash
    aarch64-none-linux-gnu-gcc -v
```
## step2:源码下载与安装

**两个必要的软件包**
```bash
apt-get install build-essential pkg-config
```

cmake3.23.0:
https://cmake.org/download/
opencv-4.5.1:
https://opencv.org/releases/page/2/
opencv_contrib-4.5.1:
https://github.com/opencv/opencv_contrib/releases/tag/4.5.1
ffmpeg-4.2.9:
https://ffmpeg.org/download.html#releases


以上软件下载源码后，解压至/opt目录下，并创键arm文件夹用于存放ffmeg相关配置文件

## step3:cmake源码编译
cmake --version检查是否安装成功

## step4:ffmepg交叉编译

### 1. 命令行执行
在/opt目录下依次执行以下命令：
```bash
 cd ffmpeg-4.2.9
```

**注意修改--cross-prefix，--cc，--cxx，--prefix，其他不变**
```bash
/configure --enable-cross-compile --target-os=linux --arch=aarch64 \
--cross-prefix=/opt/aarch64-rockchip-linux-gnu/bin/aarch64-none-linux-gnu- \
--cc=/opt/aarch64-rockchip-linux-gnu/bin/aarch64-none-linux-gnu-gcc \
--cxx=/opt/aarch64-rockchip-linux-gnu/bin/aarch64-none-linux-gnu-g++ \
--prefix=/opt/arm/fffmpeginstall \
--disable-asm --enable-parsers --disable-decoders --enable-decoder=h264 --disable-debug --enable-ffmpeg --enable-shared --disable-static --disable-stripping --disable-doc
```

```bash
make -j$(nproc)
```

```bash
make install
```
### 2. 文件检查与环境变量设置

然后进入你的--prefix目录，查看生成的文件（bin,include,lib,share）
在/bin目录下执行下面命令，若出现aarch64则表示ffmpeg交叉编译成功：
```bash
file libavcodec.so.60.35.100
```

设置ffmpeg的环境变量：
```bash
export PKG_CONFIG_PATH=/opt/arm/fffmpeginstall/lib/pkgconfig$PKG_CONFIG_PATH
export PKG_CONFIG_LIBDIR=/opt/arm/fffmpeginstall/lib$PKG_CONFIG_LIBDIR
export LD_LIBRARY_PATH=/opt/arm/fffmpeginstall/lib:$LD_LIBRARY_PATH
```
查看环境变量格式是否正确：
```bash
export 
```

**进入/opt/aarch64-rockchip-linux-gnu/aarch64-none-linux-gnu/libc/usr路径下**
**将ffmpeg生成的文件（bin,include,lib,share目录下的文件）分别对应复制到usr目录下bin，include，lib，share文件夹中**
## step5:opencv&opencv_contrib交叉编译

### 1. 创建目录与文件修改

先把opencv_contrib源码复制到opencv源码目录下
进入opencv源码目录:
创建目录用于存放cmake的生成文件
```bash
mkdir build 
```
创建目录用于存放opencv交叉编译的生成文件夹
```bash
mkdir install
```
创键交叉编译工具链文件
```bash
vim tool_chain.cmake 
```

修改tool_chain.cmake文件，添加以下内容（**注意修改相关路径**）：
```bash
set( CMAKE_SYSTEM_NAME Linux )
set( CMAKE_SYSTEM_PROCESSOR aarch64 )
set( CMAKE_C_COMPILER /opt/aarch64-rockchip-linux-gnu/bin/aarch64-none-linux-gnu-gcc)
set( CMAKE_CXX_COMPILER /opt/aarch64-rockchip-linux-gnu/bin/aarch64-none-linux-gnu-g++)
#set( OPENCV_ENABLE_PKG_CONFIG ON)
#set( CMAKE_C_FLAGS "-Wl,-rpath-link=/opt/arm/fffmpeginstall/lib")
set( CMAKE_FIND_ROOT_PATH "/opt/arm/fffmpeginstall/lib" )
set( CMAKE_CXX_FLAGS "-Wl,-rpath=/opt/arm/fffmpeginstall/lib")
set( CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER )
set( CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY )
set( CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY )
```
### 2. opencv编译链接ffmpeg
保存后关闭文件，进入opencv源码根目录，进入build文件夹，执行以下命令：
```bash
cmake -DCMAKE_INSTALL_PREFIX=../install -DOPENCV_EXTRA_MODULES_PATH=../opencv_contrib-4.5.1/modules -DWITH_FFMPEG=ON -DCMAKE_TOOLCHAIN_FILE=../toolchain.cmake ..
```
**出现ffmpeg YES 表示链接ffmpeg成功**

```bash
make -j$(nproc)
```
```bash
make install
```
完成
