---
title: Android Verified Boot
layout: post
category: Android
---

Google has a detailed documention of this feature, see [Verifying Boot](https://source.android.com/devices/tech/security/verifiedboot/verified-boot.html).
It should be read carefully. Here is part of the code.

# Products configurations

To support verified boot, products should include some configurations.

1. Include build/target/product/verity.mk as part of device configureations. The file defines some
variables:

        PRODUCT_SUPPORTS_BOOT_SIGNER := true
        PRODUCT_SUPPORTS_VERITY := true
        PRODUCT_VERITY_SIGNING_KEY := build/target/product/security/verity
        PRODUCT_PACKAGES += \
            verity_key

2. Define `PRODUCT_SYSTEM_VERITY_PARTITION` as the system device name e.g.
`/dev/block/platform/msm_sdcc.1by-name/system`

3. Add verify flag to system mount options in fstab files like "/fstab.qcom":

        #<src>                                        <mnt_point> <type> <mnt_flags and options> <fs_mgr_flags>
        /dev/block/platform/msm_sdcc.1/by-name/system /system     ext4   ro,barrier=1            wait,verify

4. Make bootloader verify boot.img and other customizations.

    I do not cover these contents in this post.

# Sign boot.img

boot.img is signed if `PRODUCT_SUPPORTS_BOOT_SIGNER` is set to true.

    # build/core/config.mk:

    BOOT_SIGNER := $(HOST_OUT_EXECUTABLES)/boot_signer


    # build/core/Makefile

    else ifeq (true,$(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_BOOT_SIGNER)) # TARGET_BOOTIMAGE_USE_EXT2 != true
    $(INSTALLED_BOOTIMAGE_TARGET): $(MKBOOTIMG) $(INTERNAL_BOOTIMAGE_FILES) $(BOOT_SIGNER)
    $(call pretty,"Target boot image: $@")
    $(hide) $(MKBOOTIMG) $(INTERNAL_BOOTIMAGE_ARGS) $(BOARD_MKBOOTIMG_ARGS) --output $@
    $(BOOT_SIGNER) /boot $@ $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_VERITY_SIGNING_KEY).pk8 $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_VERITY_SIGNING_KEY).x509.pem $@
    $(hide) $(call assert-max-image-size,$@,$(BOARD_BOOTIMAGE_PARTITION_SIZE))
    ...
    endif
    
`boot_signer` is defined in system/extra/verity/Android.mk:

    include $(CLEAR_VARS)
    LOCAL_SRC_FILES := boot_signer
    LOCAL_MODULE := boot_signer
    LOCAL_MODULE_CLASS := EXECUTABLES
    LOCAL_IS_HOST_MODULE := true
    LOCAL_MODULE_TAGS := optional
    LOCAL_REQUIRED_MODULES := BootSignature
    include $(BUILD_PREBUILT)

It is a script which calls BootSignature.jar.

    #! /bin/sh
    # Start-up script for BootSigner

    BOOTSIGNER_HOME=`dirname "$0"`
    BOOTSIGNER_HOME=`dirname "$BOOTSIGNER_HOME"`

    java -Xmx512M -jar "$BOOTSIGNER_HOME"/framework/BootSignature.jar "$@"

BootSignature.jar is defined in system/extra/verity/Android.mk:

    include $(CLEAR_VARS)
    LOCAL_SRC_FILES := BootSignature.java VeritySigner.java Utils.java
    LOCAL_MODULE := BootSignature
    LOCAL_JAR_MANIFEST := BootSignature.mf
    LOCAL_MODULE_TAGS := optional
    LOCAL_STATIC_JAVA_LIBRARIES := bouncycastle-host
    include $(BUILD_HOST_JAVA_LIBRARY)

system/extra/verity/BootSignature.java does the real work. It caculates signature of orignal
boot.img and some of its information, that is,

    signature = sign(orig_boot.img + "/boot" + len(orig_boot.img))

The signature is then appended to the orignal boot.img. Here is the asn1 structure of the
signature.

    /**
    *    AndroidVerifiedBootSignature DEFINITIONS ::=
    *    BEGIN
    *        formatVersion ::= INTEGER
    *        certificate ::= Certificate
    *        algorithmIdentifier ::= SEQUENCE {
    *            algorithm OBJECT IDENTIFIER,
    *            parameters ANY DEFINED BY algorithm OPTIONAL
    *        }
    *        authenticatedAttributes ::= SEQUENCE {
    *            target CHARACTER STRING,
    *            length INTEGER
    *        }
    *        signature ::= OCTET STRING
    *     END
    */

