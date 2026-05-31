# minicap — Agent / Maintainer Notes

## Prebuilt coverage

| SDK range | Source file | Android version |
|-----------|-------------|-----------------|
| 9–30      | `minicap_9.cpp` … `minicap_30.cpp` | Android 4–11 |
| 31–34     | `minicap_31.cpp` | Android 12–14 |
| **35**    | **`minicap_35.cpp`** | **Android 15 — built (arm64-v8a)** |

Available prebuilts in `jni/minicap-shared/aosp/libs/`: `android-9` through `android-35`
(arm64-v8a and armeabi-v7a for modern levels; x86/x86_64 only up to android-30 in the
upstream Makefile). **android-35 is currently arm64-v8a only** — the device fleet is
64-bit only (`DEVICE_IS_64BIT_ONLY`), so no 32-bit variant was built.

---

## Android 15 (SDK 35) port

### Symptom

The android-34 `.so` pushed to an Android 15 device fails to load at runtime:

```
CANNOT LINK EXECUTABLE "/data/local/tmp/minicap": cannot locate symbol
"_ZN7android21SurfaceComposerClient14destroyDisplayERKNS_2spINS_7IBinderEEE"
referenced by "/data/local/tmp/minicap.so"
```

`destroyDisplay` is only the **first** missing symbol. It is NOT a one-line fix:
compiling `minicap_31.cpp` against the Android 15 platform surfaces **six** distinct
`SurfaceComposerClient` / display-API changes that must all be ported. The history of
this section previously claimed a single `destroyDisplay` → `clear()` change — that was
wrong; the full set is below.

### Root cause — the six API changes (SDK 31 → 35)

All in `frameworks/native` (`libgui` / `libui`). Ported in `minicap_35.cpp`:

| # | Android 12 (SDK 31) | Android 15 (SDK 35) |
|---|---------------------|---------------------|
| 1 | `createDisplay(const String8&, bool secure)` | `createVirtualDisplay(const std::string&, bool isSecure, ...)` |
| 2 | `destroyDisplay(sp<IBinder>)` | `destroyVirtualDisplay(const sp<IBinder>&)` |
| 3 | `getPhysicalDisplayToken(PhysicalDisplayId(int))` | `PhysicalDisplayId` no longer constructs from `int`; enumerate via `getPhysicalDisplayIds()` → `std::vector<PhysicalDisplayId>` and index in |
| 4 | `getInternalDisplayToken()` | **removed** — use `getPhysicalDisplayIds()[0]` |
| 5 | `getStaticDisplayInfo(sp<IBinder> token, ...)` | `getStaticDisplayInfo(int64_t displayId, ...)` — takes `PhysicalDisplayId::value`, **not** the token |
| 6 | `setDisplayLayerStack(token, 0)` | second arg is now `ui::LayerStack` — use `ui::DEFAULT_LAYER_STACK` |

Plus one struct field rename:
- `ui::DisplayMode::refreshRate` → **`peakRefreshRate`** (`DisplayMode` also gained `vsyncRate`).

Unchanged (still take the `sp<IBinder>` token): `getDisplayState()`, `getActiveDisplayMode()`,
and the `Transaction` setters `setDisplaySurface()` / `setDisplayProjection()`.

### What `minicap_35.cpp` does

- `createVirtualDisplay(std::string("minicap"), false)` for the virtual display.
- `destroyVirtualDisplay(mVirtualDisplay)` to tear it down (the official replacement;
  do **not** rely on `sp::clear()` alone).
- `minicap_try_get_display_info()` treats the incoming `displayId` as an **index** into
  `getPhysicalDisplayIds()` (0 == primary), matching the old `getInternalDisplayToken()`
  fallback. It passes `physicalId.value` to `getStaticDisplayInfo()` but the
  `physicalId`'s **token** to `getDisplayState()` / `getActiveDisplayMode()`.
- `setDisplayLayerStack(mVirtualDisplay, ui::DEFAULT_LAYER_STACK)`.
- `info->fps = dconfig.peakRefreshRate;`

---

## Steps to add / rebuild Android 15 support

### 1. Source file

```bash
cp jni/minicap-shared/aosp/src/minicap_31.cpp \
   jni/minicap-shared/aosp/src/minicap_35.cpp
```

Apply the six API changes above. (The `Android.mk` `ifeq ($(PLATFORM_SDK_VERSION),35)`
case was also added for completeness, but the in-tree build below uses Soong, not that
Android.mk.)

### 2. Build the `.so` — Soong `Android.bp` (the method that actually works here)

