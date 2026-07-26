# Source modification bundle

The camera-enabled build changed two source-level areas before compilation.

## 1. Framework compatibility patch

File changed:

~~~text
frameworks/base/core/java/android/hardware/camera2/params/StreamConfigurationMap.java
~~~

The exact reusable unified patch is:

~~~text
0001-frameworks-base-stream-configuration-map-legacy-ctors.patch
~~~

It restores the hidden Android 13 and Android 14 constructors by delegating
them to Android 16's canonical constructor. Apply it to a LineageOS 23.2
android_frameworks_base checkout:

~~~bash
git apply patches/0001-frameworks-base-stream-configuration-map-legacy-ctors.patch
~~~

Then check it:

~~~bash
git diff --check
git diff -- core/java/android/hardware/camera2/params/StreamConfigurationMap.java
~~~

## 2. Vendor camera-provider thread-pool change

The second change was in the vendor camera-provider service startup path. That
file is implementation-specific: different alioth vendor trees use different
service sources, and the original full source checkout was removed after the
successful build. Therefore this repository does not falsely label a generic
snippet as an exact patch for a proprietary file.

The required semantic change is:

1. Start the provider's Binder/RPC thread pool before service registration.
2. Register the provider service.
3. Join the pool from the service process.

For a HIDL provider, the startup path must follow this pattern:

~~~cpp
android::hardware::configureRpcThreadpool(threadCount, true);

// Register the camera provider service.

android::hardware::joinRpcThreadpool();
~~~

For an AIDL provider, it must follow this pattern:

~~~cpp
ABinderProcess_setThreadPoolMaxThreadCount(threadCount);

// Register the camera provider service.

ABinderProcess_joinThreadPool();
~~~

Find the exact source in the builder's device/vendor tree:

~~~bash
rg -n "configureRpcThreadpool|joinRpcThreadpool|ABinderProcess|camera.*provider" device vendor hardware
~~~

Use only the API matching that service. Preserve the known-good thread count
from the vendor implementation; do not add both HIDL and AIDL lifecycles.

## Why proprietary files are not included

The provider binary, camera HAL, tuning data, firmware, and MiuiCamera package
are Xiaomi proprietary components. They must be extracted from a compatible
firmware release by a user who is authorized to obtain them. The repository
contains the source compatibility work and instructions, not redistributed
proprietary blobs.