The structure of the signed boot.img is as followed.

![bootimage](/images/bootimg.png)

# Build dm-verity enabled system.img

dm-verity is enabled for system.img if device configuration defines `PRODUCT_SUPPORTS_VERITY=true`.
Of cause some other verity relative configurations should also included.

system.img is built as followed.

    # build/core/Makefile

    .PHONY: systemimage

    systemimage: $(INSTALLED_SYSTEMIMAGE)

    $(INSTALLED_SYSTEMIMAGE): $(BUILT_SYSTEMIMAGE) $(RECOVERY_FROM_BOOT_PATCH) | $(ACP)
    @echo "Install system fs image: $@"
    $(copy-file-to-target)
    $(hide) $(call assert-max-image-size,$@ $(RECOVERY_FROM_BOOT_PATCH),$(BOARD_SYSTEMIMAGE_PARTITION_SIZE))

    $(BUILT_SYSTEMIMAGE): $(FULL_SYSTEMIMAGE_DEPS) $(INSTALLED_FILES_FILE)
        $(call build-systemimage-target,$@)

    define build-systemimage-target
      @echo "Target system fs image: $(1)"
      $(call create-system-vendor-symlink)
      @mkdir -p $(dir $(1)) $(systemimage_intermediates) && rm -rf $(systemimage_intermediates)/system_image_info.txt
      $(call generate-userimage-prop-dictionary, $(systemimage_intermediates)/system_image_info.txt, \
          skip_fsck=true)
      $(hide) PATH=$(foreach p,$(INTERNAL_USERIMAGES_BINARY_PATHS),$(p):)$$PATH \
          ./build/tools/releasetools/build_image.py \
          $(TARGET_OUT) $(systemimage_intermediates)/system_image_info.txt $(1) $(TARGET_OUT) \
          || ( echo "Out of space? the tree size of $(TARGET_OUT) is (MB): " 1>&2 ;\
               du -sm $(TARGET_OUT) 1>&2;\
               if [ "$(INTERNAL_USERIMAGES_EXT_VARIANT)" == "ext4" ]; then \
                   maxsize=$(BOARD_SYSTEMIMAGE_PARTITION_SIZE); \
                   if [ "$(BOARD_HAS_EXT4_RESERVED_BLOCKS)" == "true" ]; then \
                       maxsize=$$((maxsize - 4096 * 4096)); \
                   fi; \
                   echo "The max is $$(( maxsize / 1048576 )) MB." 1>&2 ;\
               else \
                   echo "The max is $$(( $(BOARD_SYSTEMIMAGE_PARTITION_SIZE) / 1048576 )) MB." 1>&2 ;\
               fi; \
               mkdir -p $(DIST_DIR); cp $(INSTALLED_FILES_FILE) $(DIST_DIR)/installed-files-rescued.txt; \
               exit 1 )
    endef

It can be simplified as two commands:

    define build-systemimage-target
        $(call generate-userimage-prop-dictionary, $(systemimage_intermediates)/system_image_info.txt, \
              skip_fsck=true)
        ./build/tools/releasetools/build_image.py \
              $(TARGET_OUT) $(systemimage_intermediates)/system_image_info.txt system.img $(TARGET_OUT)

`build_image.py` build system image from `$(TARGET_OUT)` and write the output to $(1).
`systemimage_info.txt` controls how to build the image, including whether supports dm-verity:

    $(if $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY),$(hide) echo "verity=$(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY)" >> $(1))
    $(if $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY),$(hide) echo "verity_key=$(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_VERITY_SIGNING_KEY)" >> $(1))
    $(if $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY),$(hide) echo "verity_signer_cmd=$(VERITY_SIGNER)" >> $(1))
    $(if $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SYSTEM_VERITY_PARTITION),$(hide) echo "system_verity_block_device=$(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SYSTEM_VERITY_PARTITION)" >> $(1))

