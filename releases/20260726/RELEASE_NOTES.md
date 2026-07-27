# POCO F3 (alioth) Leica/Miui Camera build — 2026-07-26

This is an unofficial Android 16 LineageOS 23.2 build for POCO F3
(alioth/aliothin) with Xiaomi MiuiCamera and the framework compatibility work
documented in this repository.

## Release assets

| File | Size | Purpose |
| --- | ---: | --- |
| lineage-23.2-20260726-UNOFFICIAL-alioth.zip | 1,527,669,804 bytes | Initial ROM installation |
| lineage-23.2-20260726-UNOFFICIAL-alioth-vendor_boot.img | 100,663,296 bytes | Matching vendor_boot / recovery image |
| lineage-23.2-20260726-UNOFFICIAL-alioth-camera-framework-system.img | 1,277,485,540 bytes | Final Android camera-framework compatibility correction |
| LeicaGalleryCompat-v1.apk | small | Fixes the first-tap recent-photo review issue on vanilla builds without MIUI Gallery |
| SHA256SUMS.txt | small | Checksums for the hosted files |

Verify every downloaded release asset before flashing:

~~~bash
sha256sum -c SHA256SUMS.txt
~~~

## Installation sequence

1. Confirm the bootloader is unlocked and copy off data you want to keep.
2. In Fastboot, flash the matching vendor_boot image:

   ~~~text
   fastboot flash vendor_boot lineage-23.2-20260726-UNOFFICIAL-alioth-vendor_boot.img
   ~~~

3. Boot the matching Lineage Recovery and install the LineageOS ZIP using the
   recovery's normal clean-install process.
4. Optional: if Google Play Store is wanted, reboot recovery when prompted and
   sideload the official Android 16 arm64 NikGapps Core package. The exact
   checked package is named NikGapps-core-arm64-16-20260222-signed.zip; its
   SHA-256 is documented in SHA256SUMS.txt but the file is not hosted here.
5. The ROM ZIP predates the final camera-framework correction. Check the active
   A/B slot with:

   ~~~text
   fastboot getvar current-slot
   ~~~

   Then flash the camera-framework system image only to that active system
   slot. The command used for this build was:

   ~~~text
   fastboot flash system_a lineage-23.2-20260726-UNOFFICIAL-alioth-camera-framework-system.img
   ~~~

   Replace system_a with system_b when b is the active slot. Do not use this
   command on another device or an unknown partition layout.
6. Boot system and test Camera before flashing a custom kernel, KernelSU, or
   performance modules.
7. If a newly captured photo's thumbnail does nothing until Camera is
   restarted, install `LeicaGalleryCompat-v1.apk`, restart Camera, and retest.
   This package is only for builds that do not already have real MIUI Gallery.
   Full rationale and source are in docs/POST_CAPTURE_REVIEW_FIX.md.

## Expected camera result

The Leica/Miui Camera UI should show a working preview and responsive controls.
The final verification found eight provider camera devices, an active
com.android.camera client, two output streams, advancing capture requests, and
recent frame metadata.

For the complete build method, framework patch, provider service requirement,
and troubleshooting steps, read the repository README and docs/BUILD.md.
