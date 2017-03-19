---
layout:     post
title:      "Compile Kaldi for Android"
date:       2017-03-18 12:00:00
author:     "Eduardo Silva"
header-img: "img/post-bg-02.jpg"
---

## First of all, download Android NDK

To compile and debug native code for your app, you need the following components:

* The Android Native Development Kit (NDK): a set of tools that allows you to use C and C++ code with Android.
* CMake: an external build tool that works alongside Gradle to build your native library. You do not need this component if you only plan to use ndk-build.
* LLDB: the debugger Android Studio uses to debug native code.
You can install these components using the SDK Manager:

1. From an open project, select **Tools > Android > SDK Manager** from the main menu.
2. Click the SDK Tools tab.
3. Check the boxes next to LLDB, CMake, and NDK.
4. Click Apply, and then click OK in the next dialog.
5. When the installation is complete, click Finish, and then click OK.
6. Install the toolchain: `<NDK root dir>/build/tools/make_standalone_toolchain.py --arch arm --api 21 --install-dir /tmp/my-android-toolchain`


## Compile OpenBlas for Android

#### Download source
```
git clone https://github.com/xianyi/OpenBLAS

# The commit we used
git checkout 99880f7
```

#### Add the Android toolchain to your path
```
export PATH=/tmp/my-android-toolchain:$PATH
```
where `/tmp/my-android-toolchain` is the path to the standalone-toolchain installed in the previous step.

#### Build without Fortran for ARMV7
```
make TARGET=ARMV7 HOSTCC=gcc CC=arm-linux-androideabi-gcc NOFORTRAN=1 libs
```

#### Install library
```
make install PREFIX=`pwd`/install
```


## Compile CLAPACK for Android

```
git clone https://github.com/simonlynen/android_libs.git
cd android_libs/lapack
git checkout a71cd07d418a

# remove some compile instructions related to tests
sed -i 's/LOCAL_MODULE:= testlapack/#LOCAL_MODULE:= testlapack/g' jni/Android.mk
sed -i 's/LOCAL_SRC_FILES:= testclapack.cpp/#LOCAL_SRC_FILES:= testclapack.cpp/g' jni/Android.mk
sed -i 's/LOCAL_STATIC_LIBRARIES := lapack/#LOCAL_STATIC_LIBRARIES := lapack/g' jni/Android.mk
sed -i 's/include $(BUILD_SHARED_LIBRARY)/#include $(BUILD_SHARED_LIBRARY)/g' jni/Android.mk

# build for android
<NDK root dir>/ndk-build
```

Libs will be created in `obj/local/armeabi[-v7a]/`.

**Copy libs (.a) to the same place you installed OpenBlas**. Kaldi will look at this directory for libf2c.a, liblapack.a, libclapack.a and libblas.a.


## Compile kaldi for Android

```
sudo apt-get install clang

git clone https://github.com/kaldi-asr/kaldi.git kaldi-android

cd kaldi-android/tools

git checkout 3fec956be88

wget -T 10 -t 1 http://openfst.cs.nyu.edu/twiki/pub/FST/FstDownload/openfst-1.6.2.tar.gz 
tar -zxvf openfst-1.6.2.tar.gz
```

Add the following snippet to openfst-1.6.2/src/lib/symbol-table-ops.cc.

```
#include <string>
#include <sstream>

namespace patch
{
    template < typename T > std::string to_string( const T& n )
    {
        std::ostringstream stm ;
        stm << n ;
        return stm.str() ;
    }
}
```

Then, replace `std::to_string` that is in line 114 of this file (symbol-table-ops.cc) to `patch::to_string`.  It solves a problem when trying to compile openfst with arm-linux-androideabi-g++. 

Now, let's compile OpenFst using the Android toolchain:

```
export PATH=/tmp/my-android-toolchain/bin:$PATH

cd openfst-1.6.2/

./configure --prefix=`pwd` --enable-static --enable-shared --enable-far --enable-ngram-fsts --host=arm-linux-androideabi LIBS="-ldl"

make -j 4

make install 

cd ..

ln -s openfst-1.6.2 openfst
```


After finishing it, let's compile kaldi source code:

```
cd ../src

export PATH=/tmp/my-android-toolchain/bin:$PATH

CXX=clang++ ./configure --static --android-incdir=/tmp/my-android-toolchain/sysroot/usr/include/ --host=arm-linux-androideabi --openblas-root=/path/to/OpenBLAS/install

sed -i 's:ANDROIDINCDIR:ANDROIDINC:g' kaldi.mk

sed -i 's:-DHAVE_EXECINFO_H=1 -DHAVE_CXXABI_H -DHAVE_OPENBLAS -DANDROID_BUILD:-DHAVE_CXXABI_H -DHAVE_OPENBLAS -DANDROID_BUILD:g' kaldi.mk

make clean -j

make depend -j

make -j 4

```

## References

[https://developer.android.com/ndk/guides/index.html]

[https://github.com/xianyi/OpenBLAS/wiki/How-to-build-OpenBLAS-for-Android]

[http://stackoverflow.com/questions/12975341/to-string-is-not-a-member-of-std-says-g-mingw]
