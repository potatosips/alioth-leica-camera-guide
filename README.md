# Leica/Miui Camera on LineageOS 23.2 for POCO F3 (alioth / aliothin)

This document records the source changes, build flow, installation order, and
on-device validation used to make the Xiaomi MiuiCamera / Leica camera
application work on the POCO F3 running Android 16-based LineageOS 23.2.

It is intentionally a source-level integration guide, not a generic camera APK
or flashable-ZIP guide. A camera UI can launch while its preview is black or its
controls do nothing, because a working camera needs the application, Android
framework, camera provider, HAL, firmware, tuning data, and SELinux policy to
agree with each other.

## Final result

| Item | Used value |
| --- | --- |
| Device | POCO F3, codename alioth / aliothin |
| Android base | LineageOS 23.2, Android 16 |
| Camera | Xiaomi MiuiCamera with the Leica-branded interface |
| GApps | NikGapps Core, Play Store-only configuration |
| ROM ZIP | lineage-23.2-20260726-UNOFFICIAL-alioth.zip |
| Boot/recovery image | lineage-23.2-20260726-UNOFFICIAL-alioth-vendor_boot.img |
| Framework iteration image | lineage-23.2-20260726-UNOFFICIAL-alioth-camera-framework-system.img |

The final verification was stronger than merely launching the camera app:

1. Android reported eight camera devices from the provider.
2. The active client was com.android.camera.
3. The app configured two preview output streams.
4. Camera request counters advanced and recent frame metadata appeared.

That is evidence of a real capture session, not an inactive UI.

## Scope and licensing

- Leica camera here means Xiaomi's MiuiCamera package, its Leica-branded UI,
  and the matching Xiaomi camera stack. This guide does not recreate or claim
  ownership of proprietary Leica image processing.
- Do not redistribute Xiaomi APKs, proprietary blobs, firmware, calibration,
  or tuning files unless you have permission. Extract them from firmware you
  are entitled to use, or require each builder to extract their own copy.
- KernelSU, the custom kernel, and uclamp profiles were installed separately on
  this phone. They are not required for the camera fix and should be excluded
  while validating the camera stack.

## Why a random camera APK or ZIP does not solve it

The actual camera path is:

~~~text
MiuiCamera.apk
  -> Android camera2 framework (framework.jar)
    -> camera provider service and Binder/RPC thread pool
      -> Xiaomi camera HAL, tuning libraries, and firmware
        -> physical cameras
~~~

The original failure was that the app opened but showed no usable preview. That
proved only that the APK could start. It did not prove that the app could open a
camera device, configure streams, submit requests, or receive frames.

There were two required compatibility fixes:

1. The Android 16 framework had progressed beyond hidden
   StreamConfigurationMap constructor signatures used by older Xiaomi camera
   provider binaries. Android 14 introduced JPEG-R arguments; Android 16 adds
   HEIC Ultra-HDR arguments. The Android 13 and Android 14 signatures had to
   remain available.
2. The vendor camera provider needed a live Binder/RPC service thread pool so
   it could serve real device-open and capture-session calls.

The fix does not replace the real HAL or fake camera metadata. It preserves
Android 16's canonical implementation and restores only the legacy hidden
constructor entry points as delegating overloads.

## Prerequisites

### Host requirements

- Linux build environment. WSL2 works, though a native Linux filesystem is
  faster for a full Android tree.
- At least 250 GB free storage for the checkout, compiler cache, and outputs.
- The JDK and packages supported by the LineageOS 23.2 branch, plus Git, repo,
  Python, ccache, build tools, adb, and fastboot.
- An unlocked device and a recovery that can install LineageOS packages.

### Source and blob requirements

Use one compatible set of:

~~~text
alioth device tree + kernel tree + vendor tree + vendor firmware
+ Xiaomi camera provider/HAL + camera tuning data + MiuiCamera package
~~~

Never infer compatibility from the MiuiCamera APK version alone. The provider,
HAL libraries, vendor camera configuration, and firmware are the important
matching unit. Mixing pieces from different MIUI/HyperOS Android generations is
a frequent cause of a black preview, provider crash, or broken capture.

## 1. Initialize LineageOS 23.2

The project uses the LineageOS 23.2 manifest:

~~~bash
mkdir -p ~/android/lineage-alioth
cd ~/android/lineage-alioth
repo init -u https://github.com/LineageOS/android.git -b lineage-23.2 --git-lfs
repo sync -c --force-sync --no-clone-bundle --no-tags -j"$(nproc)"
~~~

