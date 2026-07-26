# Rebuilding the alioth Leica/Miui Camera release

This is the exact build-oriented companion to the main guide. It describes the
workflow used for the Android 16 LineageOS 23.2 alioth build and the later
framework compatibility iteration that made the Xiaomi MiuiCamera preview
functional.

The output is intentionally an unofficial build. Do not present it as an
official LineageOS release and do not redistribute proprietary Xiaomi camera
files unless you are permitted to do so.

## Release model

The published working state consists of these ROM-owned artifacts:

| Artifact | Purpose |
| --- | --- |
| lineage-23.2-20260726-UNOFFICIAL-alioth.zip | Initial LineageOS system installation |
| lineage-23.2-20260726-UNOFFICIAL-alioth-vendor_boot.img | Matching vendor_boot and recovery image |
| lineage-23.2-20260726-UNOFFICIAL-alioth-camera-framework-system.img | Later system image containing the camera-framework compatibility correction |
| SHA256SUMS.txt | Hashes used to verify the three files |

NikGapps Core is required only if Google Play services/Play Store are wanted. It
is third-party software, so it is not republished in this repository's release.
Download the Android 16 arm64 Core package from the official NikGapps source and
verify its SHA-256 against the release manifest before installing it.

## 1. Prepare the host

The build used an Ubuntu 24.04 WSL2 environment. A native Ubuntu environment is
also suitable. Install the Android build dependencies recommended by the
LineageOS 23.2 documentation for your distribution, including:

~~~bash
sudo apt update
sudo apt install bc bison build-essential ccache curl flex g++-multilib gcc-multilib git git-lfs gnupg gperf imagemagick lib32ncurses-dev lib32readline-dev lib32z1-dev libelf-dev liblz4-tool libncurses5-dev libsdl1.2-dev libssl-dev libxml2 libxml2-utils lzop openjdk-21-jdk pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev
~~~

Install Google's repo tool somewhere in PATH:

~~~bash
mkdir -p ~/bin
curl -o ~/bin/repo https://storage.googleapis.com/git-repo-downloads/repo
chmod a+x ~/bin/repo
export PATH="$HOME/bin:$PATH"
~~~

Enable compiler caching before the first build:

~~~bash
export USE_CCACHE=1
export CCACHE_EXEC="$(command -v ccache)"
ccache -M 100G
~~~

Use a case-sensitive filesystem with enough free space. A complete Android tree
and products can consume more than 250 GB.

## 2. Fetch LineageOS and the alioth projects

~~~bash
mkdir -p ~/android/lineage-alioth
cd ~/android/lineage-alioth
repo init -u https://github.com/LineageOS/android.git -b lineage-23.2 --git-lfs
repo sync -c --force-sync --no-clone-bundle --no-tags -j"$(nproc)"
~~~

Add local manifests for your compatible alioth device tree, common Xiaomi
hardware tree, kernel tree, vendor tree, and proprietary extraction project.
The source revisions must be pinned as a set while debugging. Do not combine
the camera provider from one vendor base with a camera APK and firmware from
another base.

Before camera work, build and boot the unmodified tree once:

~~~bash
source build/envsetup.sh
lunch lineage_alioth-ap4a-userdebug
mka bacon -j"$(nproc)"
~~~

This confirms the device tree itself is healthy and prevents camera debugging
from concealing a basic bring-up failure.

## 3. Bring in the proprietary camera payload

From a compatible Xiaomi firmware/vendor source, extract the following as one
coherent set:

1. MiuiCamera.apk.
2. Its package-specific privileged permission/configuration XML files.
3. Camera provider service binary and init service declarations.
4. Camera HAL libraries, dependencies, and tuning/calibration files.
5. The vendor firmware expected by those files.

Put the app in the device vendor/prebuilt area, for example:

~~~text
device/xiaomi/alioth/proprietary/priv-app/MiuiCamera/MiuiCamera.apk
~~~

Create an Android prebuilt module:

~~~make
android_app_import {
    name: "MiuiCamera",
    apk: "proprietary/priv-app/MiuiCamera/MiuiCamera.apk",
    presigned: true,
    privileged: true,
    dex_preopt: {
        enabled: false,
    },
}
~~~

Add it to the product makefile:

~~~make
PRODUCT_PACKAGES += MiuiCamera
~~~

