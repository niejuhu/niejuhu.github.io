# Issues and solusions for building goldfish kernel on MAC

## elf.h not found

copy an elf.h to /usr/include/elf.h

## linux/types.h not found

copy an types.h to /usr/include/linux/types.h

Here is an short version.

        #include <stdint.h>

        typedef int32_t __s32;
        typedef uint8_t __u8;
        typedef uint16_t __u16;
        typedef uint16_t __u16;
        typedef uint32_t __u32;
        typedef uint64_t __u64;

## `vdso_offset_sigtramp` undeclared

Just grep `vdso_offset_sigtramp` and you will get something like this:

        /arch/arm64/kernel/vdso/vdso-offsets.h:#define vdso_offset_sigtrampt0x04e0

Change `vdso_offset_sigtrampt0x04e0` to `vdso_offset_sigtramp 0x04e0`
