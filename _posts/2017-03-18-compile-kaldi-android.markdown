---
layout:     post
title:      "Compile Kaldi for Android"
date:       2017-03-18 12:00:00
author:     "Eduardo Silva"
header-img: "img/post-bg-02.jpg"
---

The following instructions were tested in Ubuntu 16.04, 64 bits.

## Download Android NDK

To compile native code you will need to download the Android NDK 14b.
It seems there are some problems when compiling Kaldi using another NDK version
(for instance, r15). I didn't have time to find out the reason. If you discover
it, please e-mail me and I will update this post accordingly.

```
wget -q --output-document=android-ndk.zip https://dl.google.com/android/repository/android-ndk-r14b-linux-x86_64.zip
```

After that, please, unzip `android-ndk.zip`. In this example we will call
`<NDK root dir>` the absolute path to the unzipped directory.

Finally, install the toolchain. You **must** explicitly specify `--stl=libc++`,
to copy LLVM libc++ headers and libraries:

```
<NDK root dir>/build/tools/make_standalone_toolchain.py --arch arm --api 21 --stl=libc++ --install-dir /tmp/my-android-toolchain
```

This command creates a directory named /tmp/my-android-toolchain/, containing a
copy of the android-21/arch-arm sysroot, and of the toolchain binaries for a
32-bit ARM architecture.

For more details about it, please read <https://developer.android.com/ndk/guides/standalone_toolchain.html>.


## Compile OpenBLAS for Android

The following instructions were tested on commit SHA 99880f7.
But they should work in more recent versions.

#### Download source

```
git clone https://github.com/xianyi/OpenBLAS
```

#### Install gfortran
```
sudo apt-get install gfortran
```

#### Add the Android toolchain to your path

```
export PATH=/tmp/my-android-toolchain/bin:$PATH
```
where `/tmp/my-android-toolchain` is the path to the standalone-toolchain installed in the previous step.

#### Build for ARMV7

```
make TARGET=ARMV7 HOSTCC=gcc CC=arm-linux-androideabi-gcc NO_SHARED=1 NOFORTRAN=1 NUM_THREADS=32 libs
```

OBS: The variable **NUM_THREADS** does not need to be always set to 32. In my case, I had
problems when running in my phone. I received the message:
`BLAS : Program is Terminated. Because you tried to allocate too many memory regions.`.
Looking for solutions, I found this instruction at OpenBLAS FAQ
(<https://github.com/xianyi/OpenBLAS/wiki/faq#allocmorebuffers>):

```
[...] This error indicates that the program exceeded the number of buffers.
Please build OpenBLAS with larger NUM_THREADS.
For example, make NUM_THREADS=32 or make NUM_THREADS=64 [...]
```

#### Install library

```
make install NO_SHARED=1 PREFIX=`pwd`/install
```


## Compile CLAPACK for Android

I tested it in commit SHA a71cd07d418a.
But it should work with more recent versions of this code.

```
git clone https://github.com/simonlynen/android_libs.git

cd android_libs/lapack

# We will use -mfloat-abi=hard to compile other OpenBLAS and Kaldi.
# According to https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html,
# we must compile all libraries with the same ABI.
sed -i 's/-mfloat-abi=softfp/-mfloat-abi=hard/g' jni/Application.mk

# remove some compile instructions related to tests
sed -i 's/LOCAL_MODULE:= testlapack/#LOCAL_MODULE:= testlapack/g' jni/Android.mk
sed -i 's/LOCAL_SRC_FILES:= testclapack.cpp/#LOCAL_SRC_FILES:= testclapack.cpp/g' jni/Android.mk
sed -i 's/LOCAL_STATIC_LIBRARIES := lapack/#LOCAL_STATIC_LIBRARIES := lapack/g' jni/Android.mk
sed -i 's/include $(BUILD_SHARED_LIBRARY)/#include $(BUILD_SHARED_LIBRARY)/g' jni/Android.mk

# build for android
<NDK root dir>/ndk-build
```

Libs will be created in `obj/local/armeabi-v7a/`.

**Copy libs from `obj/local/armeabi-v7a/` to the same place you installed OpenBLAS libraries
(e.g: OpenBlas/install/lib)**.
Kaldi will look at this  directory for libf2c.a, liblapack.a, libclapack.a and libblas.a.


## Compile kaldi for Android

The following instructions should work with the most recent version of Kaldi.
If it doesn't, you may checkout commit SHA c68a576b. But, I insist you should
try the most recent Kaldi commit.

#### Download kaldi source code

```
git clone https://github.com/kaldi-asr/kaldi.git kaldi-android
```

#### Compile OpenFST

In the instructions, we are using OpenFST-1.6.2. But you should use the current
version used in your Kaldi repository. You may discover it looking at tools/Makefile.

```
export PATH=/tmp/my-android-toolchain/bin:$PATH

cd kaldi-android/tools

wget -T 10 -t 1 http://openfst.cs.nyu.edu/twiki/pub/FST/FstDownload/openfst-1.6.2.tar.gz

tar -zxvf openfst-1.6.2.tar.gz

cd openfst-1.6.2/

CXX=clang++ ./configure --prefix=`pwd` --enable-static --enable-shared --enable-far --enable-ngram-fsts --host=arm-linux-androideabi LIBS="-ldl"

make -j 4

make install

cd ..

ln -s openfst-1.6.2 openfst
```

#### Compile src

```
cd ../src

#Be sure android-toolchain is in your $PATH before the next step

CXX=clang++ ./configure --static --android-incdir=/tmp/my-android-toolchain/sysroot/usr/include/ --host=arm-linux-androideabi --openblas-root=/path/to/OpenBLAS/install

make clean -j

make depend -j

make -j 4
```

When using Kaldi in Android you may need to install `libc++_shared.so` (which is
located at `/tmp/my-android-toolchain/arm-linux-androideabi/lib/armv7-a/` in your
host machine) in your Android system. As I was working with a rooted Android 5.1
phone, I just copied it to `/system/lib`.

## Docker

All these instructions were summarized in a Dockerfile that may be found at
<https://github.com/jcsilva/docker-kaldi-android>.


## Updates

25/07/2017 - *Docker* section was included.

28/07/2017 - *Download Android NDK* section was update.


## References

<https://developer.android.com/ndk/guides/index.html>

<https://developer.android.com/ndk/guides/standalone_toolchain.html>

<https://github.com/xianyi/OpenBLAS/wiki/How-to-build-OpenBLAS-for-Android>

<http://stackoverflow.com/questions/22774009/android-ndk-stdto-string-support>

<https://github.com/xianyi/OpenBLAS/issues/539>
