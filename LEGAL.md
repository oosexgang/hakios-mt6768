# Legal

HakiOS is an independent research project. It is **not affiliated with, endorsed by, or connected
to Apple Inc., MediaTek Inc. or Infinix (Transsion Holdings).** iOS, iPhone and Metal are
trademarks of Apple Inc.; Helio and MediaTek are trademarks of MediaTek Inc.; Infinix is a
trademark of its owner. They are named only to say what this works with.

What this repository does **not** contain, and must never contain:

- **Nothing of Apple's**: no iOS binary, image, cache, library, shader library, font, picture or
  screenshot, no header derived from iOS firmware. The iOS user space is built on the user's own
  machine from an IPSW the user downloads from Apple's public servers (the common repository's
  `sysimage/fetch.sh`). The loader's text font is rasterized at build time on the builder's Mac
  (`tools/mkfont8x16.swift`); the bitmap is not kept.
- **Nothing of the vendor's**: no firmware of the phone (touch controller, CONNSYS Wi-Fi/Bluetooth,
  sensor hub, GPU), no partition image, no device tree taken off a phone. `pull-firmware.sh` copies
  the firmware from the user's own phone into `fw-*/`, which git ignores.
- **Nothing to impersonate an Apple device to Apple's services**: no keys, certificates, activation
  data or real serial numbers. The serial number the loader gives iOS is a placeholder
  (`XNU_SERIAL`, `HAKIOS0000000000`).

Running it changes nothing permanent on the phone except what the user flashes themself: the
loader goes into the `boot_a` partition in front of Android's own kernel, and the phone returns to
Android after every run. Unlocking the bootloader, rooting and flashing are at the user's own risk.

If you believe a file here infringes anyone's rights, open an issue and it will be removed.
None of this is legal advice.
