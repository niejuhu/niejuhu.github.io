# Android boot image

记录Android boot image相关知识，工具使用等。

## Android如何生成boot image

Android boot image通过命令`make bootimage`生成，其流程大致如下：

android-src/build/core/main.mk

    .PHONY: bootimage
    bootimage: $(INSTALLED_BOOTIMAGE_TARGET)

android-src/build/core/Makefile

    INSTALLED_BOOTIMAGE_TARGET := $(PRODUCT_OUT)/boot.img

    # 将kernel和ramdisk.img通过mkbootimg命令打包到boot.img当中
    $(INSTALLED_BOOTIMAGE_TARGET): $(MKBOOTIMG) $(INTERNAL_BOOTIMAGE_FILES) $(BOOT_SIGNER)
    $(call pretty,"Target boot image: $@")
    $(hide) $(MKBOOTIMG) $(INTERNAL_BOOTIMAGE_ARGS) $(BOARD_MKBOOTIMG_ARGS) --output $@
    $(BOOT_SIGNER) /boot $@ $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_VERITY_SIGNING_KEY) $@
    $(hide) $(call assert-max-image-size,$@,$(BOARD_BOOTIMAGE_PARTITION_SIZE))

    INTERNAL_BOOTIMAGE_ARGS := \
    $(addprefix --second ,$(INSTALLED_2NDBOOTLOADER_TARGET)) \
    --kernel $(INSTALLED_KERNEL_TARGET) \
    --ramdisk $(INSTALLED_RAMDISK_TARGET)

    # 使用cpio将out目录下的root文件夹压缩为ramdisk.img
    $(INSTALLED_RAMDISK_TARGET): $(MKBOOTFS) $(INTERNAL_RAMDISK_FILES) | $(MINIGZIP)
    $(call pretty,"Target ram disk: $@")
    $(hide) $(MKBOOTFS) $(TARGET_ROOT_OUT) | $(MINIGZIP) > $@

android-src/build/core/config.mk

    # mkbootfs实际为cpio，源码：android-src/system/core/cpio
    MKBOOTFS := $(HOST_OUT_EXECUTABLES)/mkbootfs$(HOST_EXECUTABLE_SUFFIX)

    # minigzip为简化gunzip，源码：android-src/external/zlib
    MINIGZIP := $(HOST_OUT_EXECUTABLES)/minigzip$(HOST_EXECUTABLE_SUFFIX)

    # mkbootimg源码：android-src/system/core/mkbootimg
    MKBOOTIMG := $(HOST_OUT_EXECUTABLES)/mkbootimg$(HOST_EXECUTABLE_SUFFIX)

以上流程执行了两个操作：

*   将out目录下的root文件夹，即Android跟文件系统目录压缩为ramdisk.img

        mkbootfs out-dir/ | minigzip
        # 相当于 find out-dir | cpio -o -H newc | gzip > ramdisk.img

*   将kernel与ramdisk.img打包成boot.img

        mkbootimg [options and files] --output boot.img

mkbootimg生成的boot.img文件结构在代码里写的很清楚：

    #define BOOT_MAGIC "ANDROID!"
    #define BOOT_MAGIC_SIZE 8
    #define BOOT_NAME_SIZE 16
    #define BOOT_ARGS_SIZE 512
    #define BOOT_EXTRA_ARGS_SIZE 1024

    struct boot_img_hdr
    {
        unsigned char magic[BOOT_MAGIC_SIZE];

        unsigned kernel_size;  /* size in bytes */
        unsigned kernel_addr;  /* physical load addr */

        unsigned ramdisk_size; /* size in bytes */
        unsigned ramdisk_addr; /* physical load addr */

        unsigned second_size;  /* size in bytes */
        unsigned second_addr;  /* physical load addr */

        unsigned tags_addr;    /* physical addr for kernel tags */
        unsigned page_size;    /* flash page size we assume */
        unsigned unused[2];    /* future expansion: should be 0 */

        unsigned char name[BOOT_NAME_SIZE]; /* asciiz product name */

        unsigned char cmdline[BOOT_ARGS_SIZE];

        unsigned id[8]; /* timestamp / checksum / sha1 / etc */
        /* Supplemental command line data; kept here to maintain
         * binary compatibility with older versions of mkbootimg */
        unsigned char extra_cmdline[BOOT_EXTRA_ARGS_SIZE];
    };

    /*
    ** +-----------------+
    ** | boot header     | 1 page
    ** +-----------------+
    ** | kernel          | n pages
    ** +-----------------+
    ** | ramdisk         | m pages
    ** +-----------------+
    ** | second stage    | o pages
    ** +-----------------+
    **
    ** n = (kernel_size + page_size - 1) / page_size
    ** m = (ramdisk_size + page_size - 1) / page_size
    ** o = (second_size + page_size - 1) / page_size
    **
    ** 0. all entities are page_size aligned in flash
    ** 1. kernel and ramdisk are required (size != 0)
    ** 2. second is optional (second_size == 0 -> no second)
    ** 3. load each element (kernel, ramdisk, second) at
    **    the specified physical address (kernel_addr, etc)
    ** 4. prepare tags at tag_addr.  kernel_args[] is
    **    appended to the kernel commandline in the tags.
    ** 5. r0 = 0, r1 = MACHINE_TYPE, r2 = tags_addr
    ** 6. if second_size != 0: jump to second_addr
    **    else: jump to kernel_addr
    */

## 解压Android boot.img以及还原根文件系统

按照Android boot.img的文件结构进行解压，生成kernel以及ramdisk.img：

    unpackbootimg [--outdir <outdir>] <boot.img>
    # 在outdir中生成<boot.img>-kernel,<boot.img>-ramdisk,以及参数文件<boot.img>-args

进一步可以还原ramdisk文件为根文件目录结构：

    mkdir root && cd root && gunzip -c ../<boot.img>-ramdisk | cpio -i

## 重新打包Android boot.img

如果想要修改kernel或者ramdisk.img，就需要重新打包，首先重新压缩ramdisk.img(如果有修改）

    cd root && find . | cpio -o -H newc | gzip > ../ramdisk.img

然后使用原参数重新生成boot.img:

    packbootimg --kernel <your-kernel> --ramdisk <your-ramdisk.img> --args <args-file-from-unpackbootimg>

## 关于recovery.img

同样的方法适用于recovery.img

## 工具

[unpackbootimg/packbootimg](http://www.github.com/niejuhu/mytools)