`minicap.so` links **private platform libraries** (`libgui`, `libui`, `libbinder`),
not in the public NDK — an Android 15 AOSP tree is required. The legacy paths do **not**
work in a modern tree:

- The Docker `Makefile` targets ancient per-version AOSP trees — unusable.
- `OVERRIDE_PLATFORM_SDK_VERSION=35 mm` against `aosp/Android.mk` fails: that module is
  named `minicap`, which **collides** with the `minicap` executable in
  `jni/minicap/Android.mk` once the tree parses both.

Instead, a Soong module builds it cleanly. `jni/minicap-shared/aosp/Android.bp`:

```blueprint
cc_library_shared {
    name: "minicap_aosp_shared",   // unique name (avoids the "minicap" collision)
    stem: "minicap",               // output is still minicap.so
    srcs: ["src/minicap_35.cpp"],
    local_include_dirs: ["include"],
    shared_libs: [
        "libcutils", "libutils", "libbinder",
        "libui", "libgui", "liblog",
    ],
    cflags: ["-DPLATFORM_SDK_VERSION=35", "-Wno-unused-parameter"],
    compile_multilib: "64",        // 64-bit only device
}
```

**Prerequisite parse fix:** `jni/minicap-shared/Android.mk` (the mock used to satisfy the
binary's linker) starts with `LOCAL_PATH := $(abspath $(call my-dir))`. Modern AOSP
rejects the resulting **absolute** `LOCAL_C_INCLUDES` ("C_INCLUDES must be under the
source or output directories") and this aborts the Kati parse of *any* build that touches
`external/minicap`. Change it to:

```makefile
LOCAL_PATH := $(call my-dir)
```

Then build:

```bash
source build/envsetup.sh && lunch ts11-ap4a-userdebug
m minicap_aosp_shared
```

Output: `out/target/product/<product>/system/lib64/minicap.so` (stripped, ~52 KB).

### 3. Place the output

```bash
cp out/target/product/ts11/system/lib64/minicap.so \
   jni/minicap-shared/aosp/libs/android-35/arm64-v8a/minicap.so
```

The `minicap` **binary** is unchanged (public-API, dlopen/link-by-SONAME) — reuse the
existing `libs/arm64-v8a/minicap` prebuilt.

---

## Validation (device 0309240F08, SDK 35, arm64-v8a)

```bash
adb push libs/arm64-v8a/minicap                              /data/local/tmp/minicap
adb push libs/android-35/arm64-v8a/minicap.so               /data/local/tmp/minicap.so
adb shell chmod 755 /data/local/tmp/minicap

# Display info — exercises getPhysicalDisplayIds / getStaticDisplayInfo / getDisplayState
adb shell 'LD_LIBRARY_PATH=/data/local/tmp /data/local/tmp/minicap -i'
#   -> JSON: {"id":0,"width":2000,"height":1200,"fps":60.00,"rotation":90,...}

# Frame capture — exercises createVirtualDisplay / setDisplayLayerStack / destroyVirtualDisplay
adb shell 'cd /data/local/tmp && LD_LIBRARY_PATH=. ./minicap -P 2000x1200@2000x1200/0 -s > shot.jpg'
#   -> valid JPEG (ffd8 ffe0 ... JFIF), clean exit, no missing-symbol / crash
```

---

## Build system notes

- The `minicap` **binary** (executable) uses only public NDK APIs — buildable with
  `ndk-build` standalone; the existing prebuilt is reused across SDK levels.
- The `minicap.so` **shared library** uses private platform APIs — AOSP tree required.
- For this A15 tree, the working build path is **Soong (`Android.bp`)**, not the legacy
  `aosp/Android.mk` (name collision) or the Docker `Makefile` (ancient trees).
- `build-remote.sh` is the historical pattern: rsync sources to a remote AOSP builder,
  run `make`, rsync libs back — superseded here by the in-tree Soong build.

---

## Fleet integration note

The fleet manager pushes its **own** copy of `minicap.so` (the per-SDK prebuilt) to the
device on first screenshot use. For the fleet's `adb_screenshot` to work on Android 15,
the new `android-35/arm64-v8a/minicap.so` must be present in the fleet manager's minicap
prebuilt set (and the manager must select it for SDK 35). Until then, the manager falls
back to the nearest available prebuilt (android-34), which fails on Android 15 with the
`destroyDisplay` link error above.

## Fallback option (no build needed)

If an Android 15 build environment is not available, the fleet-manager can fall back to
`adb shell screencap -p` for devices with SDK ≥ 35. This is slower (~1–2 s vs ~100 ms)
and produces a full-resolution PNG rather than a downscaled JPEG, but requires no native
build.
