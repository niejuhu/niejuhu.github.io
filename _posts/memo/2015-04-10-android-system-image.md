# How android build system image

We use command "make systemimage" to build system.img. This post list code which can show how it works.

The target `systemimage` is defined in `android-src/build/core/Makefile`:

        .PHONY systemimage
        systemimage: $(INSTALLED_SYSTEMIMAGE)

`INSTALLED_SYSTEMIMAGE` and its make rule are defined in the same file:

        INSTALLED_SYSTEMIMAGE := $(PRODUCT_OUT)/system.img

        $(INSTALLED_SYSTEMIMAGE): $(BUILT_SYSTEMIMAGE) $(RECOVERY_FROM_BOOT_PATCH) | $(ACP)
            @echo "Install system fs image: $@"
            $(copy-file-to-target)
            $(hide) $(call assert-max-image-size,$@ $(RECOVERY_FROM_BOOT_PATCH),$(BOARD_SYSTEMIMAGE_PARTITION_SIZE))

This rule just copies the intermediate system.img `$BUILT_SYSTEMIMAGE` to the final target `$(PRODUCT_OUT)/system.img`
and asserts its size. So the `BUILT_SYSTEMIMAGE` is the key point. Here we ignore the `RECOVERY_FROM_BOOT_PATCH`.


Definition and rule of `BUILT_SYSTEMIMAGE` are as follows:

        systemimage_intermediates := \
            $(call intermediates-dir-for,PACKAGING,systemimage)
        BUILT_SYSTEMIMAGE := $(systemimage_intermediates)/system.img

        $(BUILT_SYSTEMIMAGE): $(FULL_SYSTEMIMAGE_DEPS) $(INSTALLED_FILES_FILE)
            $(call build-systemimage-target,$@)

* `FULL_SYSTEMIMAGE_DEPS`

        FULL_SYSTEMIMAGE_DEPS := $(INTERNAL_SYSTEMIMAGE_FILES) $(INTERNAL_USERIMAGES_DEPS)

        INTERNAL_SYSTEMIMAGE_FILES := $(filter $(TARGET_OUT)/%, \
            $(ALL_PREBUILT) \
            $(ALL_COPIED_HEADERS) \
            $(ALL_GENERATED_SOURCES) \
            $(ALL_DEFAULT_INSTALLED_MODULES) \
            $(PDK_FUSION_SYSIMG_FILES) \
            $(RECOVERY_RESOURCE_ZIP))

        ifeq ($(INTERNAL_USERIMAGES_USE_EXT),true)
        INTERNAL_USERIMAGES_DEPS := $(SIMG2IMG)
        INTERNAL_USERIMAGES_DEPS += $(MKEXTUSERIMG) $(MAKE_EXT4FS) $(E2FSCK)
        ifeq ($(TARGET_USERIMAGES_USE_F2FS),true)
        INTERNAL_USERIMAGES_DEPS += $(MKF2FSUSERIMG) $(MAKE_F2FS)
        endif
        else
        INTERNAL_USERIMAGES_DEPS := $(MKYAFFS2)
        endif

        ifeq (true,$(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY))
        INTERNAL_USERIMAGES_DEPS += $(BUILD_VERITY_TREE) $(APPEND2SIMG) $(VERITY_SIGNER)
        endif
        SELINUX_FC := $(TARGET_ROOT_OUT)/file_contexts
        INTERNAL_USERIMAGES_DEPS += $(SELINUX_FC)

* `INSTALLED_FILES_FILE`

        INSTALLED_FILES_FILE := $(PRODUCT_OUT)/installed-files.txt
        $(INSTALLED_FILES_FILE): $(FULL_SYSTEMIMAGE_DEPS)
            @echo Installed file list: $@
            @mkdir -p $(dir $@)
            @rm -f $@
            $(hide) build/tools/fileslist.py $(TARGET_OUT) > $@

