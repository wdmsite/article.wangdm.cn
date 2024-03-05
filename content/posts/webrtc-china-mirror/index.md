---
title: "WebRTC国内镜像"
date: 2024-03-05T13:30:00+08:00
categories:
- WebRTC
tags:
- WebRTC
draft: false
---


## 0.简介

这是一个由 声网Agora WebRTC团队提供的 WebRTC 镜像源，你可以使用此版本代替 Google 官方版本，此镜像保持与Chrome正式版同步，目前为M84。

如遇到问题，请在此地址发帖：https://rtcdeveloper.com/t/topic/14914

官方英文文档，请见：https://www.w3.org/TR/webrtc/

中文文档开源版本，请见：https://github.com/RTC-Developer/WebRTC-Documentation-in-Chinese

## 1. 环境配置

### 指定同步目录

> export WORKSPACE=\`pwd\`

### 安装depot_tools

> cd $WORKSPACE 
>
> rm -rf depot_tools && git clone https://webrtc.bj2.agoralab.co/webrtc-mirror/depot_tools.git 
>
> chmod +x $WORKSPACE/depot_tools/cipd 
>
> export PATH=$PATH:$WORKSPACE/depot_tools 

### 同步WebRTC
```
mkdir -p $WORKSPACE/webrtc && cd $WORKSPACE/webrtc
vi .gclient
//添加以下内容
solutions = [
  { "name"        : "src",
    "url"         : "https://webrtc.bj2.agoralab.co/webrtc-mirror/src.git@65e8d9facab05de13634d777702b2c93288f8849",
    "deps_file"   : "DEPS",
    "managed"     : False,
    "safesync_url": "",
    "custom_deps": {
    },
  },
]
target_os = ["linux"]  /*根据需要Android, ios*/
//开始同步，时间较久
date; gclient sync; date
//出现提示是否使用谷歌的depot_tools，推荐“n”，或者等1分钟提示框超时。
//因为gclient sync获取源码之后，会向谷歌获取其它数据，如果因此报错，不想看到这个错误的话，可以设置代理之后再次sync
export http_proxy=x.x.x.x:port
export https_proxy=x.x.x.x:port
date; gclient sync; date
```

## 2. 编译

### Linux
```
apt-get update
apt-get install -y g++
export PATH=$PATH:$WORKSPACE/depot_tools
cd $WORKSPACE/webrtc/src
./build/install-build-deps.sh
gn gen out/Release "--args=is_debug=false"
ninja -C out/Release
```

### Android
```
# 安装依赖
apt-get install -y software-properties-common
add-apt-repository -y ppa:openjdk-r/ppa
./build/install-build-deps-android.sh
# 添加安卓平台
cd $WORKSPACE/webrtc
#echo "target_os = [ 'android' ]" >>.gclient
# 同步
export PATH=$PATH:$WORKSPACE/depot_tools
cd $WORKSPACE/webrtc && gclient sync --patch-ref=https://chromium.googlesource.com/chromium/src/build.git@gitlab
# 编译
cd $WORKSPACE/webrtc/src
gn gen android/Release "--args=is_debug=false target_os="android" target_cpu="arm64""
ninja -C android/Release
```

### iOS/MacOS
```
export PATH=$PATH:$WORKSPACE/depot_tools
export CDS_CLANG_BUCKET_OVERRIDE=https://chromiumsrc.gitlab.io/commondatastorage/chromium-browser-clang
git.sh
# 安装depot_tools
cd $WORKSPACE
rm -rf depot_tools && git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git depot_tools
cd $WORKSPACE/depot_tools && git checkout gitlab
chmod +x $WORKSPACE/depot_tools/cipd
cd $WORKSPACE
rm -rf $WORKSPACE/webrtc && mkdir $WORKSPACE/webrtc
cd $WORKSPACE/webrtc && 
gclient config --name src https://chromium.googlesource.com/external/webrtc.git@gitlab
cd $WORKSPACE/webrtc && 
gclient sync
 --patch-ref=https://chromium.googlesource.com/chromium/src/build.git@gitlab
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
cd $WORKSPACE/webrtc/src && 
gn gen out/Release "--args=is_debug=false"
cd $WORKSPACE/webrtc/src && 
ninja -C out/Release
```

### Windows
```
#安装依赖
1. git - https://git-scm.com/download/win
2. visualstudio2017community - https://download.visualstudio.microsoft.com/download/pr/aaebc214-bc67-4137-9bea-04fcb0c90720/2e18f27594472d0c7515d9cbe237bd56/vs_community.exe
> 2.1 Modify "Windows Software Development Kit" > +Debugging Tools for Windows
3. python2 - https://www.python.org/downloads/release/python-2716/
> 3.1 pip install pywin32
#设置
1. 执行git.sh
#安装depot_tools
cd %USERPROFILE%
rd /s /q depot_tools webrtc & git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git -b gitlab
#配置
set PATH=%PATH%;%USERPROFILE%depot_tools
cd %USERPROFILE% && ^
rd /s /q webrtc & mkdir webrtc
cd %USERPROFILE%webrtc && ^
gclient config --name src https://chromium.googlesource.com/external/webrtc.git@gitlab
#同步
set DEPOT_TOOLS_WIN_TOOLCHAIN=0
cd %USERPROFILE%webrtc && ^
gclient sync --patch-ref=https://chromium.googlesource.com/chromium/src/build.git@gitlab
#生成
cd %USERPROFILE%webrtcsrc && ^
gn gen out/Release "--args=is_debug=false"
#编译
cd %USERPROFILE%webrtcsrc && ^
ninja -C out/Release
```

## 3. 附录-依赖及镜像列表

整个开源WebRTC编译时需要下载的依赖列表

> https://chromium.googlesource.com/chromium/src/base
> 
> https://chromium.googlesource.com/chromium/src/build
> 
> https://chromium.googlesource.com/chromium/src/buildtools
> 
> https://chromium.googlesource.com/external/github.com/gradle/gradle
> 
> https://chromium.googlesource.com/chromium/src/ios
> 
> https://chromium.googlesource.com/chromium/src/testing
> 
> https://chromium.googlesource.com/chromium/src/third_party
> 
> https://chromium.googlesource.com/chromium/llvm-project/cfe/tools/clang-format
> 
> https://chromium.googlesource.com/chromium/llvm-project/libcxx
> 
> https://chromium.googlesource.com/chromium/llvm-project/libcxxabi
> 
> https://chromium.googlesource.com/external/llvm.org/libunwind
> 
> https://chromium.googlesource.com/android_ndk
> 
> https://chromium.googlesource.com/android_tools
> 
> https://chromium.googlesource.com/external/github.com/google/auto
> 
> https://chromium.googlesource.com/catapult
> 
> https://chromium.googlesource.com/external/github.com/google/compact_enc_det
> 
> https://chromium.googlesource.com/external/colorama
> 
> https://chromium.googlesource.com/chromium/tools/depot_tools
> 
> https://chromium.googlesource.com/chromium/third_party/errorprone
> 
> https://chromium.googlesource.com/chromium/third_party/ffmpeg
> 
> https://chromium.googlesource.com/chromium/deps/findbugs
> 
> https://chromium.googlesource.com/chromium/src/third_party/freetype2
> 
> https://chromium.googlesource.com/external/github.com/harfbuzz/harfbuzz
> 
> https://chromium.googlesource.com/external/github.com/google/gtest-parallel
> 
> https://chromium.googlesource.com/external/github.com/google/googletest
> 
> https://chromium.googlesource.com/chromium/deps/icu
> 
> https://chromium.googlesource.com/external/jsr-305
> 
> https://chromium.googlesource.com/external/github.com/open-source-parsers/jsoncpp
> 
> https://chromium.googlesource.com/external/junit
> 
> https://chromium.googlesource.com/chromium/llvm-project/compiler-rt/lib/fuzzer
> 
> https://chromium.googlesource.com/chromium/deps/libjpeg_turbo
> 
> https://chromium.googlesource.com/chromium/deps/libsrtp
> 
> https://chromium.googlesource.com/webm/libvpx
> 
> https://chromium.googlesource.com/libyuv/libyuv
> 
> https://chromium.googlesource.com/linux-syscall-support
> 
> https://chromium.googlesource.com/external/mockito/mockito
> 
> https://chromium.googlesource.com/chromium/deps/nasm
> 
> https://chromium.googlesource.com/external/github.com/cisco/openh264
> 
> https://chromium.googlesource.com/external/github.com/kennethreitz/requests
> 
> https://chromium.googlesource.com/external/robolectric
> 
> https://chromium.googlesource.com/chromium/third_party/ub-uiautomator
> 
> https://chromium.googlesource.com/external/github.com/sctplab/usrsctp
> 
> https://chromium.googlesource.com/chromium/deps/yasm/binaries
> 
> https://chromium.googlesource.com/chromium/deps/yasm/patched-yasm
> 
> https://chromium.googlesource.com/chromium/src/tools
> 
> https://chromium.googlesource.com/infra/luci/client-py


<br/>

**原文地址：**

[WebRTC国内镜像](https://webrtc.org.cn/mirror/)