We mainly focus on how `build_image.py` build dm-verity relatives for system.img. Here are the code snippets.

    def BuildImage(in_dir, prop_dict, out_file, target_out=None):
        """Build an image to out_file from in_dir with property prop_dict.

        Args:
          in_dir: path of input directory.
          prop_dict: property dictionary.
          out_file: path of the output image file.
          target_out: path of the product out directory to read device specific FS config files.

        Returns:
          True iff the image is built successfully.
        """

        # Check whether verity is supported in this image.
        is_verity_partition = "verity_block_device" in prop_dict
        verity_supported = prop_dict.get("verity") == "true"

        # If verity is supported, we need some space to store dm-verity tree and metadata. So the
        # partition size should be adjusted. The max partition size is:
        #   partition_size = partition_size - sizeof(dm-verity tree) - sizeof(dm-verity metadata)
        if verity_supported and is_verity_partition and fs_spans_partition:
            partition_size = int(prop_dict.get("partition_size"))

            adjusted_size = AdjustPartitionSizeForVerity(partition_size)
            if not adjusted_size:
                return False
            prop_dict["partition_size"] = str(adjusted_size)
            prop_dict["original_partition_size"] = str(partition_size)

        if fs_type.startswith("ext"):
            # The command to build ext4 image
            build_command = ["mkuserimg.sh"]

            # add args and options
            build_command.extend([in_dir, out_file, fs_type,
                                prop_dict["mount_point"]])
            ...

        # Run the command to build system.img
        (_, exit_code) = RunCommand(build_command)

        # build dm-verity tree and metadata and add them to the end of system.img
        if verity_supported and is_verity_partition:
             MakeVerityEnabledImage(out_file, prop_dict)

    def MakeVerityEnabledImage(out_file, prop_dict):
        # get partial image paths
        verity_image_path = os.path.join(tempdir_name, "verity.img")
        verity_metadata_path = os.path.join(tempdir_name, "verity_metadata.img")
  
        # build the verity tree and get the root hash and salt
        BuildVerityTree(out_file, verity_image_path, prop_dict):
  
        # build the metadata blocks
        BuildVerityMetadata(image_size, verity_metadata_path, root_hash, salt,
                               block_dev, signer_path, signer_key)
  
        # build the full verified image
        BuildVerifiedImage(out_file,
                              verity_image_path,
                              verity_metadata_path)

    def BuildVerityMetadata(image_size, verity_metadata_path, root_hash, salt,
                          block_device, signer_path, key):
        # Just run the following command:
        cmd_template = (
            "system/extras/verity/build_verity_metadata.py %s %s %s %s %s %s %s")
        cmd = cmd_template % (image_size, verity_metadata_path, root_hash, salt,
                              block_device, signer_path, key)
        print cmd
        status, output = commands.getstatusoutput(cmd)

    def BuildVerityTree(sparse_image_path, verity_image_path, prop_dict):
        # Just run the following command:
        cmd = "build_verity_tree -A %s %s %s" % (
            FIXED_SALT, sparse_image_path, verity_image_path)
        print cmd
        status, output = commands.getstatusoutput(cmd)

    def BuildVerifiedImage(data_image_path, verity_image_path,
                         verity_metadata_path):
        # Run the flollowing commands:
        # append2simg data_image_path verity_metadata_path
        # append2simg data_image_path verity_image_path

It also can be simplified as:

    partition_size -= sizeof(dm-verity tree) + sizeof(dm-verity metadata)
    make_ext4fs -l partition_size -a /system system.img $TARGET_OUT $TARGET_OUT <other options>
    root_hash, salt = `build_verity_tree -A FIXED_SALT system.img verity.img`
    build_verity_metadata.py sizeof(system.img) verity_metadata.img root_hash salt <system device name> system/extras/verity/verity_signer <verity_key>
    append2simg system.img verity_metadata.img
    append2simg data_image_path verity.img

Source code for the commands:

cmds                     | source code
-------------------------|------------
mkuserimg.sh             | system/extras/ext4_utils/mkuserimg.sh
make_ext4fs              | system/extras/ext4_utils/make_ext4fs_main.c
build_verity_tree        | system/extras/verity/build_verity_tree.cpp
build_verity_metadata.py | system/extras/verity/build_verity_metadata.py
append2simg              | system/core/libsparse/append2simg.c

Here is the structure of dm-verity enabled system.img.

