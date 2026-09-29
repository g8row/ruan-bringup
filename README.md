# Evolution X for the Redmi Pad Pro 5G / POCO Pad 5G (ruan)

**Status:** unofficial, **untested on hardware**. Test builds exist and are waiting for testers.

`ruan` is the 5G model of the Redmi Pad Pro (SM7435 "parrot"). The Wi-Fi model, `dizi`, runs Evolution X 16/17
and LineageOS 23.2 (see [dizi-bringup](https://github.com/g8row/dizi-bringup)). The two share the kernel, the
module code, the base device trees, the display, touch, pen, audio and cameras. ruan adds the modem, GNSS and
telephony. The full stock-ROM comparison is in [docs/ruan-vs-dizi.md](docs/ruan-vs-dizi.md).

So the ruan ROM is the dizi ROM with a thin ruan layer on top:

| Repo | Branch | What |
|---|---|---|
| [device_xiaomi_ruan](https://github.com/g8row/device_xiaomi_ruan) | `bka` | Includes dizi's `BoardConfig.mk`/`device.mk` with `DIZI_VENDOR := ruan` and `DIZI_TELEPHONY := true`. Adds the ruan stock `dtbo.img`, the modem HALs manifest (`manifest_ruan.xml`), `init.ruan.rc`, the QTI telephony packages and overlays |
| [vendor_xiaomi_ruan](https://github.com/g8row/vendor_xiaomi_ruan) | `bka` | Every blob from ruan HyperOS `OS3.0.303.0.WFSMIXM` (Global), including the ones shared with dizi |
| [device_xiaomi_dizi](https://github.com/g8row/device_xiaomi_dizi) | `bka` | The shared device tree |
| [device_xiaomi_dizi-kernel](https://github.com/g8row/device_xiaomi_dizi-kernel) | `bka` | Source-built 5.10 GKI Image and the stock modules (identical code on both models) |
| [vendor_xiaomi_dizi](https://github.com/g8row/vendor_xiaomi_dizi) | `bka` | dizi blobs (the dizi tree references it; ruan uses its own through `DIZI_VENDOR`) |

## Building

### 1. Sync Evolution X `bka` with the ruan local manifest

```sh
mkdir evox && cd evox
repo init -u https://github.com/Evolution-X/manifest -b bka --git-lfs
mkdir -p .repo/local_manifests
curl -o .repo/local_manifests/ruan.xml \
  https://raw.githubusercontent.com/g8row/ruan-bringup/main/manifest/ruan.xml
repo sync -c -j8 --no-tags
```

googlesource rate-limits large syncs. If some projects end up without a checkout, re-run
`repo sync -j2` for the paths that `repo list -p | while read p; do [ -e $p/.git ] || echo $p; done` prints.

### 2. Apply the platform patches

These are the same as for dizi. They live in `device/xiaomi/dizi/patches/<project path>/`.

```sh
for p in system/core build/make frameworks/native vendor/lineage vendor/gms; do
  for f in device/xiaomi/dizi/patches/$p/*.patch; do [ -f "$f" ] && git -C $p am -3 "$PWD/$f"; done
done
```

The patches:
- **system/core:** keep `/data/resource-cache` on upgrade.
- **build/make:** releasetools accepts a target-files zip, which signing needs.
- **frameworks/native:** realtime Vulkan queue priority.
- **vendor/lineage:** the kernel out dir for a relative `OUT_DIR`.
- **vendor/gms:** a uses-library fix.

### 3. Build

```sh
source build/envsetup.sh
lunch lineage_ruan-bp4a-user        # or -userdebug for a debug build
m evolution                         # flashable zip in out/target/product/ruan/
m superimage                        # for the fastboot package
```

- dizi and ruan build from the same tree. Soong parses every `Android.bp`, so a broken ruan module also breaks
  the dizi build, and the other way round.
- **Build dizi after every ruan change.** dizi-bringup's `tools/out-manifest.sh` diffs two builds of a device.
- With dizi-bringup's tooling, `DEVICE=ruan tools/build.sh <id>` keeps a separate `out-ruan` so dizi stays
  incremental. Switching devices then costs about 10 minutes, not a full build.

### 4. Tester package (signed)

```sh
WITH_ADB_INSECURE= DIZI_ADB_KEYS= DEVICE=ruan tools/build.sh ruan-test-N evolution superimage
DEVICE=ruan tools/build.sh ruan-test-N target-files-package otatools
DEVICE=ruan SIGN_BENCH=1 tools/sign-release.sh EvolutionX-16.0-<date>-ruan-test<N>-11.11-Unofficial
```

- The output is in `release/out/<name>/`:
  - the signed OTA zip;
  - `<name>-fastboot.zip` (images, `flash-ruan.sh`, INSTALL.md, collect-logs);
  - the recovery-side images.
- Use your own keys (dizi-bringup `keys/`, never committed). Updates only install over a build signed with the
  same keys.

## Installing

See [release/INSTALL.md](release/INSTALL.md). There are two paths:
- Fastboot the recovery-side images, format data in recovery, and sideload the zip.
- Or run the all-fastboot [release/flash-ruan.sh](release/flash-ruan.sh).

Before either:
- The tablet needs HyperOS **OS3.0.303.0 or newer** firmware.
- **Never relock the bootloader.**

## Reporting problems

Run [release/collect-logs.sh](release/collect-logs.sh) (Linux/macOS) or `collect-logs.bat` (Windows) with the
tablet connected. It saves `adb bugreport`, logcat, dmesg and the telephony state. Send the archive, even if
everything works.

## What to test first on ruan

1. **Calls, SMS and mobile data** (qcril, IMS). This is the main new part.
2. **GPS**: the GNSS HAL and the `gps` group.
3. **SIM detection, and both SIM trays if present.**
4. **Everything shared with dizi:** Wi-Fi, BT, cameras (from the ruan dtbo), audio, pen, charging.

## Known differences from dizi, and open points

- The ruan dtbo also carries the dizi board. The dizi ROM's ruan entry has no camera nodes, which is why ruan
  ships its own `prebuilts/dtbo.img`.
- **Telephony features stay on** (`DIZI_TELEPHONY := true`). The dizi build drops them, because the Wi-Fi model
  otherwise crash-loops `com.qti.phone`.
- **Carrier config and APNs:** they come from `vendor/lineage`. Some carriers may need entries.
- **Android 17 (cnb):** `device_xiaomi_ruan` and `vendor_xiaomi_ruan` have a `cnb` branch (64-bit only, on top
  of dizi's `cnb`). Build it from an Evolution X `cnb` tree with `lunch lineage_ruan-cp2a-user`. The first signed
  build, `EvolutionX-17.0-20260929-ruan-12.2-Unofficial`, is untested on hardware; see
  [release/INSTALL-cnb.md](release/INSTALL-cnb.md). Local manifest: [manifest/ruan-cnb.xml](manifest/ruan-cnb.xml).