Add your compatible alioth device, kernel, hardware, and vendor projects using
local manifests. Pin their revisions while debugging. Updating half of the
trees after proving one working combination makes camera regressions impossible
to diagnose.

Build and boot a baseline first. Confirm the expected device, Android version,
and provider list:

~~~bash
adb shell getprop ro.product.device
adb shell getprop ro.build.version.release_or_codename
adb shell dumpsys media.camera
~~~

## 2. Integrate MiuiCamera as a privileged system app

MiuiCamera must be part of the ROM image, rather than installed as a normal
user APK. The final package location is equivalent to:

~~~text
system/priv-app/MiuiCamera/MiuiCamera.apk
~~~

Use either an Android prebuilt module or a controlled copy-file entry in the
device product. A typical presigned prebuilt definition is:

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

Add it to the product:

~~~make
PRODUCT_PACKAGES += MiuiCamera
~~~

Use a genuine pre-signed proprietary APK with presigned enabled. Do not
re-sign the Xiaomi APK with the platform key unless you understand the signature
and permission consequences.

### Required companion files

Compare the stock firmware and the custom build for camera-specific:

- privileged permission XML files;
- package configuration;
- SELinux labels and policy;
- camera provider service declarations;
- vendor tuning files under vendor/etc/camera;
- camera HAL libraries under vendor/lib64/hw and related paths.

Useful inspection commands:

~~~bash
adb shell pm path com.android.camera
adb shell dumpsys package com.android.camera
adb shell ls -lZ /system/etc/permissions /system_ext/etc/permissions /vendor/etc/permissions
adb shell ls -lZ /vendor/bin/hw /vendor/lib64/hw /vendor/etc/camera
~~~

Do not broadly grant privileged permissions to unrelated packages. Add only the
package-specific permissions and configuration that the matching stock camera
package requires.

## 3. Keep the Xiaomi provider and HAL coherent

Do not replace the Xiaomi provider with a generic AOSP provider. Keep the
matching vendor pieces together:

- camera provider service binary and init declaration;
- HAL implementation and supporting camera libraries;
- camera XML/tuning/calibration files;
- the vendor image and firmware that the device tree expects.

Before and after a change, inventory the running stack:

~~~bash
adb shell dumpsys media.camera
adb shell getprop | grep -i camera
adb shell ps -A -o USER,PID,NAME | grep -i camera
~~~

If the provider lists no devices, crashes, or is missing, repair the coherent
vendor extraction instead of copying individual files from random builds.

## 4. Restore StreamConfigurationMap compatibility

### File changed

Edit:

~~~text
frameworks/base/core/java/android/hardware/camera2/params/StreamConfigurationMap.java
~~~

Android 13 exposed a hidden constructor with no JPEG-R fields. Android 14 added
three JPEG-R arrays. The Android 16 canonical constructor also carries three
HEIC Ultra-HDR arrays. Older compiled provider code can resolve either old
signature at runtime, so Android 16 needs both legacy signatures.

### Correct implementation rule

Do not overwrite Android 16's canonical constructor body with Android 13/14
code. Add two hidden delegating constructors directly above the Android 16
canonical constructor:

- Android 13 overload: passes null for JPEG-R and HEIC Ultra-HDR arrays.
- Android 14 overload: forwards its JPEG-R arrays and passes null for HEIC
  Ultra-HDR arrays.
- Both retain enforceImplementationDefined as true.

The Android 16 implementation converts null JPEG-R and HEIC Ultra-HDR sets to
empty sets. Therefore the compatibility overloads preserve modern behavior
without inventing unsupported formats.

### Reference patch

Formatting may vary by branch, but the patch structure should be exactly this:

~~~java
// Android 13 binary-compatibility constructor. @hide
public StreamConfigurationMap(
        StreamConfiguration[] configurations,
        StreamConfigurationDuration[] minFrameDurations,
        StreamConfigurationDuration[] stallDurations,
        StreamConfiguration[] depthConfigurations,
        StreamConfigurationDuration[] depthMinFrameDurations,
        StreamConfigurationDuration[] depthStallDurations,
        StreamConfiguration[] dynamicDepthConfigurations,
        StreamConfigurationDuration[] dynamicDepthMinFrameDurations,
        StreamConfigurationDuration[] dynamicDepthStallDurations,
        StreamConfiguration[] heicConfigurations,
        StreamConfigurationDuration[] heicMinFrameDurations,
        StreamConfigurationDuration[] heicStallDurations,
        HighSpeedVideoConfiguration[] highSpeedVideoConfigurations,
        ReprocessFormatsMap inputOutputFormatsMap,
        boolean listHighResolution) {
    this(configurations, minFrameDurations, stallDurations,
            depthConfigurations, depthMinFrameDurations, depthStallDurations,
            dynamicDepthConfigurations, dynamicDepthMinFrameDurations,
            dynamicDepthStallDurations,
            heicConfigurations, heicMinFrameDurations, heicStallDurations,
            // jpegRConfigurations, jpegRMinFrameDurations, jpegRStallDurations
            null, null, null,
            // heicUltraHDRConfigurations, heicUltraHDRMinFrameDurations,
            // heicUltraHDRStallDurations
            null, null, null,
            highSpeedVideoConfigurations, inputOutputFormatsMap,
            listHighResolution, true);
}

