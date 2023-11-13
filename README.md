# How to Compile MOC-Kernel ?
`based on Ubuntu-22.04`
## 1.init repo to download android-common-kernel source code & tools
init repo
```
mkdir android-kernel && cd android-kernel
repo init -u https://android.googlesource.com/kernel/manifest
vim .repo/mainfests/manifest_9937098.xml
```
edit manifest_9937098.xml as below
```
<manifest>
<remote name="aosp" fetch="https://android.googlesource.com/" review="https://android.googlesource.com/"/>
<default upstream="master-kernel-build-2021" dest-branch="master-kernel-build-2021" remote="aosp" sync-j="4"/>
<superproject name="kernel/superproject" remote="aosp" revision="common-android12-5.10-2023-04"/>
<project path="build" name="kernel/build" revision="aa9e9a42c7b058c1d339c7a03f97683b21eb9e35"/>
<project path="hikey-modules" name="kernel/hikey-modules" revision="6f0a2a72f849d8bb8e708587582c20019ef91a3c" upstream="android12-5.10" dest-branch="android12-5.10"/>
<project path="common" name="kernel/common" revision="61344663df420c35b7d70ac37c28880ffea72f51" upstream="android12-5.10-2023-04" dest-branch="android12-5.10-2023-04"/>
<project path="kernel/tests" name="kernel/tests" revision="c2ea6143e8f1efb9a68cca88159210e16cde1bac"/>
<project path="kernel/configs" name="kernel/configs" revision="c10b7ea022edc356d37c092d7ca46bcb860f8a90"/>
<project path="common-modules/virtual-device" name="kernel/common-modules/virtual-device" revision="c6a28520439360ffc10ab1d5f39f94b168f9010d" upstream="android12-5.10" dest-branch="android12-5.10"/>
<project path="prebuilts-master/clang/host/linux-x86" name="platform/prebuilts/clang/host/linux-x86" clone-depth="1" revision="6e3223f76384455acde43affde3df0ea9df66c0d"/>
<project path="prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.17-4.8" name="platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.17-4.8" clone-depth="1" revision="4e6f66acf138d40d9a80be24b275abb9c6eed729"/>
<project path="prebuilts/build-tools" name="platform/prebuilts/build-tools" clone-depth="1" revision="cfedc16ec3deb680fca6fe2aff44a1837a97b50d"/>
<project path="prebuilts/kernel-build-tools" name="kernel/prebuilts/build-tools" clone-depth="1" revision="ca5b087f88c0302ff66f59a6f26be663e92baf15"/>
<project path="tools/mkbootimg" name="platform/system/tools/mkbootimg" revision="34bc8bfb493401f7aadbd3b434f9fbfa521e9301"/>
</manifest>
```
sync repo
```
repo init -m manifest_9937098.xml
repo sync -j$(nproc)
```
wait for about 40mins (mine is 2m/s)

## 2.since you've got clang12.0.5 and build scripts and mkbootimage, you can choose to compile from either google's source code or MOC-Kernel's source code
`if you'd like to compile from MOC-Kernel's source code, you can  continue reading this section, if not, you can skip to section 3`
clone MOC-Kernel's source code and rename it to "build"
```
git clone https://github.com/moculll/MOC-Kernel.git
mv common common_bak && mv MOC-Kernel common
```

## 3.download released boot.img from google to get ramdisk
```
curl -O https://dl.google.com/android/gki/gki-certified-boot-android12-5.10-2023-04_r1.zip
unzip gki-certified-boot-android12-5.10-2023-04_r1.zip
python3 tools/mkbootimg/unpack_bootimg.py --boot_img boot-5.10.img --out default_boot_unpack
```
then you can get boot_signature、kernel、ramdisk in the default_boot_unpack folder
`tips: if you have magisk, and you're using gki kernel too, you can replace ramdisk by your boot's unpacked one, then magisk can stay as before since you've flashed the boot image`

## 4.build kernel by scripts you got from google

for the first time you compile the kernel, you need to run this

```
LTO=thin BUILD_BOOT_IMG=1 SKIP_VENDOR_BOOT=1 BOOT_IMAGE_HEADER_VERSION=4 KERNEL_BINARY=Image GKI_RAMDISK_PREBUILT_BINARY=default_boot_unpack/ramdisk BUILD_CONFIG=common/build.config.gki.aarch64 build/build.sh
```

for the second time, in order to save time, you can run ccache clang and disable mrproper

```
LTO=thin BUILD_BOOT_IMG=1 SKIP_VENDOR_BOOT=1 BOOT_IMAGE_HEADER_VERSION=4 KERNEL_BINARY=Image GKI_RAMDISK_PREBUILT_BINARY=default_boot_unpack/ramdisk CC="ccache clang" SKIP_MRPROPER=1 BUILD_CONFIG=common/build.config.gki.aarch64 build/build.sh
```

## 5.flash boot.img into your device
since a few devices use a/b partition, you need to use fastboot/adb tool to get which partition you're using
##### fastboot tool
```
fastboot getvar all
```
output: (bootloader) current-slot: a(or b)

##### adb tool
```
adb shell /bin/getprop ro.boot.slot_suffix
```
output: _a(or _b)

#### now you've known which slot you're using, you can flash boot.img as blow
```
fastboot set_active a
```

`you can boot from ram rather than flash in case for kernel can't work`
```
fastboot boot boot.img
```
`if it works, that means you can flash it into your boot partition right away`
```
fastboot flash boot boot.img
```

### Enjoy your GKI kernel !