![systemimage](/images/systemimg.png)

# Verify boot.img

It is bootloader's job to verify boot.img. We do not touch it here.

# Verify system.img

When android boots, the init process using the verity public key (that is `/verity_key`) to verify
signature of dm-verity metadata. Then mount system as dm-verity device.

The init program first triggers "late-init" action in the boot process:

    // system/core/init.c:

    ...
    action_for_each_trigger("late-init", action_add_queue_tail);
    ...
    for (;;) {
       execute_one_command();
    }

The "late-init" commands will trigger "fs" action:

    # system/core/rootdir/init.rc:

    on late-init
    trigger early-fs
    trigger fs
    trigger post-fs
    trigger post-fs-data

then "fs" run `mount_all` command with fstab file (e.g. fstab.qcom) as paramenter:

    # /init.target.rc:

    on fs
    mount_all fstab.qcom

fstab file records how to mount Android partitions. To verify system.img the system partition
entry in the fstab file should have verify flag as descripted before.

The `mount_all` command is handle by `do_mount_all` in the init program:

    // system/core/init/keywords.h:

    KEYWORD(mount_all, COMMAND, 1, do_mount_all)

    // system/core/init/builtins.c:

    int do_mount_all(int nargs, char **args)
    {
        ...
        fstab = fs_mgr_read_fstab(args[1]);
        child_ret = fs_mgr_mount_all(fstab);
        fs_mgr_free_fstab(fstab);
        ...
    }

    // system/core/fs_mgr/fs_mgr.c

    int fs_mgr_mount_all(struct fstab *fstab)
    {
         ...
         if ((fstab->recs[i].fs_mgr_flags & MF_VERIFY) && device_is_secure()) {
              int rc = fs_mgr_setup_verity(&fstab->recs[i]);
              if (device_is_debuggable() && rc == FS_MGR_SETUP_VERITY_DISABLED) {
                  INFO("Verity disabled");
              } else if (rc != FS_MGR_SETUP_VERITY_SUCCESS) {
                  ERROR("Could not set up verified partition, skipping!\n");
                  continue;
              }
         }
         ...
    }

The function `fs_mgr_setup_verity` verifies signature of dm-verity metadata and creates dm-verity
device with the mount point:

    // system/core/fs_mgr/fs_mgr_verity.c:

    int fs_mgr_setup_verity(...)
    {
        /* Check if dm-verity is disabled */
        retval = read_verity_metadata(device_size, fstab->blk_device, &verity_table_signature,
                                    &verity_table);
        if (retval < 0) goto out;

        fd = open("/dev/device-mapper", O_RDWR);

        /* create dm-verity device with mount point */
        create_verity_device(io, mount_point, fd);

        /* get the created dm-verity device name which is /dev/block/dm-<dev-num> */
        get_verity_device_name(io, mount_point, fd, &verity_blk_name);

        /* verify signature of dm-verity metadata */
        read_verity_metadata();
        verity_table();

        /* Get the dm-verity mode */
        if (load_verity_state(fstab, &mode) < 0) {
            mode = VERITY_MODE_EIO;
        }

        /* start the device */
        load_verity_table();
        resume_verity_table();

        /* mark the underlying block device as read-only */
        fs_mgr_set_blk_ro(fstab->blk_device);

        /* update the block device name, which is /dev/block/dm-<dev-num> */
        fstab->blk_device = verity_blk_name;
    }

In the userdebug or eng build, dm-verity can be disable and re-enabled at runtime by adb commands:

    adb disable-verity
    adb enable-verity

`adb disable-verity` command just replaces `MAGIC_NUMBER` in the dm-verity metadata with
`VERITY_METADATA_MAGIC_DISABLE`.

dm-verity has several working mode which control what to do when system crack detected.

    // Verity modes
    enum verity_mode {
        VERITY_MODE_EIO = 0,        // return error
        VERITY_MODE_LOGGING = 1,    // just log
        VERITY_MODE_RESTART = 2,    // restart system
        VERITY_MODE_LAST = VERITY_MODE_RESTART,
        VERITY_MODE_DEFAULT = VERITY_MODE_RESTART
    };

The working mode is determined by kernel cmdline and other logic. See `load_verity_state` function
in the `fs_mgr_verity.c` file.