* `build-systemimage-target`

        # $(1): output file
        define build-systemimage-target
        @echo "Target system fs image: $(1)"
        $(call create-system-vendor-symlink)
        @mkdir -p $(dir $(1)) $(systemimage_intermediates) && rm -rf $(systemimage_intermediates)/system_image_info.txt
        $(call generate-userimage-prop-dictionary, $(systemimage_intermediates)/system_image_info.txt, \
            skip_fsck=true)
        $(hide) PATH=$(foreach p,$(INTERNAL_USERIMAGES_BINARY_PATHS),$(p):)$$PATH \
            ./build/tools/releasetools/build_image.py \
            $(TARGET_OUT) $(systemimage_intermediates)/system_image_info.txt $(1) \
            || ( echo "Out of space? the tree size of $(TARGET_OUT) is (MB): " 1>&2 ;\
                du -sm $(TARGET_OUT) 1>&2;\
                echo "The max is $$(( $(BOARD_SYSTEMIMAGE_PARTITION_SIZE) / 1048576 )) MB." 1>&2 ;\
                mkdir -p $(DIST_DIR); cp $(INSTALLED_FILES_FILE) $(DIST_DIR)/installed-files-rescued.txt; \
                exit 1 )
        endef

        # $(1): the path of the output dictionary file
        # $(2): additional "key=value" pairs to append to the dictionary file.
        define generate-userimage-prop-dictionary
        $(if $(INTERNAL_USERIMAGES_EXT_VARIANT),$(hide) echo "fs_type=$(INTERNAL_USERIMAGES_EXT_VARIANT)" >> $(1))
        $(if $(BOARD_SYSTEMIMAGE_PARTITION_SIZE),$(hide) echo "system_size=$(BOARD_SYSTEMIMAGE_PARTITION_SIZE)" >> $(1))
        $(if $(BOARD_USERDATAIMAGE_FILE_SYSTEM_TYPE),$(hide) echo "userdata_fs_type=$(BOARD_USERDATAIMAGE_FILE_SYSTEM_TYPE)" >> $(1))
        $(if $(BOARD_USERDATAIMAGE_PARTITION_SIZE),$(hide) echo "userdata_size=$(BOARD_USERDATAIMAGE_PARTITION_SIZE)" >> $(1))
        $(if $(BOARD_CACHEIMAGE_FILE_SYSTEM_TYPE),$(hide) echo "cache_fs_type=$(BOARD_CACHEIMAGE_FILE_SYSTEM_TYPE)" >> $(1))
        $(if $(BOARD_CACHEIMAGE_PARTITION_SIZE),$(hide) echo "cache_size=$(BOARD_CACHEIMAGE_PARTITION_SIZE)" >> $(1))
        $(if $(BOARD_VENDORIMAGE_FILE_SYSTEM_TYPE),$(hide) echo "vendor_fs_type=$(BOARD_VENDORIMAGE_FILE_SYSTEM_TYPE)" >> $(1))
        $(if $(BOARD_VENDORIMAGE_PARTITION_SIZE),$(hide) echo "vendor_size=$(BOARD_VENDORIMAGE_PARTITION_SIZE)" >> $(1))
        $(if $(BOARD_OEMIMAGE_PARTITION_SIZE),$(hide) echo "oem_size=$(BOARD_OEMIMAGE_PARTITION_SIZE)" >> $(1))
        $(if $(INTERNAL_USERIMAGES_SPARSE_EXT_FLAG),$(hide) echo "extfs_sparse_flag=$(INTERNAL_USERIMAGES_SPARSE_EXT_FLAG)" >> $(1))
        $(if $(mkyaffs2_extra_flags),$(hide) echo "mkyaffs2_extra_flags=$(mkyaffs2_extra_flags)" >> $(1))
        $(hide) echo "selinux_fc=$(SELINUX_FC)" >> $(1)
        $(if $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY),$(hide) echo "verity=$(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY)" >> $(1))
        $(if $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY),$(hide) echo "verity_key=$(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_VERITY_SIGNING_KEY)" >> $(1))
        $(if $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SUPPORTS_VERITY),$(hide) echo "verity_signer_cmd=$(VERITY_SIGNER)" >> $(1))
        $(if $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SYSTEM_VERITY_PARTITION),$(hide) echo "system_verity_block_device=$(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SYSTEM_VERITY_PARTITION)" >> $(1))
        $(if $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_VENDOR_VERITY_PARTITION),$(hide) echo "vendor_verity_block_device=$(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_VENDOR_VERITY_PARTITION)" >> $(1))
        $(if $(2),$(hide) $(foreach kv,$(2),echo "$(kv)" >> $(1);))
        endef

    `build-systemimage-target` calls `generate-userimage-prop-dictionary` to produce the `system_image_info.txt` and then
    calls `build_image.py` to produce the intermediate system.img.
