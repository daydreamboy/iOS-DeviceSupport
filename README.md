# iOS-DeviceSupport
[TOC]



## 1、Symbols

Symbols folders in ~/Library/Developer/Xcode/iOS DeviceSupport

| name                   |
| ---------------------- |
| 9.2.1 (13D15)          |
| 9.3.5 (13G36)          |
| 10.3.2 (14F89)         |
| 10.3.3 (14G60)         |
| 11.0 (15A372)          |
| 11.0.1 (15A402)        |
| 11.0.1 (15A403)        |
| 11.4 (15F79)           |
| 12.1.4 (16D57)         |
| 12.4.1 (16G102) arm64e |
| 12.4.1 (16G102)        |
| 12.4.8 (16G201)        |
| 13.1 (17A5821e)        |
| 13.1.2 (17A861)        |
| 13.3 (17C54)           |
| 13.4.1 (17E262)        |
| 13.5.1 (17F80)         |
| 13.6 (17G68) arm64e    |
| 13.6.1 (17G80)         |
| 13.7 (17H35) arm64e    |
| 14.0 (18A373) arm64e   |
| 14.0.1 (18A393) arm64e |
| 14.2.1 (18B121) arm64e |
| 14.3 (18C66)           |
| 14.4 (18D52) arm64e    |
| 14.4.1 (18D61) arm64e  |
| 14.4.2 (18D70) arm64e  |
| 14.5 (18E199) arm64e   |
| 14.5.1 (18E212) arm64e |
| 14.6 (18F72) arm64e    |
| 14.7.1 (18G82) arm64e  |
| 14.8 (18H17) arm64e    |
| 15.0.2 (19A404) arm64e |
| 15.1 (19B74) arm64e    |




~~Google Drive: https://drive.google.com/drive/folders/1pkdZpu0bME4wXFd1CQhuIhjUkWVKNDkO~~



## 2、从固件中获取符号文件

iOS固件会有系统符号文件，通过一些工具，可以从固件中提取出符号文件，



### (1) 下载对应版本的固件

国内有几个网址，可以提供固件下载

* https://www.i4.cn/firmware_iPhone_iPhone%20X_____.html
* https://iosgujian.com/iPhone/iPhone11/iPhone12,1



