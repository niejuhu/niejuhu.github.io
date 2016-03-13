---
title: Android Secure Boot
layout: post
category: Android
---

Google has a detailed document of this feature, see [Verifying Boot](https://source.android.com/devices/tech/security/verifiedboot/verified-boot.html).
Here is part of the code.

# Sign boot.img

boot.img is signed after building if `PRODUCT_SUPPORTS_VERITY` is set to true in the makefile.
See [Android Boot Image]

build/core/config.mk:

        BOOT_SIGNER := $(HOST_OUT_EXECUTABLES)/boot_signer

        build/core/Makefile

        ifeq ($(TARGET_BOOTIMAGE_USE_EXT2),true)
        ...
        else ifeq (true,$(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY)) # TARGET_BOOTIMAGE_USE_EXT2 != true
        $(INSTALLED_BOOTIMAGE_TARGET): $(MKBOOTIMG) $(INTERNAL_BOOTIMAGE_FILES) $(BOOT_SIGNER)
            $(call pretty,"Target boot image: $@")
            $(hide) $(MKBOOTIMG) $(INTERNAL_BOOTIMAGE_ARGS) $(BOARD_MKBOOTIMG_ARGS) --output $@
            $(BOOT_SIGNER) /boot $@ $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_VERITY_SIGNING_KEY) $@
            $(hide) $(call assert-max-image-size,$@,$(BOARD_BOOTIMAGE_PARTITION_SIZE))
        ...
        endif

boot_signer is defined in system/extra/verity/Android.mk:

        include $(CLEAR_VARS)
        LOCAL_SRC_FILES := boot_signer
        LOCAL_MODULE := boot_signer
        LOCAL_MODULE_CLASS := EXECUTABLES
        LOCAL_IS_HOST_MODULE := true
        LOCAL_MODULE_TAGS := optional
        LOCAL_REQUIRED_MODULES := BootSignature
        include $(BUILD_PREBUILT)

boot_signer is a script, it calls BootSignature.jar which defined in the same makefile.

system/extra/verity/boot_signer:

        #! /bin/sh

        # Start-up script for BootSigner

        BOOTSIGNER_HOME=`dirname "$0"`
        BOOTSIGNER_HOME=`dirname "$BOOTSIGNER_HOME"`

        java -Xmx512M -jar "$BOOTSIGNER_HOME"/framework/BootSignature.jar "$@"

system/extra/verity/Android.mk:

        include $(CLEAR_VARS)
        LOCAL_SRC_FILES := BootSignature.java VeritySigner.java Utils.java
        LOCAL_MODULE := BootSignature
        LOCAL_JAR_MANIFEST := BootSignature.mf
        LOCAL_MODULE_TAGS := optional
        LOCAL_STATIC_JAVA_LIBRARIES := bouncycastle-host
        include $(BUILD_HOST_JAVA_LIBRARY)

system/extra/verity/BootSignature.java:

        /**
         *    AndroidVerifiedBootSignature DEFINITIONS ::=
         *    BEGIN
         *        FormatVersion ::= INTEGER
         *        AlgorithmIdentifier ::=  SEQUENCE {
         *            algorithm OBJECT IDENTIFIER,
         *            parameters ANY DEFINED BY algorithm OPTIONAL
         *        }
         *        AuthenticatedAttributes ::= SEQUENCE {
         *            target CHARACTER STRING,
         *            length INTEGER
         *        }
         *        Signature ::= OCTET STRING
         *     END
         */

# Verify boot.img

In the boot process of device aboot verify the boot.img.

bootable/bootloader/lk/AndroidBoot.mk:

        ifeq ($(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY),true)
          VERIFIED_BOOT := VERIFIED_BOOT=1
        else
          VERIFIED_BOOT := VERIFEID_BOOT=0
        endif

        $(TARGET_EMMC_BOOTLOADER): emmc_appsbootldr_clean | $(EMMC_BOOTLOADER_OUT) $(INSTALLED_KEYSTOREIMAGE_TARGET)
            $(MAKE) -C bootable/bootloader/lk TOOLCHAIN_PREFIX=$(CROSS_COMPILE) BOOTLOADER_OUT=../../../$(EMMC_BOOTLOADER_OUT) $(BOOTLOADER_PLATFORM) EMMC_BOOT=1 $(SIGNED_KERNEL) $(VERIFIED_BOOT) $(DEVICE_STATUS)

