---
title: cmake中find_package()函数的使用
catalog: true
comments: true
indexing: true
header-img: ../../../../img/article_header/article_header.png
top: false
tocnum: true
date: 2020-05-19 16:58:54
subtitle: find_package()
tags:
  - c/c++
  - cmake
categories:
  - cmake
---

#  cmake中find_package()函数的使用

## 背景

今天在玩一个密码学库，它是使用cmake构建的，编译安装过程很曲折，出现了一些错误。作为一个不怎么熟悉 cmake 的新手，单单一个 find_package() 函数就让我花费了很多时间。



## find_package()函数

我们在使用 cmake 编译某个程序的时候，经常会提示找不到某个所依赖的库，那么这是时候我们就需要检查我们引入依赖库的路径对不对了， Cmake中一个自动寻找函数find_package()可以帮我们实现这个功能。

**find_package可以根据cmake内置的.cmake的脚本去找相应的库的模块，也就是相关的h、a、so文件位置。**

一般我们会这么写，来引用已经安装好的库：

```cmake
# CMakeList.txt
list(INSERT CMAKE_MODULE_PATH 0 "${CMAKE_SOURCE_DIR}/../cmake")
find_package(NTL REQUIRED)
find_package(GMP REQUIRED)
find_package(GMPXX REQUIRED)
```



## 问题来了

但是这么使用的前提在于这些库的开发者已经给我们提供了相关的 .cmake 脚本。比如上面的的NTL、GMP、GMPCXX库都是没有提供 .cmake脚本的，在安装的时候只会将相关的头文件拷贝到 /usr/local/include，将库文件(a文件或so文件)拷贝到 /usr/local/lib。这两个目录是默认目录，也可以是自己指定的安装目录。

那么针对于这种没有提供 .cmake 的库，如果我们直接这么使用，会出现类似下面的错误：

```shell
By not providing "FindlibGMP.cmake" in CMAKE_MODULE_PATH this project has
  asked CMake to find a package configuration file provided by "libGMP", but
  CMake did not find one.

  Could not find a package configuration file provided by "libGMP" with any
  of the following names:

    libGMPConfig.cmake
    libgmp-config.cmake

  Add the installation prefix of "libGMP" to CMAKE_PREFIX_PATH or set
  "libGMP_DIR" to a directory containing one of the above files.  If "libGMP"
  provides a separate development package or SDK, be sure it has been
  installed.
```

这是由于找不到对应的 .cmake 模块。开发者没有提供，那么我们可以自己写：

1. 在 /usr/local/lib 目录下常见文件夹 cmake
2. 在 /usr/local/lib/cmake 下创建一个以库命名的文件夹，如 libGMP

```
➜  ~ ls /usr/local/lib/cmake
libGMP  libGMPXX  libNTL 
```

​3. 创建 .cmake 文件 (libGMPConfig.cmake)，内容如下：
```
SET(LAPACK_DIR /usr/local/lib/) 
SET(LAPACK_INCLUDE_DIRS /usr/local/include) 
SET(LAPACK_LIBRARIES /usr/local/lib)
```
以上指定的是默认安装在 /usr/local/下的库位置，内容可以一致，也可以修改为自己指定的目录。

重新编译，发现不会报错了。

4. cmake 可以是 /usr/local/lib 之外的其它自定义目录，如果是其它目录，那么就需要在 CMakeList.txt 中明确指定，否则将会找不到

```cmake
# CMakeList.txt
set (libGMPXX_DIR 自定义目录)
```

