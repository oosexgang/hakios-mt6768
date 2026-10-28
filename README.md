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
| boot | works | `mtkloader/`: MMU off, eMMC through MSDC0, device tree for XNU; 4 cores |
| CPU | works | big cores clocked up (`mtk_cpufreq=3`) |
| storage | works, read only | the system image read off the eMMC as needed (`kernel/bsd/dev/redwood_emmc.c`, `emmc_root=1`): ~2.5 GB of 4 for iOS (docs/x6826b-emmc.md) |
| display | works | 720×1640, iOS at scale 2 |
| touch | works | OmniVision TD4160 over SPI (`kernel/bsd/dev/redwood_tcm.c`; the loader sends its firmware) |
| buttons | works | power, volume (`kernel/bsd/dev/redwood_mtkkeys.c`) |
| GPU | works | Mali-G52 brought up from the kernel (`redwood_mali.c`, `RedwoodMali.cpp`), Mesa's panfrost, a Metal driver: CoreAnimation draws through Metal, iOS 16's own wallpaper |
| sensors | partial | accelerometer through the SCP sensor hub (`redwood_scp.c`), screen rotation (docs/sensors-x6826b.md) |
| Wi-Fi / Bluetooth | in progress | CONNSYS powered, firmware in, its CPU running, the WMT command channel over BTIF/STP in full mode (`redwood_connsys.c`, docs/wifi-x6826b.md); no network yet |
| audio, camera, haptics, cellular | missing | |

## What is here

- `mtkloader/` — the bare-metal loader (MIT): chains in front of Android's kernel in `boot_a`, brings up
  what XNU cannot (touch firmware, PMIC, clocks), reads the eMMC, builds XNU's device tree and boot
  arguments.
- `kernel/` — this phone's drivers in XNU (APSL 2.0, see `kernel/README.md`).
- `tools/` — running it: `xnu-cycle.sh` (one run: build, flash `boot_a`, start XNU, fetch the log),
  `boot-xnu.sh`, `chainboot.py`, `system-flash.sh` (the system image into `xnusys`), `gpt-carve.py`,
  probes (`gpu-probe.sh`, `touch-probe.sh`, `msdc-test.sh`), log readers (`fbshot.py`, `frames.py`,
  `cache-name.py`, `rwm-recipes.py`).
- `pull-firmware.sh` — the phone's firmware out of **your** phone into `fw-*/` (needs root on Android).
- `docs/` — how each part was found and what each run showed.
- `device.conf` — the phone and the state of each component.

## Not here, by design

No file of iOS, no firmware of the phone, no screenshot (LEGAL.md). `mtkloader/build.sh` makes two
headers at build time and git ignores both: `touchfw.h` from your phone's `hdl_firmware.img`
(`pull-firmware.sh`), and `font8x16.h`, the loader's text font, rasterized on your Mac.

## Running

What it needs:
- an Infinix Hot 20 (X6826B) with an unlocked bootloader, rooted Android (Magisk), its stock
  `boot_a` image saved;
- a Mac with the common HakiOS tree (kernel, user space, the iOS image built from your own IPSW).

```sh
export HAKIOS_SERIAL=...             # your phone, as `adb devices` shows it
./pull-firmware.sh                   # fw-touch/, fw-connsys/
XNU_SECS=300 XNU_ARGS="cpus=4 touchscreen=2 buttons=2 mali=1 mtk_cpufreq=3 metal=1 camtl=1" \
    tools/xnu-cycle.sh                # one run; the log lands in logs/
```

The scripts here still expect the layout of the tree they were written in — `mtkloader/` and `tools/`
at the top of the common tree, next to `xnuloader/` (its `identity.h`, made from your IPSW),
`xnu-deps/` and `userland/`. Moving them under the common repository's `devices/<name>/` is the next
step. Kernel files keep the project's former prefix (`redwood_`, `kern.redwood.*`) until they are
merged into `hakios-xnu-kernel` under `hakios_` names.

## Licenses

`kernel/` is APSL 2.0 (`LICENSE.APSL`), as code built into XNU; everything else is MIT (`LICENSE`).
See `NOTICE` and `LEGAL.md`.
