---
marp: true
size: 16:9
theme: am_red
paginate: true
headingDivider: [2, 3]
footer: \ *easterNday* *Build your own kernel* *November 17, 2023*
---

<!-- _class: cover_d -->
<!-- _paginate: "" -->
<!-- _footer: 阅读广而精取，厚积而薄出 -->

# Build your own kernel image

## Use "Github Actions" for kernel cloud compilation

@easterNday
Warehouse address: [https://github.com/DogDayAndroid/Android-Builder](https://github.com/DogDayAndroid/Android-Builde)

<!-- _header: Table of Contents<br>CONTENTS<br>![](https://github.com/DogDayAndroid/Android-Builder/raw/main/.assets/logo.svg)-->
<!-- _class: toc_b -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

- [What is a kernel? ](#3)
- [How to get the kernel source code corresponding to the device? ](#5)
- [How to select the compilation toolchain? ](#10)
- [How to compile the kernel? ](#16)
- [How to package the kernel? ](#23)
- [How to use Github Action to complete kernel compilation? ](#27)
- [How to enable KernelSU support? ](#45)
- [How to enable Docker support? ](#45)
- [What projects are referenced by this project? ](#46)

## Introduction to Android Kernel

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

## Kernel for Android

> A complete operating system, including three main parts: kernel, function library, and user interface.

**Strictly speaking, Android is not a Linux distribution, but the Linux kernel is essential for Android. **

The startup of an Android device is divided into three stages:

- **Bootloader:** After the device is powered on, it will first start executing the boot code of the processor's on-chip `ROM`, search for the `Bootloader` code, load it into the memory and complete the hardware initialization.

- **Linux Kernel:** After the hardware is initialized, the `Linux` kernel code is loaded into the memory, initializes various software and hardware environments, loads drivers, and mounts the root file system.

- **Android System Service:** And executes `init.rc` to notify `Android` to start.

Therefore, the `Android` system is actually a series of system service processes running on the `Linux Kernel`.

## Search for kernel source code

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

## Query device code

<!-- _class: cols-2-64 -->

<div class="ldiv">

> The device code is usually essential when compiling the kernel

Execute on the Android device terminal (`adb shell`):

```bash
getprop | grep device
```

Look for text with the words `ro.xx.device` in the returned results. The content inside is your device code, for example:

```bash
# The phone code is thyme
[ro.product.device]: [thyme]
```

</div>

<div class="rdiv">

For example, I query **Xiaomi 10S**:

```bash
❯ adb shell thyme:/ $ getprop | grep device [bluetooth.device.class_of_device]: [90,2,12] [bluetooth.profile.hid.device.enabled]: [true] [cache_key.bluetooth.bluetooth_device_get_bond_state]: [-1781083723495810673] [cache_key.system_server.device_policy_man ager_caches]: [-1749000785485656421] [debug.tracing.battery_stats.device_idle]: [0] [debug.tracing.device_state]: [0:DEFAULT] [ro.boot.boot_devices]: [soc/1d84000.ufshc] [ro.boot.bootdevice]: [1d84000.ufshc] [ro.frp.pst]: [/dev/block/bootdevice/by-name/frp] [ro.lineage.device]: [thyme] [ro.opa.eligible_device]: [true] [ro.product.bootimage.device]: [thyme] [ro.product.device]: [thyme] [ro.product.mod_device]: [thyme_global] [ro.product .odm.device]: [thyme] [ro.product.product.device]: [thyme] [ro.product.system.device]: [thyme] [ro.product.system_ext.device]: [thyme] [ro.product.vendor.device]: [thyme] [ro.product.vendor_dlkm.device]: [thyme] [sys.usb.mtp.device_type]: [3]
```

</div>

## Get device architecture

> Device architecture is also a necessary part of the compilation process

The method to query the device architecture is also very simple.

Execute on the Android device terminal (`adb shell`):

```bash
uname -m
```

For example, I query **Xiaomi 10S**:

```
thyme:/ $ uname -m
aarch64
```

The device architecture is displayed as `aarch64`, which means that my device is of `aarch64` architecture.

## Get the device kernel version

Execute on the Android device terminal (`adb shell`):

```bash
uname -r
```

The output content is in the format of:

- [version].[patch version].[subversion number]-[kernel identifier]-[commit record]

For example, I query **Xiaomi 10S**:

```bash
thyme:/ $ uname -r
4.19.157-Margatroid-g2b220a0a942c
```

The corresponding kernel version is displayed as `4.19.157`

## Get kernel source code

<!-- _class: cols-2 -->

<div class="ldiv">

The general format of kernel source code is `[android_]kernel_device manufacturer_cpu/codename`.

For example, the code name of **Xiaomi 10S** is `thyme`, the CPU model is `sm8250`, and the manufacturer is `xiaomi`, then the search format should be the following:

- kernel_xiaomi_thyme
- kernel_xiaomi_sm8250
- android_kernel_xiaomi_thyme
- android_kernel_xiaomi_sm8250

Using the above keywords to search on `Github`, you can generally find the corresponding source code.

</div>

<div class="rdiv">
In addition, we can also obtain it through the open source code of various mobile phone manufacturers.

| Ways | Detailed introduction |
| ---------------- | :------------------------------------------------------------------ |
| Xiaomi kernel open source | [Xiaomi kernel open source](https://github.com/MiCode/Xiaomi_Kernel_OpenSource/) |
| Huawei open source code | [Huawei open source code](https://consumer.huawei.com/en/opensource/) |
| Go to the mobile phone community to find the source code | [XDA forum](https://forum.xda-developers.com/) |

~~<u>It is worth noting that, generally speaking, the codes of various mobile phone manufacturers will be castrated and cannot be used out of the box. </u>~~

</div>

## Cross compiler selection

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

## Definition of cross compilation

#### What is cross compilation (Cross Compile)?

- The so-called **"cross compilation"** means that the development compilation platform for compiling source code and the target operating platform for executing the compiled source code program are two different platforms.

#### Why use cross compilation?

- Native compilation cannot be implemented on the target platform, but the CPU architecture or operating system of the platform capable of implementing source code compilation is different from that of the target platform.

## Various CPU architectures

Through the previous content, we have obtained the CPU architecture of our device, which can be classified into the following categories:

- **arm:** 32-bit Arm architecture, including `arm`, `arm32`, `armv7`

- **arm64:** 64-bit Arm architecture, including `arm64`, `aarch64`

- **x86:** 32-bit Intel x86 architecture, including `x86`, `i386`, `i686`

- **x86_64:** 64-bit Intel x86 architecture, including `x86_64`, `amd64`

Generally speaking, the model of our handheld device is generally one of `arm` and `arm64`. Of course, other situations are not ruled out (the Android system used on computers such as Phoenix OS is x86 or x86_64 architecture).

## Triple format of cross-compilation toolchain

Generally speaking, the naming convention of binary files of cross-compilation toolchain is: `{arch}-{vendor}-{sys}-{abi}`

They correspond to the following contents respectively:

- **arch:** corresponding device architecture;

- **vendor:** vendor name, which is generally omitted or not available;

- **sys:** corresponding system, which is generally `linux` when `Android` is compiled;

- **abi:** binary interface between two program units, which is generally `gnu` or `gnueabi` when `Android` is compiled.

Some compilation toolchains omit the `vendor` or `abi` part. Some common naming rules for `Android` compilation are:

- aarch64-linux-gnu-
- arm-linux-gnueabi-
- aarch64-linux-android-
- arm-linux-androideabi-

## Selection of cross-compilation toolchain

> Using an unmatched compiler may result in a failure to boot or even compilation failure.

Tools for building the Android kernel can only be pulled from the Android source code, and there are version restrictions. Too new or too old will not work..

Generally speaking, the compilation toolchain for 2023 models should all be in the form of `clang` + `gcc`, but for some older models, only `gcc` compilation may be supported. Here are some device descriptions:

CAF models (Qualcomm devices except Google):

- `3.18`, `4.4`, `4.9` kernels all use Google's `gcc` by default

- `4.14` and above kernels use `clang` + `gcc` by default

Google models (Pixel series):

- Starting with `Pixel2`, `clang` is used for compilation, and `Pixel3` starts using `clang`'s `LTO` optimization

## Download of cross-compilation toolchain

Generally speaking, we can pull and download the cross-compilation toolchain we need through the repository of third-party systems (for example: `LineageOS`, `PixelExperience`) or `Google` official.

- LineageOS:
- - **Clang** [ux-x86_aarch64_aarch64-linux-android-4.9](https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9)
- - **Gcc for ARM:** [LineageOS/android_prebuilts_gcc_linux-x86_arm_arm-linux-androideabi-4.9](https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_arm_arm-linux-androideabi-4.9)

- Google:
- - **Clang:** [platform/prebuilts/clang/host/linux-x86](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86)
- - **Gcc for ARM64:** [platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9](https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9)
- - **Gcc for ARM:** [platform/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9](https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9)

Please note that the above is just an example. If there is a problem with kernel compilation, you may need to change the specific `clang` and `gcc` versions.

## Kernel compilation

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

## Environment preparation

We assume that you are using `Ubuntu` for kernel compilation development locally. We need to install some necessary compilation tools:

```bash
sudo apt update
sudo apt-get install -y bc bison build-essential ccache curl flex g++-multilib gcc-multilib git git-lfs gnupg gperf \
lib32ncurses5-dev lib32readline-dev lib32z1-dev libelf-dev liblz4-tool libncurses5 libncurses5-dev \
libsdl1.2-dev libssl-dev libxml2 libxml2-utils lzop pngcrush rsync \
schedtool squashfs-tools xsltproc zip zlib1g-dev imagemagick
```

Or if you use `ArchLinux` for development, you only need to install:

```bash
paru -S aosp-devel linageos-devel
```

## Kernel source code cloning

To compile the kernel, we first need to clone the kernel source code of the corresponding device. Here we take **Xiaomi 10S** as an example.

The repository I chose is [https://codeberg.org/DogDayAndroid/android_kernel_xiaomi_thyme](https://codeberg.org/DogDayAndroid/android_kernel_xiaomi_thyme)

Execute the following command in the terminal:

```bash
git clone https://codeberg.org/DogDayAndroid/android_kernel_xiaomi_thyme
```

Wait for the repository to be pulled, and the kernel source code can be cloned.

## Check the device defconfig in the kernel source code

In the **kernel source code search** section, we have obtained the device **code** and **architecture**. Generally speaking, the configuration file will be defined according to these contents.

For example, my **Xiaomi 10S** is codenamed **thyme**, and the architecture is **aarch64**, which is **arm64**. For specific architectures, please refer to the following:

- **arm:** 32-bit Arm architecture, including `arm`, `arm32`, `armv7`

- **arm64:** 64-bit Arm architecture, including `arm64`, `aarch64`

Generally speaking, the `defconfig` file is located in the following path:

- `<kernel source folder>/arch/<device architecture>/configs/`
- `<kernel source folder>/arch/<device architecture>/configs/vendor/`

In the kernel source code I just cloned, the `defconfig` file path corresponding to **thyme** is `arch/arm64/configs/thyme_defconfig`. Please make sure you can find your `defconfig` file path.

## Configuration of cross-compilation toolchain

<!-- _class: smalltext -->

Here I use **LineageOS**'s cross-compilation toolchain for demonstration:

- **Clang:** [LineageOS/android_prebuilts_clang_kernel_linux-x86_clang-r416183b](https://github.com/LineageOS/android_prebuilts_clang_kernel_linux-x86_clang-r416183b)
- **Gcc for ARM64:** [LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9](https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9)
- **Gcc for ARM:** [LineageOS/android_prebuilts_gcc_linux-x86_arm_arm-linux-androideabi-4.9](https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_arm_arm-linux-androideabi-4.9)

Execute in terminal:

```bash
git clone https://github.com/LineageOS/android_prebuilts_clang_kernel_linux-x86_clang-r416183b clang
git clone https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9 gcc64
git clone https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_arm_arm-linux-androideabi-4.9 gcc32
```

Then temporarily add the compilation toolchain to our environment variables:

```bash
export PATH="$PWD/clang/bin:$PWD/gcc64/bin:$PWD/gcc32/bin:$PATH"
```

## Start building the kernel

First, we need to understand the parameters of our build:

| Parameters | Description | General parameters |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| `CC` | Specify the compiler to be used, because `make` uses `gcc` by default, so in fact, this parameter is only used when you use `clang` for compilation | `clang` |
| `CROSS_COMPILE` | Your main cross-compilation chain tool, if you only use `gcc` for compilation, please specify the parameter as `aarch64-linux-android-`, the same for 32-bit | `aarch64-linux-gnu-` |
| `CLANG_TRIPLE` | Only when using `clang` is only needed for compilation. It is used to specify the toolchain to be used when `clang` is not effective. However, this parameter is basically not required when using the tools mentioned in the previous section. | `aarch64-linux-gnu-` |
| `CROSS_COMPILE_ARM32` | This parameter is only required when compiling a 32-bit kernel or a kernel with a vdso patch. | `arm-linux-gnueabi-` |
| `CROSS_COMPILE_COMPAT` | Similar to the parameter `CROSS_COMPILE_ARM32`, but the kernel version is4.19 and later versions should use this parameter instead of `CROSS_COMPILE_ARM32` | `arm-linux-gnueabi-` |

## Start building the kernel

<!-- _class: tinytext -->

Here is a general build script:

```bash
#!/bin/bash
args="-j$(nproc --all) O=out ARCH=arm64 CC=clang CLANG_TRIPLE=aarch64-linux-gnu- CROSS_COMPILE=aarch64-linux-android- \
CROSS_COMPILE_ARM32=arm-linux-androideabi- LD=ld.lld AR=llvm-ar NM=llvm-nm OBJCOPY=llvm-objcopy OBJDUMP=llvm-objdump READELF=llvm-readelf \
OBJSIZE=llvm-size STRIP=llvm-strip LDGOLD=aarch64-linux-gnu-ld.gold LLVM_AR=llvm-ar LLVM_DIS=llvm-dis"
make ${args} <configName>
make ${args}
```

Where `ARCH=arm64` specifies the architecture as `arm64`, the other parameters correspond to the contents in the above table.

Where `<copnfigName>` is our `defconfig` file, for example:

- **Xiaomi 10S** corresponds to the `defconfig` file path `arch/arm64/configs/thyme_defconfig`, fill in `thyme_defconfig` here;

- Some projects may be located in `arch/arm64/configs/vendor/thyme_defconfig`, in this case, `vendor/thyme_defconfig` should be filled in.

This build script specifies the architecture as `arm64`, uses `clang` for compilation, and specifies the corresponding cross-compilation tool.

It is worth noting that this script is not universal, and the specific kernel needs to be analyzed and modified by yourself.

## Kernel packaging

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

## Get kernel compilation product

Through the content in the previous section, we can complete the kernel compilation. At this time, we need to consider how to package the kernel.

Generally speaking, the kernel compiled product is located in the `arch/<device architecture>/boot/` folder.

- In the compiled product, there may be files such as `Image`, `Image.gz`, `Image.gz-dtb`, and what we need is the file with `dtb`, that is, `Image.gz-dtb`. However, the compiled product may also be `Image.lz4-dtb` or `Image-dtb`, but in short, it is always right to bring `dtb`.

- For some kernel source codes, the compiled product may be in the form of `Image` + `independent dtb file / dtbo.img`, and it is also OK if it is these two files.

## Anykernel3

`Anykernel` is a kernel flashing tool originally written by `koush`, and later taken over by `osm0sis` and iterated many times.

The project address is: [https://github.com/osm0sis/AnyKernel3](https://github.com/osm0sis/AnyKernel3)

First, we clone the project and make some modifications to the script:

```bash
git clone https://github.com/osm0sis/AnyKernel3 AnyKernel3
# Modify the script
sed -i 's/do.devicecheck=1/do.devicecheck=0/g' AnyKernel3/anykernel.sh
sed -i 's!block=/dev/block/platform/omap/omap_hsmmc.0/by-name/boot;!block=auto;!g' AnyKernel3/anykernel.sh
sed -i 's/is_slot_device=0;/is_slot_device=auto;/g' AnyKernel3/anykernel.sh
```

Please note that these modifications make the packaged flash package available for any device to flash, which is risky. If you need to use the flash package safely, please refer to the official documentation of `Anykernel3`.

## Flash package packaging and flashing

According to the instructions in **Get kernel compilation product**, we copy `Image-dtb` or `Image` + `dtb` or `Image` + `dtbo.img` to the folder of `Anykernel3`, and then use `zip` to compress it to complete the flash package. The compression command is as follows:

```bash
cd AnyKernel3/
zip -q -r "AnyKnerl3.zip" *
```

After packaging, we can enter the device's `TWRP` to flash the compressed package.

## Compile kernel using Github Action

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

## Project address

[https://github.com/DogDayAndroid/Android-Builder](https://github.com/DogDayAndroid/Android-Builder)

Currently, this project will support the following:

- Use `Github Actions` to build kernel
- After cloning this project to your local computer, you can use scripts to automatically build `LineageOS`
- Use `Github Actions` to build `TWRP`

The specific contents of each project will be displayed separately in each folder. You can enter these folders to view their respective readme files to learn how to use them.

#### License

[![by-nc-sa](https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png)](http://creativecommons.org/licenses/by-nc-sa/4.0/)
This work is licensed under a [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-nc-sa/4.0/).

## How to use

- `Fork` this project on GitHub

- Modify `config/*.config.json` files through the Github webpage or pull them to local, and submit the changes

- View the `Action` page of the Github webpage, find `Build kernels` and `Run workflow`

- Wait for the compilation to complete, then enter the corresponding page to download the compiled product

## Compilation process

<!-- mermaid.js -->
<script src="https://unpkg.com/mermaid@8.1.0/dist/mermaid.min.js"></script>
<script>mermaid.initialize({startOnLoad:true});</script>

<div class="mermaid">
timeline
Kernel compilation process
section Read configuration
Clone repository
Read configuration file to generate matrix
Set compilation date
section Kernel compilation
Prepare : 📐 Increase swap partition to 10G : 😄 Clone repository
: ⭐ Install necessary packages for Ubuntu : 🚄 ccache initialization configuration : 🚅 Restore cache
: 🌟 Clone kernel source code : 💫 Get compilation toolchain
Basic configuration : 😎 Set compilation parameters : 😋 Configure KernelSU : 😋 Configure LXC Docker
: 👍 Start kernel compilation
section File upload and release
Middleware upload : 💛 Upload Image : 💙 Upload Image.gz : 💙 Upload Image.gz-dtb
: 💜 Upload dtb : ❤️ Upload dtbo.img : ☢️ Upload output folder
Anykernel3 packaging : ⏰ Download Anykernel3 : ⏰ Package kernel : 💾 Upload flash package
Release
</div>

## Configuration parameter analysis

Each configuration template consists of the following parts:

| Field name | Description |
| -------------- | ---------------------------------------------------------------------------------------------- |
| kernelSource | Information about the kernel source code, including name, repository address, branch, and device type. |
| toolchains | An array containing information about the toolchain to be used, including repository address, branch, and name. |
| enableCcache | A Boolean value indicating whether the compilation tool named `ccache` is used to speed up compilation. |
| params | An object containing information about the build parameters, including architecture type, cross compiler, compiler, and other information. |
| AnyKernel3 | An object containing information about building the kernel flash package, including the `AnyKernel3` repository address, branch, and other information used. |
| enableKernelSU | A Boolean value indicating whether the kernel patch named `KernelSU` is used. |
| enableLXC | A Boolean value indicating whether to enable `Docker` support. |

## Kernel Source Configuration (kernelSource)

```json
"kernelSource": {
"name": "", // Your favorite name, no effect, generally set to device name + compilation tool chain version
"repo": "", // Kernel source repository address
"branch": "", // Corresponding branch name of kernel source repository
"device": "", // Corresponding device number
"defconfig": "" // Corresponding defconfig file relative path
}
```

The `name` part has no effect on the entire compilation process, so in theory you can set it at will.

`repo`, `branch` are used to clone kernel source code. We will clone all submodules under the source code by default to ensure the integrity of the kernel.

The content filled in `defconfig` is the relative path of your `defconfig` file The relative path to the `arch/arm64/configs` or `arch/arm/configs` folder. The reason for this is that some `defconfig` files may exist in subdirectories. When `make` is called, we need to explicitly specify the relative path.

## Kernel source code configuration (kernelSource)

Here is a basic example:

```json
"kernelSource": {
"name": "Mi6X",
"repo": "https://github.com/Diva-Room/Miku_kernel_xiaomi_wayne",
"branch": "TDA",
"device": "wayne",
"defconfig": "vendor/wayne_defconfig"
}
```

This kernel is the kernel source code of **Xiaomi 6X**. After opening its `Github` address on the web page, we can see that its main branch is `TDA`, and its `defconfig` file is located in `/arch/arm64/configs/vendor/wayne_defconfig`, so set `defconfig` to `vendor/wayne_defconfig`.

### Toolchain configuration (toolchains)

Cross-compilation toolchain is an important tool for us to compile the kernel, but the download form of the compilation toolchain is varied. You can use `git` to pull and download, or you can get it by downloading. Therefore, we have adapted for different acquisition methods:

- Use `Git` to pull the compilation toolchain
- Use `Wget` to download the compilation toolchain

### Toolchain configuration (toolchains) - Use `Git` to pull the compilation toolchain

```json
"toolchains": [
{
"name": "proton-clang",
"repo": "https://github.com/kdrag0n/proton-clang",
"branch": "master",
"binaryEnv": ["./bin"]
}
]
```

This part of the configuration is actually similar to the kernel source code configuration. We will also use the following command to pull the source code from the repository:

```bash
git clone --recursive --depth=1 -j $(nproc) --branch <branch> <repo> <name>
```

However, a new `binaryEnv` is added in this section, which is used to add global environment variable settings to our compilation toolchain, such as `./bin` here. After adding the content, the `bin` folder under the compilation toolchain will be added to the environment variable.

### Toolchain Configuration (toolchains) - Use `Wget` to download the compilation toolchain

In this way, we can get the compilation toolchain compressed package in the format of `.zip` | `.tar` | `.tar.gz` | `.rar`.

```json
"toolchains": [
{
"name": "clang",
"url": "https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/master-kernel-build-2022/clang-r450784d.tar.gz",
"binaryEnv": ["./bin"]
},
{
"name": "gcc",
"url": "https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/+archive/refs/tags/android-12.1.0_r27.tar.gz",
"binaryEnv": ["bin"]
}
]
```

`Action` will download and unzip them, and there will also be `binaryEnv` in this part, Its function is similar to the above function, so it will not be described in detail.

### Toolchain configuration (toolchains)

These two methods are not mutually exclusive. If we need both `Git` pull and `Wget` download compilation toolchain, we can also mix them like the following configuration:

```json
"toolchains": [
{
"name": "clang",
"repo": "https://gitlab.com/ThankYouMario/android_prebuilts_clang-standalone/",
"branch": "11",
"binaryEnv": ["bin"]
},
{
"name": "gcc",
"url": "https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/+archive/refs/tags/android-12.1.0_r27.tar.gz",
"binaryEnv": ["bin"]
}
]
```

## Compilation parameters (params)

Usually when we compile the kernel locally, we use a compilation command similar to the following:

```sh
make -j$(nproc --all) \
O=out \
ARCH=arm64 \
CC=clang \
CLANG_TRIPLE=aarch64-linux-gnu- \
CROSS_COMPILE=aarch64-linux-gnu- \
CROSS_COMPILE_ARM32=arm-linux-gnueabi-
```

## Compilation parameters (params)

Therefore, our compilation parameter configuration is also configured in a similar way:

```json
"params": {
"ARCH": "arm64",
"CC": "clang",
"externalCommands": {
"CLANG_TRIPLE": "aarch64-linux-gnu-",
"CROSS_COMPILE": "proton-clang/bin/aarch64-linux-gnu-",
"CROSS_COMPILE_ARM32": "proton-clang/bin/arm-linux-gnueabi-"
}
}
```

`-j` and `O=out` are automatically configured by the compilation script, so they do not need to be set in the configuration. `ARCH` and `CC` correspond to the command part above, and other more parameters correspond to the `externalCommands` part.

## Compilation parameters (params)
<!-- _class: tinytext -->

```json # Template for markw(Xiaomi4) "params": { "ARCH": "arm64", "CC": "clang", "externalCommands": { "CLANG_TRIPLE": "aarch64-linux-gnu-", "CROSS_COMPILE": "aarch64-linux-android-", "CROSS_ COMPILE_ARM32": "arm-linux-androideabi-", "LD": "ld.lld", "AR": "llvm-ar", "NM": "llvm-nm", "OBJCOPY": "llvm-objcopy", "OBJDUMP": "llvm-objdump", "READELF": "llvm-readelf", "OBJSIZE": "llvm-size ", "STRIP": "llvm-strip",
"LDGOLD": "aarch64-linux-gnu-ld.gold",
"LLVM_AR": "llvm-ar",
"LLVM_DIS": "llvm-dis",
"CONFIG_THINLTO": ""
}
}
```

## Enable `ccache` to speed up compilation

During kernel compilation, repeated compilation will take up a lot of time. `ccache` allows us to reuse some of the middleware caches from previous compilations to speed up compilation. For example, the compilation command in the previous section should be: after enabling `ccache`:

```sh
make -j$(nproc --all) \
O=out \
ARCH=arm64 \
CC="ccache clang" \
CLANG_TRIPLE=aarch64-linux-gnu- \
CROSS_COMPILE=aarch64-linux-gnu- \
CROSS_COMPILE_ARM32=arm-linux-gnueabi-
```

## Enable `ccache` to accelerate compilation

This introduces a separate configuration parameter `enableCcache`. We only need to set `enableCcache` to `true` during configuration to implement the same command:

```json
"enableCcache": true,
"params": {
"ARCH": "arm64",
"CC": "clang",
"externalCommands": {
"CLANG_TRIPLE": "aarch64-linux-gnu-",
"CROSS_COMPILE": "aarch64-linux-android-",
"CROSS_COMPILE_ARM32": "arm-linux-androideabi-"
}
}
```

## Kernel flash package configuration (AnyKernel3)

Currently this project only supports `AnyKernel3`, its configuration is as follows:

```json
"AnyKernel3": {
"use": true,
"release": true,
"custom": {
"repo": "https://github.com/easterNday/AnyKernel3/",
"branch": "thyme"
}
}
```

## Kernel Flashing Package Configuration (AnyKernel3)

In this configuration, I used the custom `AnyKernel3` for packaging. If you don't want to `fork` an additional repository to implement it, you can choose to delete the `custom` field to use the original `AnyKernel3` to package your kernel. The configuration after deletion is as follows:

```json
"AnyKernel3": {
"use": true,
"release": true
}
```

The `use` in the configuration indicates whether you use `AnyKernel3` for packaging, and `release` Indicates whether you will release the packaged flash package. `release` is effective only when `AnyKernel3` is set to `true`, otherwise it defaults to `false`.

## Additional compilation parameter settings

#### KernelSU

Use `"enableKernelSU": true,` to control whether to enable `KernelSU`, set to `false` to disable.

#### LXC Docker

Use `"enableLXC": false` to control whether to enable `Docker` support, set to `true` to enable.

## Tutorials and references

<!-- _class: trans -->
<!-- _footer: "" -->
<!-- _paginate: "" -->

## Tutorial

- [Compile and customize an awesome Android kernel yourself](https://parrotsec-cn.org/t/topic/2168)
- [To make Android phones more power-efficient and smooth, you can try "flashing the kernel"](https://sspai.com/post/56296)
- [[Kernel-oriented] Cross-compiler selection](https://www.akr-developers.com/d/129)
- [[Vernacular version] ClangBuiltLinux Clang usage](https://www.akr-developers.com/d/121)
- [Neutron-clang compilation instructions](https://github.com/Neutron-Toolchains/clang-build-catalogue#building-linux)
- [[Kernel-oriented] On how to flash the kernel elegantly](https://www.akr-developers.com/d/125)

## Reference

- [DogDayAndroid/KSU_Thyme_BuildBot](https://github.com/DogDayAndroid/KSU_Thyme_BuildBot/blob/main/build.sh): The local compilation script used by the kernel I compiled myself.
- [UtsavBalar1231/Drone-scripts](https://github.com/UtsavBalar1231/Drone-scripts): A compilation script used by many people, and some of my code is also referenced from here.
- [EndCredits/kernel_xiaomi_sm7250](https://github.com/EndCredits/kernel_xiaomi_sm7250/blob/android-4.19-main/build.sh): The same compilation script, but the compilation chain is not provided, but I also refer to the script process.
- [xiaoleGun/KernelSU_Action](https://github.com/xiaoleGun/KernelSU_Action): `KernelSU` compilation script, also for reference.

---

<!-- _class: lastpage -->
<!-- _footer: "" -->

###### ～ Welcome to communicate ～