bootable/bootloader/lk/app/aboot/aboot.c:

        aboot_init -> boot_linux_from_mmc -> verify_signed_bootimg:

        #if VERIFIED_BOOT
            if(boot_into_recovery)
            {
                ret = boot_verify_image((unsigned char *)bootimg_addr,
                        bootimg_size, "recovery");
            }
            else
            {
                ret = boot_verify_image((unsigned char *)bootimg_addr,
                        bootimg_size, "boot");
            }
            boot_verify_print_state();
        #else
            ret = image_verify((unsigned char *)bootimg_addr,
                               (unsigned char *)(bootimg_addr + bootimg_size),
                               bootimg_size,
                               auth_algo);
        #endif

bootable/bootloader/lk/platform/msm_shared/boot_verifier.c:

        boot_verify_image: verify

# Sign system.img

system.img is signed if `PRODUCT_SUPPORTS_VERITY` is set to true.
See [Android System Image]

build/core/Makefile:

        $(if $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY),$(hide) echo "verity=$(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY)" >> $(1))
        $(if $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY),$(hide) echo "verity_key=$(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_VERITY_SIGNING_KEY)" >> $(1))
        $(if $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY),$(hide) echo "verity_signer_cmd=$(VERITY_SIGNER)" >> $(1))

build/tools/releasetools/build_image.py:

        def BuildImage(...):
            is_verity_partition = "verity_block_device" in prop_dict
            verity_supported = prop_dict.get("verity") == "true"

            if verity_supported and is_verity_partition:
                if not MakeVerityEnabledImage(out_file, prop_dict):
                    return False
        def MakeVerityEnabledImage(out_file, prop_dict):
            "build_verity_tree -A %s %s %s" % (FIXED_SALT, sparse_image_path, verity_image_path)
            "system/extras/verity/build_verity_metadata.py %s %s %s %s %s %s %s" %
                          (image_size,
                           verity_metadata_path,
                           root_hash,
                           salt,
                           block_device,
                           signer_path,
                           key)
            "append2simg %s %s" data_image_path, verity_metadata_path
            "append2simg %s %s" data_image_path, verity_image_path

`build_verity_tree` and `build_verity_metadata.py` are both in `system/extra/verity` dir.

# Verify system.img

The init program will trigger the "late-init" property in the boot process:

system/core/init.c:

        ...
        action_for_each_trigger("late-init", action_add_queue_tail);
        ...
        for (;;) {
           execute_one_command();
        }

The "late-init" property will trigger "fs" property:

system/core/rootdir/init.rc:

        on late-init
        trigger early-fs
        trigger fs
        trigger post-fs
        trigger post-fs-data

then "fs" property will trigger "mount_all" action with "fstab.<target>" as paramenter:

/init.target.rc:

        on fs
        mount_all fstab.<target>

fstab.<target> records how to mount Android partitions. To verify system.img the system partition entry in the fstab file should have verify flag:

fstab.<target>:

        /dev/block/mtdblock0 /system ext4 ro,barrier=1 wait,verify

The "mount_all" action is handle by "do_mount_all" in the init program:

system/core/init/keywords.h:

        KEYWORD(mount_all, COMMAND, 1, do_mount_all)

system/core/init/builtins.c:

        int do_mount_all(int nargs, char **args)
        {
            ...
            fstab = fs_mgr_read_fstab(args[1]);
            child_ret = fs_mgr_mount_all(fstab);
            fs_mgr_free_fstab(fstab);
            ...
        }

system/core/fs_mgr/fs_mgr.c

        int fs_mgr_mount_all(struct fstab *fstab)
        {
            ...
            if ((fstab->recs[i].fs_mgr_flags & MF_VERIFY) &&
                    !device_is_debuggable()) {
                wait_for_file("/dev/device-mapper", WAIT_TIMEOUT);
                if (fs_mgr_setup_verity(&fstab->recs[i]) < 0) {
                    ERROR("Could not set up verified partition, skipping!\n");
                    continue;
                }
            }
            ...
        }

The function "fs_mgr_setup_verity" creates dm-verity device with the mount point, verify signature of dm-verity metadata and setup property:

system/core/fs_mgr/fs_mgr_verity.c:

        int fs_mgr_setup_verity(...)
        {
            fd = open("/dev/device-mapper", O_RDWR);

            /* create dm-verity device with mount point */
            create_verity_device(io, mount_point, fd);

            /* get the created dm-verity device name which is /dev/block/dm-<dev-num> */
            get_verity_device_name(io, mount_point, fd, &verity_blk_name);

            /* verify signature of dm-verity metadata */
            read_verity_metadata();
            verity_table();

            /* start the device */
            load_verity_table();
            resume_verity_table();

            /* update the block device name, which is /dev/block/dm-<dev-num> */
            fstab->blk_device = verity_blk_name;

            /* setup property indicating that the mount_point partition is verified:
             *      partition.<mount_point>.verified=1
             */
            set_verified_property();
        }
