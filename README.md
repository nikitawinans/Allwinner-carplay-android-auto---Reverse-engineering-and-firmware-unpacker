# Allwinner-carplay-android-auto---Reverse-engineering-and-firmware-unpacker
Allwinner Carplay / Android auto devices - Reverse engineering and firmware unpacker

* - ## Firmware unpacker in section 8

    
# Allwinner sun8iw19 PND/DVR Firmware Analysis (Tina Linux)
I analyzed more of 20 firmware images for the same Allwinner product line:
1.
2.
... and more

None of the images are encrypted.

## 1. Hardware
- **SoC:** Allwinner **sun8iw19** (V5xx series — confirmed via the
  `allwinner,sun8iw19p1` compatible string in the device tree `sunxi.fex`).
- **Board family:** `DEVICE_PRODUCT=v2.1`, codename **`tccdr`**
  ("touch-car-carplay-dongle-recorder") 
- **Screens:** 1600×600 (most models), 1600×720 (`T86B1A_..._1036`),
  854×480 (`R12`).
- **Wi-Fi/BT module:** Broadcom **AP6334** (AP6337 firmware also present on
  some builds as a fallback) plus a Realtek **RTL8733BS** driver bundled as
  an alternative (`extrootfs_v1x`) — board revisions can ship either chip.
- **Container format:** Allwinner **IMAGEWTY v3**.