以iPhone14,5_15.0.1_19A348_Restore.ipsw为例，下载地址是[这里](https://updates.cdn-apple.com/2021FallFCS/fullrestores/002-09688/07029187-8FB6-4F93-9429-128201E13265/iPhone14,5_15.0.1_19A348_Restore.ipsw)。

ipsw文件一般都比较大，选择网络好和空闲的时间来下载。



### (2) 确定符号文件的位置

将ipsw文件解压，它实际是个zip文件，因此换下后缀名可以解压出来，得到下面的一些文件和文件夹

```shell
$ iPhone14,5_15.0.1_19A348_Restore.ipsw ls -l
total 12722096
-rw-r--r--@  1 wesley_chen  staff  6251791817 Jan  9  2007 018-79898-003.dmg
-rw-r--r--@  1 wesley_chen  staff   121403931 Jan  9  2007 038-42528-641.dmg
-rw-r--r--@  1 wesley_chen  staff   123465243 Jan  9  2007 038-42637-643.dmg
-r--r--r--@  1 wesley_chen  staff      314550 Jan  9  2007 BuildManifest.plist
drwxr-xr-x@ 33 wesley_chen  staff        1056 Jan  9  2007 Firmware
-r--r--r--@  1 wesley_chen  staff        1030 Jan  9  2007 Restore.plist
-rw-r--r--@  1 wesley_chen  staff    16727656 Jan  9  2007 kernelcache.release.iphone14
```

按照文件排序，找到最大的那个dmg文件，这里是018-79898-003.dmg，将它打开。

根据下面的路径，找到dyld_shared_cache_arm64e文件

```shell
/System/Library/Caches/com.apple.dyld
```

还有其他两个文件

* /System/Library/Caches/com.apple.dyld/dyld_shared_cache_arm64e.1
* /System/Library/Caches/com.apple.dyld/dyld_shared_cache_arm64e.symbols



### (3) 获取dsc_extractor工具

有很多工具可以从dyld_shared_cache_arm64e文件提取符号文件，但是dsc_extractor提取的产物，是和Xcode连接真机提取的产物是一样的。

这篇文章的描述[^1]，如下

> [dsc_extractor (source code)](https://opensource.apple.com/source/dyld/). More info [here](https://gist.github.com/NSExceptional/85527151eeec4b0640187a0a165da1cd). It produces the best results among all tools, but without branch islands workaround. Do note this is the exact same output provided by XCode automatically.

所以使用dsc_extractor工具，是最佳选择。但是dsc_extractor工具不是现成的，需要自己编译出来。

有两个文章[^2][^3]介绍如何编译这个工具，按照它们的步骤一步步来



#### a. 下载dyld文件

Apple的opensource网站提供下载dyld文件，地址是：https://opensource.apple.com/tarballs/dyld/

找到最新的dyld文件的下载地址。

执行下面几个命令

```shell
$ wget https://opensource.apple.com/tarballs/dyld/dyld-852.2.tar.gz
$ tar -xvzf dyld-852.2.tar.gz
```

dyld文件，由于更新了很多版本，因此解压后的文件结构可能会不一样。



#### b. 从dsc_extractor.cpp获取extract代码

找到需要的dsc_extractor.cpp，位置如下

```
dyld-852.2/dyld3/shared-cache/dsc_extractor.cpp
```

说明

> 如果有dyld.xcodeproj，可以打开搜一下dsc_extractor.cpp



在dsc_extractor.cpp的代码中，找到下面一段被if条件编译注释掉的代码，如下

```c
#if 0
// test program
#include <stdio.h>
#include <stddef.h>
#include <dlfcn.h>


typedef int (*extractor_proc)(const char* shared_cache_file_path, const char* extraction_root_path,
                              void (^progress)(unsigned current, unsigned total));

int main(int argc, const char* argv[])
{
    if ( argc != 3 ) {
        fprintf(stderr, "usage: dsc_extractor <path-to-cache-file> <path-to-device-dir>\n");
        return 1;
    }

    //void* handle = dlopen("/Volumes/my/src/dyld/build/Debug/dsc_extractor.bundle", RTLD_LAZY);
    void* handle = dlopen("/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/usr/lib/dsc_extractor.bundle", RTLD_LAZY);
    if ( handle == NULL ) {
        fprintf(stderr, "dsc_extractor.bundle could not be loaded\n");
        return 1;
    }

    extractor_proc proc = (extractor_proc)dlsym(handle, "dyld_shared_cache_extract_dylibs_progress");
    if ( proc == NULL ) {
        fprintf(stderr, "dsc_extractor.bundle did not have dyld_shared_cache_extract_dylibs_progress symbol\n");
        return 1;
    }

    int result = (*proc)(argv[1], argv[2], ^(unsigned c, unsigned total) { printf("%d/%d\n", c, total); } );
    fprintf(stderr, "dyld_shared_cache_extract_dylibs_progress() => %d\n", result);
    return 0;
}


#endif
```

上面代码的大致逻辑是：在本地的Xcode.app中找到dsc_extractor.bundle这个动态库，加载到内存，然后通过dlsym拿到dyld_shared_cache_extract_dylibs_progress函数，然后用这个函数对dyld_shared_cache_arm64e文件进行提取。

另外，代码中清楚说明了，如何使用这个命令行工具

> usage: dsc_extractor <path-to-cache-file> <path-to-device-dir>



#### c. 准备extract代码

有了上面现成的代码，那么copy&paste一下，变成自己的dsc_extractor.cpp文件，如下

```cpp
#include <stdio.h>
#include <stddef.h>
#include <dlfcn.h>


typedef int (*extractor_proc)(const char* shared_cache_file_path, const char* extraction_root_path,
                              void (^progress)(unsigned current, unsigned total));

int main(int argc, const char* argv[])
{
    if ( argc != 3 ) {
        fprintf(stderr, "usage: dsc_extractor <path-to-cache-file> <path-to-device-dir>\n");
        return 1;
    }

    //void* handle = dlopen("/Volumes/my/src/dyld/build/Debug/dsc_extractor.bundle", RTLD_LAZY);
    void* handle = dlopen("/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/usr/lib/dsc_extractor.bundle", RTLD_LAZY);
    if ( handle == NULL ) {
        fprintf(stderr, "dsc_extractor.bundle could not be loaded\n");
        return 1;
    }

    extractor_proc proc = (extractor_proc)dlsym(handle, "dyld_shared_cache_extract_dylibs_progress");
    if ( proc == NULL ) {
        fprintf(stderr, "dsc_extractor.bundle did not have dyld_shared_cache_extract_dylibs_progress symbol\n");
        return 1;
    }

    int result = (*proc)(argv[1], argv[2], ^(unsigned c, unsigned total) { printf("%d/%d\n", c, total); } );
    fprintf(stderr, "dyld_shared_cache_extract_dylibs_progress() => %d\n", result);
    return 0;
}
```

编译之前，检查下这个路径是否有dsc_extractor.bundle文件

```shell
$ ls -l /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/usr/lib/dsc_extractor.bundle
-rwxrwxr-x  1 wesley_chen  staff  567024 Apr 10  2021 /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/usr/lib/dsc_extractor.bundle
```



#### d. 生成dsc_extractor命令行工具

执行下面的命令

```shell
$ clang++ -o dsc_extractor dsc_extractor.cpp
```

如果顺利的话，会得到名为dsc_extractor的可执行文件。



#### e. 提前dyld_shared_cache_arm64e文件

执行下面命令

```shell
$ ./dsc_extractor dyld_shared_cache_arm64e arm64e/
0/2369
1/2369
2/2369
3/2369
4/2369
5/2369
...
dyld_shared_cache_extract_dylibs_progress() => 0
```

如果顺利的话，会输出进度，并打印“dyld_shared_cache_extract_dylibs_progress() => 0”。

注意

> 如果没有进度，打印“dyld_shared_cache_extract_dylibs_progress() => 0”，说明存在问题。很可能提取的dyld_shared_cache_arm64e文件的版本，要高于Xcode的版本。比如说上面固件是15.0.1，但是Xcode版本还是12.5，不是Xcode 13.1。尽可能使用最新版本的Xcode，然后执行dsc_extractor工具



















## References

[^1]:https://iphonedev.wiki/index.php/Dyld_shared_cache
[^2]:http://crash.163.com/#news/!newsId=31
[^3]:https://gist.github.com/NSExceptional/85527151eeec4b0640187a0a165da1cd

