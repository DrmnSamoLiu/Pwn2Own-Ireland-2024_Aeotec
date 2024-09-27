# Foreword
My friend and I was planning to pwn this Aeotec smart home hub for this year's P2O, but sadly we realized we won't have much time to work on it before the deadline. <br><br>
We decided to publish the information we gathered as recon work in hopes that it could help the user community or other fellow security researchers. If you used any of the information provided here in your work, please remember to credit this repository.<br><br>
Feel free to open discussions in the `Discussions` tab, although we cannot guarantee we'll join or reply to the threads.

# High Resolusion Image
![Front](https://github.com/user-attachments/assets/fdef6198-f807-4e90-93b7-ed8cbdbab0cd)
![Back](https://github.com/user-attachments/assets/acc2cf27-c7c3-4bd2-ad6b-0432f9b7bf61)



# Hardware
- UART pins are clearly marked on the back side of the PCB.
- No interactive shell available through UART, only some kernel boot logs.

<br><br>

**Processor**: NXP MCIMX6Y1DVM05AB <br><br>
**Storage**: Samsung KLM4G1FETE-B041, eMMC 5.1, 4GiB. Dimension: 11x10x0.8
```
Number  Start  End     Size    Type     File system  Flags
 1      1536B  537MB   537MB   primary  fat16        lba
 2      537MB  3909MB  3372MB  primary               lvm

```
<br><br>
**Arch**:
```
$ readelf -A busybox
Attribute Section: aeabi
File Attributes
  Tag_CPU_name: "7VE"
  Tag_CPU_arch: v7
  Tag_CPU_arch_profile: Application
  Tag_ARM_ISA_use: Yes
  Tag_THUMB_ISA_use: Thumb-2
  Tag_FP_arch: VFPv3
  Tag_Advanced_SIMD_arch: NEONv1
  Tag_ABI_PCS_wchar_t: 4
  Tag_ABI_FP_rounding: Needed
  Tag_ABI_FP_denormal: Needed
  Tag_ABI_FP_exceptions: Needed
  Tag_ABI_FP_number_model: IEEE 754
  Tag_ABI_align_needed: 8-byte
  Tag_ABI_align_preserved: 8-byte, except leaf SP
  Tag_ABI_enum_size: int
  Tag_ABI_VFP_args: VFP registers
  Tag_CPU_unaligned_access: v6
  Tag_MPextension_use: Allowed
  Tag_DIV_use: Allowed in v7-A with integer division extension
  Tag_Virtualization_use: TrustZone and Virtualization Extensions

```

# Firmware

- OTA update happens through TLS, so can't just easily find out download URL by capturing packets.
- Uboot, Linux kernel, op-tee and initrd are not encrypted in the eMMC.
- User data and rootfs LVM partitions are encrypted with LUKS:
```
  ACTIVE            '/dev/vg_emmc/lv_data' [256.00 MiB] inherit
  ACTIVE            '/dev/vg_emmc/lv_golden.old' [48.00 MiB] inherit
  ACTIVE            '/dev/vg_emmc/lv_factory' [32.00 MiB] inherit
  ACTIVE            '/dev/vg_emmc/lv_root.old' [32.00 MiB] inherit
  ACTIVE            '/dev/vg_emmc/lv_root' [96.00 MiB] inherit
  ACTIVE            '/dev/vg_emmc/lv_golden' [56.00 MiB] inherit

```

```
LUKS header information for /dev/vg_emmc/lv_root

Version:        1
Cipher name:    aes
Cipher mode:    xts-plain64
Hash spec:      sha256
Payload offset: 8192
MK bits:        256
MK digest:      <redacted> 
MK salt:        <redacted> 
 
MK iterations:  9752
UUID:          <redacted>

Key Slot 0: ENABLED
        Iterations:             156038
        Salt:                   65 9c 27 ce a2 8a e5 1c 83 d5 53 df 05 37 5e 93 
                                18 50 98 43 41 9f 25 38 9f 26 1d 3e 22 f2 32 8d 
        Key material offset:    8
        AF stripes:             4000
Key Slot 1: DISABLED
Key Slot 2: DISABLED
Key Slot 3: DISABLED
Key Slot 4: DISABLED
Key Slot 5: DISABLED
Key Slot 6: DISABLED
Key Slot 7: DISABLED

```

# Boot Process

1. Uboot loads `standard.fit` from `lba` partition in the eMMC.
    - `standard.fit` is a Yocto Flattened Image Tree, which contains images for the boot process.
<br><br>
2. Op-tee image is loaded and it will then load Linux kernel.
<br><br>
3. When booted into initrd,  `tee-supplicant` is started. 
    - `/usr/sbin/get_key`  is called during the execution of `init.d/11-mountcrypto`
    - It will get the key from eMMC RPMB space through op-tee.
    - The key will then be used to decrypt LUKS encryoted LVM volumes.

# FIT Image

```
Image contains unit addresses @, this will break signing
FIT description: FIT image for Poky (Yocto Project Reference Distro)/1.0/hubv3
Created:         Tue Apr  5 19:00:00 2011
 Image 0 (kernel@1)
  Description:  Linux Kernel 1
  Created:      Tue Apr  5 19:00:00 2011
  Type:         Kernel Image
  Compression:  uncompressed
  Data Size:    5796936 Bytes = 5661.07 KiB = 5.53 MiB
  Architecture: ARM
  OS:           Linux
  Load Address: 0x80008000
  Entry Point:  0x80008000
  Hash algo:    sha1
  Hash value:   eb90455968fcbe1e64caf30f34a3fd81cd526350
 Image 1 (fdt@1)
  Description:  Device tree blob 1
  Created:      Tue Apr  5 19:00:00 2011
  Type:         Flat Device Tree
  Compression:  uncompressed
  Data Size:    27909 Bytes = 27.25 KiB = 0.03 MiB
  Architecture: ARM
  Hash algo:    sha1
  Hash value:   db88ea671970cc77bf48d3a65cca896948fd1336
 Image 2 (ramdisk@1)
  Description:  Initramfs 1
  Created:      Tue Apr  5 19:00:00 2011
  Type:         RAMDisk Image
  Compression:  uncompressed
  Data Size:    17013657 Bytes = 16614.90 KiB = 16.23 MiB
  Architecture: ARM
  OS:           Linux
  Load Address: unavailable
  Entry Point:  unavailable
  Hash algo:    sha1
  Hash value:   df0541528ac1a7fec28859686a7e08ca8e2323f8
 Image 3 (optee@1)
  Description:  OP-TEE OS 1
  Created:      Tue Apr  5 19:00:00 2011
  Type:         Kernel Image
  Compression:  uncompressed
  Data Size:    1182260 Bytes = 1154.55 KiB = 1.13 MiB
  Architecture: ARM
  OS:           Linux
  Load Address: 0x8effffe4
  Entry Point:  0x8f000000
  Hash algo:    sha1
  Hash value:   5a9a7cd9abffbbf0d7f8d42712630b39b2b697a8
 Default Configuration: 'conf@1'
 Configuration 0 (conf@1)
  Description:  Boot and mount LVM/LUKS
  Kernel:       optee@1
  Init Ramdisk: ramdisk@1
  FDT:          fdt@1
  Loadables:    kernel@1
  Hash algo:    sha1
  Hash value:   unavailable
  Sign algo:    sha1,rsa2048:prod
  Sign value:   4f081297d67c23550531b59d6b2479f615af0a5c8d02bc8ba6a4137b0d83496d95c66faf4802d2f8483530af738d8c834948b483e9b8e4baefa06016048545ad9719684033013cbd8a6f4864174fff2e08b0a14d1c57379264076437dcd239f25713d677d9acdb8ec184c03a0d5573652e556fbc91a0a19d7097c23a7e79e0710265b2dcca9b20c0b26716e9b6e93292cb10911ec8e3c8fb73f5bd39bc213ccd0a234ebfc609488a14103dc54b64371671675e02cfb55026660e2bac2da3a80b94f6287660c66cc84f1b22fa35d1096b20fff26407c2809dc16c73c4feacb5ed7974bd5e008192bda0e382b2603a1271fb7dcd1f277f8df5527293d3420c6bdd
  Timestamp:    Mon Jun 17 22:47:46 2024
```

# Side Note

- It MAY be possible to boot into custom Linux kernel in a USB drive. (I haven't tested it personally.)<br><br>
    - The Uboot boot command `boot_primary` checks for `"$boot_mode" = "1"`, which seems to be set if USB-OTG power is enabled. (Or injecting voltage into `OTG1_VBUS` pad on the back side of the PCB?)
    - If true, it looks for `standard.fit` in USB volume and load it to `$loadaddr`. If this succeeded too, it then proceed to execute `bootm $loadaddr` **\*without checking any security parameter such as signatures\***.