Use the exact package permissions/configuration from the compatible source
firmware. Avoid broad permission grants and do not re-sign a proprietary APK
as a shortcut.

## 4. Add the Android 13 and Android 14 framework compatibility constructors

Edit:

~~~text
frameworks/base/core/java/android/hardware/camera2/params/StreamConfigurationMap.java
~~~

Android 16's implementation has additional HEIC Ultra-HDR fields compared to
Android 14. Add two delegating hidden constructors immediately before the
Android 16 canonical constructor:

- Android 13 form: no JPEG-R inputs; delegates with null JPEG-R and null HEIC
  Ultra-HDR arrays.
- Android 14 form: forwards JPEG-R inputs; delegates with null HEIC Ultra-HDR
  arrays.

Both must preserve the old parameter order and pass true for
enforceImplementationDefined. The complete reference code is in the main
README under "Reference patch". Keep the Android 16 canonical constructor
unchanged.

Review the change:

~~~bash
git -C frameworks/base diff -- core/java/android/hardware/camera2/params/StreamConfigurationMap.java
~~~

The failure mode for a wrong signature is typically a runtime
NoSuchMethodError when the vendor camera provider is used.

## 5. Ensure the vendor camera provider can process requests

Find the camera provider implementation:

~~~bash
rg -n "camera.*provider|configureRpcThreadpool|joinRpcThreadpool|ABinderProcess" device vendor hardware
~~~

The provider must initialize its matching HIDL or AIDL Binder/RPC thread pool
before registering the service, and the calling service thread must join the
pool afterward. Keep the provider's known-good thread count if one is present.

For HIDL this is the configureRpcThreadpool then joinRpcThreadpool lifecycle.
For AIDL it is the ABinderProcess_setThreadPoolMaxThreadCount then
ABinderProcess_joinThreadPool lifecycle. Do not combine both APIs in one
service.

## 6. Produce the initial ROM artifacts

~~~bash
source build/envsetup.sh
lunch lineage_alioth-ap4a-userdebug
mka bacon -j"$(nproc)"
~~~

Collect the initial artifacts from:

~~~text
out/target/product/alioth/
~~~

The release names are:

~~~text
lineage-23.2-20260726-UNOFFICIAL-alioth.zip
lineage-23.2-20260726-UNOFFICIAL-alioth-vendor_boot.img
~~~

Calculate checksums rather than trusting a filename:

~~~bash
sha256sum lineage-23.2-20260726-UNOFFICIAL-alioth.zip lineage-23.2-20260726-UNOFFICIAL-alioth-vendor_boot.img
~~~

## 7. Produce the later camera-framework system image

After identifying the StreamConfigurationMap compatibility problem, rebuild the
Android product with the framework patch present. Package the resulting system
image using the normal device build flow. The verified artifact was named:

~~~text
lineage-23.2-20260726-UNOFFICIAL-alioth-camera-framework-system.img
~~~

The separate system image is required because the initial ZIP predates the
final framework correction. A future clean release should rebuild the full ROM
ZIP after this patch so users need only the main ROM package and vendor_boot
image.

Hash the final image:

~~~bash
sha256sum lineage-23.2-20260726-UNOFFICIAL-alioth-camera-framework-system.img
~~~

## 8. Release checklist

1. Verify every hash from a freshly calculated SHA-256 sum.
2. Flash vendor_boot, sideload the ROM, then install the framework system image
   only with the correct active-slot procedure for the device.
3. If using GApps, reboot Lineage Recovery when prompted before sideloading
   NikGapps Core.
4. Boot without custom kernel/KernelSU changes and test Camera first.
5. Confirm provider devices, active com.android.camera client, output streams,
   requests, and frame metadata.
6. Upload only ROM-owned output files as release assets. Link to external
   packages such as NikGapps instead of mirroring them without permission.

## Build provenance

- Android source branch: LineageOS lineage-23.2.
- Device target: lineage_alioth-ap4a-userdebug.
- Build timestamp used in filenames: 2026-07-26.
- Camera framework modification: Android 13 and Android 14 hidden
  StreamConfigurationMap compatibility overloads retained on Android 16.
- Camera service modification: vendor provider Binder/RPC thread-pool lifecycle
  corrected so live requests could be served.

The main README contains on-device validation and troubleshooting. Treat those
tests as part of the build: an artifact is not a camera-enabled release until
the preview and capture pipeline have been verified on hardware.
