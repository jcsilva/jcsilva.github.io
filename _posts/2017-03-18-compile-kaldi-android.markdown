---
layout:     post
title:      "Compile Kaldi in Android"
date:       2017-03-18 12:00:00
author:     "Eduardo Silva"
header-img: "img/post-bg-02.jpg"
---

## First of all, download Android NDK (https://developer.android.com/ndk/guides/index.html):

To compile and debug native code for your app, you need the following components:

* The Android Native Development Kit (NDK): a set of tools that allows you to use C and C++ code with Android.
* CMake: an external build tool that works alongside Gradle to build your native library. You do not need this component if you only plan to use ndk-build.
* LLDB: the debugger Android Studio uses to debug native code.
You can install these components using the SDK Manager:

1. From an open project, select Tools > Android > SDK Manager from the main menu.
2. Click the SDK Tools tab.
3. Check the boxes next to LLDB, CMake, and NDK.
4. Click Apply, and then click OK in the next dialog.
5. When the installation is complete, click Finish, and then click OK.
6. Install the toolchain: <NDK root dir>/build/tools/make_standalone_toolchain.py --arch arm --api 21 --install-dir /tmp/my-android-toolchain


## Compile OpenBlas for Android (https://github.com/xianyi/OpenBLAS/wiki/How-to-build-OpenBLAS-for-Android)

### Add the Android toolchain to your path
```
export PATH=/tmp/my-android-toolchain:$PATH
```
where `/tmp/my-android-toolchain` is the path to the standalone-toolchain installed in the previous step.

### Build without Fortran for ARMV7
```
make TARGET=ARMV7 HOSTCC=gcc CC=arm-linux-androideabi-gcc NOFORTRAN=1 libs
```

### Install library
```
make install PREFIX=`pwd`/install
```


## Compile CLAPACK for Android

```
git clone https://github.com/simonlynen/android_libs.git
cd android_libs/lapack
sed -i 's:LOCAL_MODULE:= testlapack:#LOCAL_MODULE:= testlapack:g' jni/Android.mk
sed -i 's:LOCAL_SRC_FILES:= testclapack.cpp:#LOCAL_SRC_FILES:= testclapack.cpp:g' jni/Android.mk
sed -i 's:LOCAL_STATIC_LIBRARIES := lapack:#LOCAL_STATIC_LIBRARIES := lapack:g' jni/Android.mk
sed -i 's:include $(BUILD_SHARED_LIBRARY):#include $(BUILD_SHARED_LIBRARY):g' jni/Android.mk

<NDK root dir>/ndk-build
```

Libs will be created in `obj/local/armeabi[-v7a]/`.

Copy libs to the same place you installed OpenBlas. Kaldi will look at this directory for libf2c.a, liblapack.a, libclapack.a and libblas.a.


## Compile kaldi for Android

```
sudo apt-get install clang
git clone https://github.com/kaldi-asr/kaldi.git kaldi-android
cd kaldi-android/tools

sed -i 's:cd openfst-$(OPENFST_VERSION)/; ./configure --prefix=`pwd` --enable-static --enable-shared --enable-far --enable-ngram-fsts CXX=$(CXX) CXXFLAGS="$(CXXFLAGS)" LDFLAGS="$(LDFLAGS)" LIBS="-ldl":cd openfst-$(OPENFST_VERSION)/; ./configure --prefix=`pwd` --host=arm-linux-androideabi --enable-static --enable-shared --enable-far --enable-ngram-fsts CXX=$(CXX) CXXFLAGS="$(CXXFLAGS)" LDFLAGS="$(LDFLAGS)" LIBS="-ldl":g' Makefile # compile openfst for arm

make

cd ../src
export PATH=/tmp/my-android-toolchain/bin:$PATH # to find out arm-linux-androideabi-clang++

CXX=clang++ ./configure --static --openblas-root=/media/data/git/OpenBLAS/install --android-incdir=/tmp/my-android-toolchain/sysroot/usr/include/ --host=arm-linux-androideabi

sed -i 's:ANDROIDINCDIR:ANDROIDINC:g' kaldi.mk

sed -i 's:-DHAVE_EXECINFO_H=1 -DHAVE_CXXABI_H -DHAVE_OPENBLAS -DANDROID_BUILD:-DHAVE_CXXABI_H -DHAVE_OPENBLAS -DANDROID_BUILD:g' kaldi.mk

make clean -j
make depend -j
make -j 8


```