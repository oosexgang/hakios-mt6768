# hakios-mt6768

HakiOS for the **Infinix Hot 20 (X6826B)**: MediaTek Helio G85 (**MT6768**), GPU **Mali-G52 MC2**, eMMC.
iOS 16.5's own user space (launchd → backboardd → SpringBoard → apps) on Apple's open XNU kernel
(xnu-8796.121.2), with this phone's own loader and drivers. The first MediaTek phone of the project;
everything else in the organisation is Qualcomm.

Nothing on the phone is replaced for good: `mtkloader` sits in `boot_a` in front of Android's own kernel,
starts XNU once when Android asks for it, and the phone comes back to Android after every run. The iOS
system image lives in a partition of its own (`xnusys`, carved from the end of `userdata`) and the kernel
reads it off the eMMC as it needs it.

## What works

| | | |
| --- | --- | --- |
| boot | works | mtkloader (`mtk/loader/` of the common tree): MMU off, eMMC through MSDC0, device tree for XNU; 4 cores |
| CPU | works | big cores clocked up (`mtk_cpufreq=3`) |
| storage | works, read only | the system image read off the eMMC as needed (`hakios.mtk_emmc.c`, `emmc_root=1`): ~2.5 GB of 4 for iOS (docs/x6826b-emmc.md) |
| display | works | 720×1640, iOS at scale 2 |
| touch | works | OmniVision TD4160 over SPI (`hakios.tcm.c`; the loader sends its firmware) |
| buttons | works | power, volume (`hakios.mtk_keys.c`) |
| GPU | works | Mali-G52 brought up from the kernel (`hakios.mali.c`, `hakios.mali.cpp`), Mesa's panfrost, a Metal driver: CoreAnimation draws through Metal, iOS 16's own wallpaper |
| sensors | partial | accelerometer through the SCP sensor hub (`hakios.mtk_scp.c`), screen rotation (docs/sensors-x6826b.md) |
| Wi-Fi / Bluetooth | in progress | CONNSYS powered, firmware in, its CPU running, the WMT command channel over BTIF/STP, the MCU patch by PDA taken (`hakios.mtk_connsys.c`, docs/wifi-x6826b.md); no network yet |
| audio, camera, haptics, cellular | missing | |

## What is here

This is the device's repository; everything else is the common tree's branch `mtk`
([hakios-xnu-kernel, branch mtk](https://github.com/oosexgang/hakios-xnu-kernel/tree/mtk)): the kernel and the MT6768's drivers
(`mtk/mt6768/`), the loader (`mtk/loader/`, mtkloader) and the scripts that build `boot_a` and run the phone (`mtk/tools/`).

- `device.conf` — the phone and the state of each component.
- `configs/` — `x6826b_defconfig` and `x6826b.config`: the kernel's configuration for this phone.
- `pull-firmware.sh` — the phone's firmware out of **your** phone into `fw-*/` at the top of the tree (needs root on Android).
- `docs/` — how each part was found and what each run showed.

## Not here, by design

No file of iOS, no firmware of the phone, no screenshot (LEGAL.md). The loader's build makes two headers and git ignores both:
`touchfw.h` from your phone's `hdl_firmware.img` (`pull-firmware.sh`), and `font8x16.h`, the loader's text font, rasterized on
your Mac.

## Running

What it needs:
- an Infinix Hot 20 (X6826B) with an unlocked bootloader, rooted Android (Magisk), its stock
  `boot_a` image saved (`backup/boot_a_magisk.img` at the top of the tree);
- a Mac, your own Darwin userland 16.5 IPSW (for `xnuloader/identity.h`).

```sh
git clone -b mtk https://github.com/oosexgang/hakios-xnu-kernel.git hakios && cd hakios
tools/hakios-device.py sync x6826b      # this repository, into devices/x6826b
export HAKIOS_SERIAL=...                # your phone, as `adb devices` shows it
devices/x6826b/pull-firmware.sh         # fw-touch/, fw-connsys/
xnuloader/mkidentity.sh your.ipsw
make x6826b_defconfig && make kernel    # hakios-kernel: mali mtk_keys mtk_scp tcm mtk_emmc mtk_connsys
make phone SECS=300 RAMDISK=...         # boot_a written, XNU once, Android back, the log in logs/
```

The user space that runs on this phone (Mesa's panfrost, the Metal driver) still reads the project's former names
(`kern.redwood.*`, `RedwoodMali`) and is not yet in the common tree; `RAMDISK=` and the system image in `xnusys` are its.

## Licenses

Everything here is MIT (`LICENSE`); the kernel drivers this phone uses are APSL 2.0 in the common repository's `mtk/`.
See `NOTICE` and `LEGAL.md`.
