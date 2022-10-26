---
title: pkgconfig
date: 2022-10-20 17:56:47
tags: [pkg, makefile]
description: pkgconfig简单介绍
---

# 背景
最近在上传UADK的dpdk crypto驱动，
由于该驱动依赖UADK库，社区要求UADK库必须提供相应的pkgconfig文件．
简单记录一下调试过程中遇到的问题.

本文尝试回答三个问题

1. pkgconfig是什么
2. pkgconfig怎么用
3. pkgconfig怎么写

# pkgconfig是什么

应用程序动态加载库的时候需要指定库名-lxx, 在哪里找这个库-Lxx,以及头文件地址-Ixx
pkgconfig就是库提供的描述性文件，xx.pc
描述库的头文件地址，库被安装的路径，库的版本信息等．
是库提供给使用者用的，库自身并不需要．

比如UADK对外提供3个库, 默认安装到系统目录
```
$ ls /usr/local/lib
libwd.so libwd_crypto.so libwd_comp.so
```

对应的3个pkgconfig文件，分别描述各自库的信息
```
$ ls /usr/local/lib/pkgconfig
libcrypto.pc libwd_comp.pc libwd.pc
```

通过pkgconfig文件，pkg-config即可查到3个库的信息
```
$ pkg-config libwd --cflags --libs
-I/usr/local/include -L/usr/local/lib -lwd

$ pkg-config libcrypto --cflags --libs
-I/usr/local/include -L/usr/local/lib -lcrypto

$ pkg-config libwd_comp --cflags --libs
-I/usr/local/include -L/usr/local/lib -lwd_comp

$ pkg-config libwd --modversion
2.3.37
```
其他例子
`openssl: libcrypto.pc  libssl.pc  openssl.pc`

# pkgconfig怎么用
1. uadk 指定目录安装
安装库到系统目录需要root权限，未必能满足，
同时也可能对系统造成破坏，未必能修复，笔者曾直接替换过glibc，系统直接挂掉．
所以库一般要求能指定目录安装．
比如uadk可以用--prefix指定目录安装
由于不在系统目录，pkgconfig是找不到的，需要export PKG_CONFIG_PATH才行．
同理，程序找不到指定目录的库，需要设LD_LIBRARY_PATH

    ```
$ git clone https://github.com/Linaro/uadk.git
$ cd uadk
$ mkdir build
$ ./autogen.sh
$ ./configure --prefix=$PWD/build
$ make
$ make install
$ ls build
bin  include  lib
$ ls build/lib/pkgconfig/
libwd_comp.pc  libwd_crypto.pc  libwd.pc

$ pkg-config libwd --cflags
Package libwd was not found in the pkg-config search path.
Perhaps you should add the directory containing `libwd.pc'
to the PKG_CONFIG_PATH environment variable
No package 'libwd' found

$ export PKG_CONFIG_PATH=$PWD/build/lib/pkgconfig
$ pkg-config libwd --cflags
-I/home/xxx/uadk/build/include

$ ./build/bin/test_hisi_hpre 
./build/bin/test_hisi_hpre: error while loading shared libraries: libwd.so.2: cannot open shared object file: No such file or directory
$ LD_LIBRARY_PATH=$PWD/build/lib ./build/bin/test_hisi_hpre
failed to init_hpre_global_config, ret -19!

    ```

2. dpdk
像dpdk这种依赖很多第三方库的大型软件，第三方库地址不可能都一样，找不到编译都过不了．
所以dpdk要求所有依赖的库必须同时提供pkgconfig文件．
dpdk某个驱动如果找不到所依赖的库，则自动忽略该驱动，但不影响其他驱动继续编译．
如果uadk指定目录编译，则需要配置PKG_CONFIG_PATH
只有能被pkg-config发现，才会被dpdk检测到．

    ```
$ pkg-config libwd --libs
ckage libwd was not found in the pkg-config search path.
Perhaps you should add the directory containing `libwd.pc'
to the PKG_CONFIG_PATH environment variable
No package 'libwd' found

$ cd dpdk
$ mkdir build
$ meson build
=================
Content Skipped
=================
drivers:
crypto/uadk:	missing dependency, "libwd"

$ export PKG_CONFIG_PATH=$PWD/build/lib/pkgconfig
$ pkg-config libwd --libs
-L/home/xxx/uadk/build/lib -lwd

$ meson build --reconfigure
===============
Drivers Enabled
===============
crypto:
	uadk, ...
$ ninja -C build
    ```

    那dpdk是如何通过pkgconfig文件自动检测到库的呢？
    需要在具体驱动的meson.build加入检测依赖库
    ```
$ vi drivers/crypto/uadk/meson.build
dep = dependency('libwd', required: false, method: 'pkg-config')
if not dep.found()
	build = false
	reason = 'missing dependency, "libwd"'
else
	ext_deps += dep
endif
    ```

