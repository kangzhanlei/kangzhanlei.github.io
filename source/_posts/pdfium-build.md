---
title: "win10编译pdfium"
author: "kzl"
tags: 
	- pdfium
categories:
	- 工具
date: 2019-04-24T13:35:02+08:00
layout: "post"
---

在win10+vs2019下编译pdfium动态库。

## 准备工作

需要预先安装的有：

* vs2019（最低是2017）
* python
* depot_tools
* ninja
* vpn(proxifer+shadowsocks)

shadowsocks开启本地代理端口1080，proxifer配置代理规则，默认走shadowsocks的1080端口，将所有流量都代理出去。



## 环境设置

cmd以管理员方式打开

```bash
set DEPOT_TOOLS_WIN_TOOLCHAIN=0
set GYP_MSVS_VERSION=2019
```

在需要安装的路径下，执行以下命令：

```bash
mkdir repo
cd repo
gclient config --unmanaged https://pdfium.googlesource.com/pdfium.git
gclient sync
cd pdfium
```

在下载的过程中，可能会遇到几个错误，例如下载clang-format始终下载不了，这个可以忽略，修改`repo/pdfium/DEPS`文件, 将关于`clang_format_win`的hook段删除掉。然后再次执行`gclient sync`重试。

```bash
{
    # Pull clang-format binaries using checked-in hashes.
    'name': 'clang_format_win',
    'pattern': '.',
    'action': [ 'download_from_google_storage',
                '--no_resume',
                '--platform=win32',
                '--no_auth',
                '--bucket', 'chromium-clang-format',
                '-s', 'pdfium/buildtools/win/clang-format.exe.sha1',
    ],
  },
```



## 编译

在`repo/pdfium`中，执行以下命令

```bash
gn args out/Release
```

将会在`repo/pdfium`目录下产生一个`out/Release`子目录，后续编译的文件都在这里。

然后会弹出一个文本编辑器，要求输入编译选项，这里贴我的配置：

```bash
# Build arguments go here.
# See "gn args <out_dir> --list" for available build arguments.
use_goma = false  # Googlers only. Make sure goma is installed and running first.
is_debug = false  # Enable debugging features.

# Set true to enable experimental Skia backend.
pdf_use_skia = false
# Set true to enable experimental Skia backend (paths only).
pdf_use_skia_paths = false

pdf_enable_xfa = false  # Set false to remove XFA support (implies JS support).
pdf_enable_v8 = false  # Set false to remove Javascript support.
pdf_is_standalone = true  # Set for a non-embedded build.
is_component_build = false # Disable component build (must be false)

clang_use_chrome_plugins = false  # Currently must be false.
```

成功之后，执行：

```bash
ninja -C out/Release pdfium
```



最后发现什么都没得到。google一下，才知道pdfium不会生成动态库。所以寻找了很久的解决方案，最后在`repo/pdfium/BUILD.gn`文件中，修改如下：

* 修改`pdfium_common_config`，添加define为： `FPDFSDK_EXPORTS`
* 把`jumbo_static_library`修改为`shared_library`



然后重新执行`ninja -C out/Release pdfium`， 最终得到了pdfium.dll文件。

如果需要构建32位的动态库，需要修改`repo/pdfium/out/Release/args.gn`，添加一行

```bash
target_cpu="x86"
```



## vs构建

也可以使用vs来构建项目。在`repo/pdfium`中执行

```bash
gn gen --ide=vs out/Dev
```



然后会生成一个all.sln的解决方案。vs打开，执行构建，漫长等待之后，也会出现pdfium.dll。