// Android 14 binary-compatibility constructor. @hide
public StreamConfigurationMap(
        StreamConfiguration[] configurations,
        StreamConfigurationDuration[] minFrameDurations,
        StreamConfigurationDuration[] stallDurations,
        StreamConfiguration[] depthConfigurations,
        StreamConfigurationDuration[] depthMinFrameDurations,
        StreamConfigurationDuration[] depthStallDurations,
        StreamConfiguration[] dynamicDepthConfigurations,
        StreamConfigurationDuration[] dynamicDepthMinFrameDurations,
        StreamConfigurationDuration[] dynamicDepthStallDurations,
        StreamConfiguration[] heicConfigurations,
        StreamConfigurationDuration[] heicMinFrameDurations,
        StreamConfigurationDuration[] heicStallDurations,
        StreamConfiguration[] jpegRConfigurations,
        StreamConfigurationDuration[] jpegRMinFrameDurations,
        StreamConfigurationDuration[] jpegRStallDurations,
        HighSpeedVideoConfiguration[] highSpeedVideoConfigurations,
        ReprocessFormatsMap inputOutputFormatsMap,
        boolean listHighResolution) {
    this(configurations, minFrameDurations, stallDurations,
            depthConfigurations, depthMinFrameDurations, depthStallDurations,
            dynamicDepthConfigurations, dynamicDepthMinFrameDurations,
            dynamicDepthStallDurations,
            heicConfigurations, heicMinFrameDurations, heicStallDurations,
            jpegRConfigurations, jpegRMinFrameDurations, jpegRStallDurations,
            // heicUltraHDRConfigurations, heicUltraHDRMinFrameDurations,
            // heicUltraHDRStallDurations
            null, null, null,
            highSpeedVideoConfigurations, inputOutputFormatsMap,
            listHighResolution, true);
}
~~~

The old parameter order is the important ABI contract. A missing, reordered,
or incorrectly typed parameter will still produce a runtime constructor
resolution failure.

Inspect the source diff before compiling:

~~~bash
git -C frameworks/base diff -- core/java/android/hardware/camera2/params/StreamConfigurationMap.java
~~~

## 5. Give the provider a working service thread pool

Camera enumeration is not a complete test. The provider must process device
open and capture-session requests on an available Binder/RPC thread pool.

Find the provider implementation and init declaration:

~~~bash
rg -n "camera.*provider|configureRpcThreadpool|joinRpcThreadpool|ABinderProcess" device vendor hardware
~~~

Use the service idiom matching the existing provider type.

For a HIDL provider:

~~~cpp
android::hardware::configureRpcThreadpool(/* maxThreads */ N,
        /* callerWillJoin */ true);

// Register the provider service.

android::hardware::joinRpcThreadpool();
~~~

For an AIDL provider:

~~~cpp
ABinderProcess_setThreadPoolMaxThreadCount(N);

// Register the provider service.

ABinderProcess_joinThreadPool();
~~~

Keep the vendor's known-good thread count when it exists. Do not add both
HIDL and AIDL lifecycle code to one service. The required result is a
registered provider with a live thread pool, rather than a service that can
enumerate but stalls on real work.

## 6. Build

From the Android tree:

~~~bash
source build/envsetup.sh
lunch lineage_alioth-ap4a-userdebug
mka bacon -j"$(nproc)"
~~~

The precise lunch target may differ if the device tree defines a different
product, but it must be the Android 16 Lineage target for alioth.

Record checksums for every release artifact:

~~~bash
sha256sum out/target/product/alioth/lineage-*.zip out/target/product/alioth/*vendor_boot*.img > SHA256SUMS.txt
~~~

For a later framework-only iteration, rebuild and package the relevant system
image using the normal product build flow. Do not flash a generic system image
to a dynamic-partition device without confirming the exact slot and partition
layout.

## 7. Flash order

The working install sequence was:

1. Boot to Fastboot.
2. Flash the matching vendor_boot image:

   ~~~powershell
   fastboot flash vendor_boot lineage-23.2-20260726-UNOFFICIAL-alioth-vendor_boot.img
   ~~~

3. Boot the matching Lineage Recovery.
4. Perform the clean-install wipe/format required by the ROM instructions.
   Do not blindly apply an OrangeFox/TWRP advanced-wipe preset to a dynamic
   partition layout.
5. Sideload the LineageOS ZIP.
6. When recovery asks to reboot recovery before installing additional
   packages, choose Yes. After recovery restarts, sideload NikGapps Core:

   ~~~powershell
   adb sideload NikGapps-core-arm64-16-20260222-signed.zip
   ~~~

7. Boot system, finish first boot, enable USB debugging, and test the camera
   before flashing a custom kernel, KernelSU module, or scheduler profile.

The later framework image was flashed only after checking the active A/B slot:

~~~powershell
fastboot getvar current-slot
~~~

Use your device's active slot and the install method appropriate for your
partition layout. Never copy a system_a command blindly to another device.

## 8. Validate a real camera session

### Package and provider

~~~bash
adb shell pm path com.android.camera
adb shell dumpsys media.camera
adb shell ps -A -o USER,PID,NAME | grep -i camera
~~~

Confirm that com.android.camera resolves to the intended system MiuiCamera.apk,
the provider lists the expected cameras, and the provider process stays alive.

### Test through the launcher

Open Camera from Launcher3, not only with an adb shell activity command. Grant
permissions if prompted. Verify:

- live preview;
- front/rear switch;
- photo capture and saved image;
- video record and stop;
- responsive modes and settings;
- repeat open/close stability.

### Confirm streams and frames

With Camera open:

~~~bash
adb shell dumpsys media.camera | grep -E +  "Client|com.android.camera|Stream|Request|frame|Camera ID"
~~~

A good session has an active com.android.camera client, configured output
streams, increasing requests, and frame metadata. A process merely appearing in
the process list is not proof of camera operation.

## Troubleshooting

| Symptom | Most likely cause | Correct action |
| --- | --- | --- |
| App opens but preview is black | Provider thread-pool or framework ABI mismatch | Check logcat, service state, and both compatibility constructors. |
| NoSuchMethodError for StreamConfigurationMap | Missing or incorrect Android 13/14 constructor signature | Reapply the overloads with the exact parameter order and rebuild framework/system. |
| No camera devices listed | Mismatched vendor image, missing HAL/config, or provider service failure | Restore a coherent device/vendor/firmware extraction. |
| Provider crashes when the app opens | Framework/vendor mismatch or SELinux denial | Capture logcat and dmesg; do not repeatedly reinstall the APK. |
| Camera disappears after OTA | Camera package/config was not integrated into the source build | Add the prebuilt and required XML/configuration to the device product, then rebuild. |
| GApps cannot install after the ROM ZIP | Recovery has not entered its post-install state | Reboot recovery when Lineage Recovery asks, then sideload the add-on. |
| Fastboot cannot find the image | Wrong working directory or filename | Use an absolute path or verify the file location before running fastboot. |

Capture logs cleanly:

~~~bash
adb logcat -c
adb logcat -b all -v color | grep -Ei "camera|provider|CameraService|StreamConfigurationMap|avc: denied"
~~~

## Maintenance

1. Re-test after every Android platform/framework rebase.
2. Keep the Android 13/14 overloads until the Xiaomi provider is rebuilt
   against Android 16's API shape.
3. Update camera APK, provider, HAL, blobs, and firmware as a tested set.
4. Keep the camera patch separate from kernel, KernelSU, and performance tweaks.
5. Publish source revisions and SHA-256 checksums with every release.

## References

- LineageOS 23.2 manifest:
  https://github.com/LineageOS/android/tree/lineage-23.2
- LineageOS framework base:
  https://github.com/LineageOS/android_frameworks_base/tree/lineage-23.2
- Android 13 StreamConfigurationMap:
  https://android.googlesource.com/platform/frameworks/base/+/android-13.0.0_r83/core/java/android/hardware/camera2/params/StreamConfigurationMap.java
- Android 14 StreamConfigurationMap:
  https://android.googlesource.com/platform/frameworks/base/+/android-14.0.0_r33/core/java/android/hardware/camera2/params/StreamConfigurationMap.java

When the camera regresses, begin with provider devices, active client,
configured streams, frame metadata, and the first relevant logcat error. That
is safer and faster than swapping random APKs or flashable ZIPs.