3. 库版本检测
pkgconfig可以帮助检测到系统安装依赖库的版本是否满足要求，不满足则忽略或者提示错误．
比如用户给我们报了bug说uadk编译不过．
分析原因是uadk某些测试文件依赖openssl1.1, 但用户系统装是openssl3.0，两者差别很大．
这时候就需要uadk对系统openssl版本信息做检测了．

    ```
$ cd uadk
$ vi configure.ac
PKG_CHECK_MODULES(libcrypto, libcrypto < 3.0 libcrypto >= 1.1,
	     [ AC_DEFINE(HAVE_CRYPTO, 1, [Have crypto])
	       have_crypto=true ],
	     [ have_crypto=false ])
AM_CONDITIONAL([HAVE_CRYPTO], [test "x$have_crypto" = "xtrue"])

$ vi test/Makefile.am
if HAVE_CRYPTO
SUBDIRS += hisi_hpre_test
endif
    ```
    ./configure会判断系统中openssl版本是否是1.1.x, 相应设置flag HAVE_CRYPTO.
    Makefile.am会根据该flag决定是否编译该测试文件．

4. pkgconfig 直接用在Makefile
遇到个问题在x86上某些时候uadk编不过，提示不认识openssl里面的函数符号．
检查也有-lcrypto, 也有头文件．
仔细检查发现是由于Makefile写的不规范，没有-L信息，找不到libcrypto.so,
之前没发现是因为配置了环境变量LIBRARY_PATH，掩盖了问题．
可以在Makefile使用pkgconfig，不依赖环境变量，确保所有环境一致．
比如在configure.ac使用PKG_CHECK_MODULES(libcrypto)后，
使用$(libcrypto_LIBS)替代 -L/usr/local/lib -lcrypto
使用$(libcrypto_CFLAGS)替代 -I/usr/local/include

    ```
$ vi configure.ac
PKG_CHECK_MODULES(libcrypto, libcrypto < 3.0 libcrypto >= 1.1,
	     [ AC_DEFINE(HAVE_CRYPTO, 1, [Have crypto])
	       have_crypto=true ],
	     [ have_crypto=false ])
AM_CONDITIONAL([HAVE_CRYPTO], [test "x$have_crypto" = "xtrue"])

$ vi test/hisi_hpre_test/Makefile.am
test_hisi_hpre_LDADD+= $(libcrypto_LIBS)
    ```

列举的这些只是最近遇到的，可以用pkgconfig解决的．

# pkgconfig怎么写
[具体代码可参考](https://github.com/Linaro/uadk/commit/bda6a068c3a0389f5b2f8e32030c64a8ed29f39f)

贴个范本
```
$ vi lib/libwd_cyrpto.pc.in 
prefix=@prefix@
exec_prefix=@exec_prefix@
libdir=@libdir@
includedir=@includedir@

Name: UADK_libwd_crypto
Description: UADK Crypto Algorithm library
Version: @VERSION@
Libs: -L${libdir} -lwd_crypto
Requires.private: libwd
Cflags: -I${includedir}
Libs.Private: -ldl -lnuma

$ vi Makefile.am

pkginclude_HEADERS = include/wd.h ...
nobase_pkginclude_HEADERS = v1/wd.h ...

pkgconfigdir = $(libdir)/pkgconfig
pkgconfig_DATA = lib/libwd_crypto.pc lib/libwd_comp.pc lib/libwd.pc
CLEANFILES += $(pkgconfig_DATA)

```
提供几个对外的库，则需要提供几个相应的pkgconfig xx.pc
但并不是直接手动填写具体的信息，而是提供xx.pc.in, 从configure.ac里获取变量自动生成的．
这样诸如prefix是可以根据用户的实际输入动态改变的．
VERSION也是从configure.ac获取自动改变的,不需要每次都修改.

说明几个变量
Requires: xx是当期库的依赖库，用户程序也需要显示-lxx才能工作的，系统需要xx.pc
Requires.private: xx是当前库的隐含依赖库，一般是当前仓库内部之间的依赖，系统需要xx.pc
Libs.Private: xx 是当前库自身的依赖，用户程序不需要-lxx也能工作，系统并没有xx.pc
比如libwd自己需要numa，但用户程序并没有显示使用numa函数, 则使用Libs.Private，
这样即使没有numa.pc也可以, 发现apt-get install安装的库可能没有提供xx.pc，比如numa.

Makefile.am
默认头文件会安装到系统目录比如/usr/local/include
include_HEADERS: /usr/local/include/xx.h
nobase_include_HEADERS: /usr/local/include/v1/xx.h
如果需要安装头文件到package目录,可以使用
pkginclude_HEADERS: /usr/local/include/uadk/xx.h
nobase_pkginclude_HEADERS: /usr/local/include/uadk/v1/xx.h

pkgconfigdir和pkgconfig_DATA是安装相应的xx.pc到指定pkg路径
CLEANFILES是make clean删除生成的xx.pc


个人笔记，错误欢迎指正．