- ## 2. Software
- - OS: **Tina Linux** (Allwinner's OpenWrt fork), version **3.5.1** , or version **4.0.0**.
- Init: BusyBox init + `/etc/rc.common` (OpenWrt-style procd scripts).
- Main application: **`usr/bin/sdvcam`** (~5 MB, stripped ARM/Thumb-2 ELF,
  musl libc) — this is the core UI/media binary and the one that contains
- `usr/lib/eyesee-mpp/`: `libcarplay.so`, `libairplay.so`,
  `liblylink{aa,client,server,ui,mw,usb}.so` (CarPlay/AirPlay stack and the
  vendor's own "lylink" wireless-dongle protocol), `libtxzEngineUSB.so`,
  OpenSSL 1.1.1, protobuf.
- GUI: **MiniGUI** (older builds) / an LVGL-style resource set under
  `usr/share/minigui` (the UI framework itself is internally called
  **`EyeseeLinux`**, confirmed by C++ mangled symbols in `sdvcam`).
- Partitions inside the `IMAGEWTY` container: `rootfs`, `usr`, `extrootfs`
  (+ `_v1x`/`_v2x` board-revision variants), `overlay`, `boot`/`recovery`
  (Android bootimg-format kernel), `misc`, `boot_logo`.
- Every image's `etc/img_config.cfg` carries an internal Jenkins build hash
  and branch name, tagged per reseller/batch (`SHUNXINDA`, `BLACKBOXMYCAR`,
  `XXTJUI`, `QDWIFISPSKSXTZSCDDKFWCD`.

  ## 3. Password
  | Where | Value | Scope | Source |
|---|---|---|---|
| `/etc/hostapd.conf` — main Wi-Fi AP password | **`88888888`** | All 20 images, identical | `wpa_passphrase=88888888` in plaintext |
| `/etc/hostapd.conf` — AP SSID | `lylink` (12 images) or `APCarplay` (8 images) | Split roughly evenly | plaintext |
| `usr/share/minigui/res/data/setting.lua` — CarPlay-dongle AP default password | **`12345678`** (`wifiap_carlink_pwd`) | | plaintext default |
| `usr/share/minigui/res/data/setting.lua` — RTSP AP default password | **`12345678`** (`wifiap_rtsp_pwd`) | Same scope | plaintext default |
| `/etc/shadow` — root password | **`tina`** | 20 of 20 images | DES-crypt hash `91rMiZzGliXHM`, cracked by brute force |
| `/etc/passwd` — root password hash | not cracked | All 20 images, byte-identical hash | MD5-crypt `$1$xvD2MFQL$mJLupzaSgmoFwggj1q6HC1` |

## 4. The hidden numeric codes in `sdvcam`

This is the headline finding. Inside **every one of the 20** `sdvcam`
binaries (same bytes, just at a different file offset per build — the
binary otherwise differs only in minor version bumps) there is a cluster of
plaintext ASCII strings sitting together in `.rodata`:

| Code | Effect | Confidence |
|---|---|---|
| **101010** | Reloads the boot logo from the SD card — confirmed by the device owner; matches the adjacent strings `ml_logo_size` / `ml_prompt_logo_update_success` ("logo update success" toast, found separately in the same string pool) | **Confirmed on real hardware** |
| **232323** | Toggles **"Wireless Debug"**. Two messages found: `"Debug Enabled Error"` and `"Wireless Debug Enabled!!!"` — the exact pair that fires presumably depends on current state/success. Given that `/etc/init.d/adbd` explicitly disables `ADB_TRANSPORT_PORT` (network ADB) in a commented-out line, this code may well be what re-enables ADB-over-Wi-Fi at runtime — that would tie directly into the "no SSH/Telnet, only USB ADB" finding below. Not confirmed. | Inferred, plausible |
| **787878** | Toggles **"System Debug"** on/off as a whole (`"System Debug Disabled!!!"` vs `"System Debug Enabled, Effective after reboot!!!"` — the "effective after reboot" wording means it's a persisted config flag, not an instant switch). This is very likely the gate that enables the two codes below, and probably also writes one or more of the `*_log_save_enable.bin` flag files described in §5. | Inferred, fairly confident |
| **888888** | **Exports the system log.** Message is state-dependent: `"System log export successful!!!"` if System Debug (787878) is already on, otherwise `"System Debug not enable yet!!!"` — i.e. 888888 depends on 787878 having been entered first. | Inferred, fairly confident |
| **898989** | Toggles a **frame/screen capture** feature (`"ScreenCapture enabled!!!"` / `"ScreenCapture disabled!!!"`, plus a separate `"ScreenCapture done!"` toast found elsewhere in the same string pool, next to `ml_prompt_logo_update_success`). | Inferred, fairly confident |

## 5. Where "System Debug" / log export / screen capture likely write to

No single consolidated "export path" string sits directly next to the toast
messages in §4, but the rest of `sdvcam`'s string pool strongly suggests the
following, all on the **root of the SD card** (`/mnt/extsd/`, i.e. *not*
inside the normal `DCIM/` recording folders):

- **System log:** at runtime the log is written to `/tmp/log` (RAM-backed,
  lost on reboot, lines prefixed `LOG-`/`CDR-`). No literal `cp`/`tar`
  command string referencing a persistent destination was found, but two
  sibling flag-file mechanisms were found that match the same
  "debug-enable gates logging-to-SD-card" pattern: `/mnt/extsd/
  awb_log_save_enable.bin` and `/mnt/extsd/isp_log_save_enable.bin` — when
  either file exists on the card, the ISP/auto-white-balance pipeline
  writes `.log` files directly to the SD-card root
  (`/mnt/extsd/awb_%d_%d_%d_%d_%d_%d.log`). 787878/888888 most plausibly
  create a similar flag file and then flush `/tmp/log`'s content out to the
  card the same way.
- **Screen/frame capture:** the only capture-shaped mechanism found is a
  pair of flag files, `/mnt/extsd/flag_capture` and `/mnt/udisk/
  flag_capture`, sitting right next to the raw-video-output code
  (`AW_MPI_VO_SendFrame`, `SaveYUV440`). When this flag file exists, the
  system dumps raw sensor/display frames as **`yuv440_%05d-%dx%d.nv21`**
  files straight into the SD-card root. These are *raw* NV21 YUV frames,
  not PNG/JPEG screenshots — a small NV21→image conversion step (ffmpeg can
  do it with the right `-pix_fmt`/`-s` flags once the resolution is known
  from the filename) would be needed to actually view them.

  ## 6. SSH / remote access

**No SSH or Telnet daemon exists in any of the 20 firmwares** — checked the
opkg package lists, every `/etc/init.d/*` and `/etc/rc.d/*` script, and
every binary under `usr/{bin,sbin,lib}` and `rootfs/{bin,sbin}` for
`sshd`/`dropbear`/`telnetd`. None present.

The only built-in remote-shell path is **ADB over USB** (not network):
`/etc/init.d/adbd` sets up a USB-gadget FunctionFS endpoint
(`idVendor=0x18d1`, `idProduct=0xD002` — Google's standard ADB VID/PID);
`ADB_TRANSPORT_PORT` (which would enable ADB-over-TCP) is explicitly
commented out in the init script (see the §4 note on code `232323` — it may
be exactly what flips this on).

The Wi-Fi AP (`lylink`/`APCarplay`, password `88888888`) is a client-pairing
network for the companion app, not a path to a shell — nothing listens on
SSH/Telnet behind it, only the proprietary CarPlay/"lylink" media protocol.

## 7. How to open `.fex` files

`.fex` is not one format — it's just the file extension Allwinner tooling
uses for pieces cut out of the `IMAGEWTY` container. What's actually inside
depends entirely on the file name:

| File(s) | Real format | How to open |
|---|---|---|
| `sys_config.fex`, `config.fex`, `sys_partition.fex`, `split_xxxx.fex`, `cardscript.fex`, `img_config.fex` | Plain text (INI-like, Chinese comments, sometimes GBK-encoded) | Any text editor; if it shows garbled characters, open as GBK/CP936 |
| `sunxi.fex` | Device tree blob (DTB) | `dtc -I dtb -O dts sunxi.fex -o sunxi.dts` (Device Tree Compiler) |
| `boot.fex`, `recovery.fex` | Android boot image (`ANDROID!` header) | `abootimg` / `mkbootimg` tooling, or Android Image Kitchen (AIK) |
| `rootfs.fex`, `usr.fex`, `extrootfs*.fex`, `res.fex` | **SquashFS 4 (xz)**, with a sometimes-shuffled superblock field order depending on build | `unpack_fw.py` (below) auto-detects both header layouts and writes a fixed `.squashfs` into `squashfs_fixed/`, openable directly in 7-Zip |
| `boot0_*.fex`, `u-boot*.fex`, `toc0/toc1.fex`, `fes1.fex`, `boot_package*.fex` | Allwinner boot-chain blobs (eGON/TOC/boot0) | Binary/vendor-specific; `sunxi-tools` / the `uboot-sunxi` project have parsers |
| `boot_logo.fex` | Plain JPEG (with Exif) — this is exactly the file code `101010` reloads from the SD card | Opens directly as an image |
| `misc.fex`, `*GPT*.fex`, `*MBR*.fex`, `dlinfo.fex`, `arisc.fex`, `aultools.fex`/`aultls32.fex`, `usbtool*.fex`, `cardtool.fex` | Factory/service binaries and partition tables | No user data inside; not worth opening |


## 8. Firmware unpacker (included)

**`unpack_fw.py`** — universal Allwinner `IMAGEWTY` unpacker. 
Pure Python 3,no dependencies, runs on Windows or Linux. 
It:
- splits a `.img`/`.zip` into every `.fex` file;
- auto-detects, for each SquashFS partition, whether its superblock is
  standard or field-shuffled (verified against all 16 images — both
  layouts occur depending on the build date);
- extracts SquashFS partitions into `tree/<partition>/`;
- flags other partition types it recognizes (ext*, UBI, Android bootimg).

- ```
python unpack_fw.py firmware.img out_dir
python unpack_fw.py firmware.zip  out_dir
```



  
