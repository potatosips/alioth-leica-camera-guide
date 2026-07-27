# Fix: newly captured photo thumbnail does nothing

## Symptom

MiuiCamera saves a new photo and shows its thumbnail, but tapping that
thumbnail does nothing. Closing or minimizing Camera and opening it again
makes the same thumbnail open in Google Photos.

This is not a MediaStore, permission, or Google Photos problem. The photo is
already present in `DCIM/Camera` and indexed. The difference is the Camera
process's post-capture state.

## Root cause

The Xiaomi Camera app checks whether `com.miui.gallery` is installed to choose
its Parallel Process Provider (PPP) compatibility version. On this vanilla
LineageOS build MIUI Gallery is absent. The app selects PPP version `4`, then
intentionally returns from its `gotoGallery` path for a newly-created
`external_primary` MediaStore URI. It therefore never sends the review intent.

After Camera is restarted, its thumbnail is reloaded through a different URI
path and the review intent can be sent. That explains why reopening Camera is a
workaround, but not a fix.

The Android photo picker is not applicable here: it is for selecting an input
photo. The correct action for a camera's recent-thumbnail control is a review
or view intent for the photo it just captured.

## Compatibility package

`tools/gallery-compat/AndroidManifest.xml` builds a deliberately minimal,
non-launcher package with the package name expected by MiuiCamera. It contains
no code, activities, services, permissions, media access, or gallery UI. Its
only metadata is:

```xml
<meta-data
    android:name="com.miui.gallery.target_ppp_version"
    android:value="2" />
```

That makes MiuiCamera use its supported PPP compatibility path. The camera
continues to send its review intent to Google Photos, which remains the image
viewer on this build.

Do not use this package on a ROM that already ships a real MIUI Gallery package
with the same name.

## Build

The released `LeicaGalleryCompat-v1.apk` was built from the manifest with
Android `aapt2`, Android platform `android.jar`, and a locally generated
signing key:

```powershell
aapt2 link -I android.jar --manifest AndroidManifest.xml `
  -o LeicaGalleryCompat-unsigned.apk

keytool -genkeypair -keystore leica-gallery-compat.keystore `
  -storepass changeit -keypass changeit -alias leicagallerycompat `
  -keyalg RSA -keysize 2048 -validity 10000 `
  -dname "CN=Leica Gallery Compatibility, OU=Local, O=Alioth, L=Dhaka, C=BD"

jarsigner -keystore leica-gallery-compat.keystore `
  -storepass changeit -keypass changeit `
  -signedjar LeicaGalleryCompat-v1.apk `
  LeicaGalleryCompat-unsigned.apk leicagallerycompat
```

The signing key is intentionally not published. Rebuilding the package with a
different key is fine as long as the prior compatibility package is uninstalled
first.

## Install and verify

```powershell
adb install LeicaGalleryCompat-v1.apk
adb shell am force-stop com.android.camera
adb shell am start -n com.android.camera/.Camera
```

Take a new photo, wait for its thumbnail, then tap the thumbnail. A working
log has both lines below without restarting Camera:

```text
CAM_GalleryHelper: gotoGallery: thumbnail uri=content://media/...
ActivityTaskManager: START ... com.google.android.apps.photos ...
```

To remove only the compatibility shim:

```powershell
adb uninstall com.miui.gallery
```
