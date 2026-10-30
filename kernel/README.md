# kernel/

This phone's drivers in Apple's XNU (xnu-8796.121.2), as files of the kernel tree (the paths below are
the tree's). Licensed under the Apple Public Source License 2.0 (`../LICENSE.APSL`), as code built
into XNU.

| file | what | boot-arg |
| --- | --- | --- |
| `bsd/dev/redwood_emmc.c`, `bsd/sys/redwood_emmc.h` | the eMMC's `xnusys` partition as a read-only block device and the root | `emmc_root=1` |
| `bsd/dev/redwood_tcm.c` | the touch screen (TD4160, TCM protocol over SPI2) | `touchscreen=2` |
| `bsd/dev/redwood_mtkkeys.c` | the buttons; the big cores' clock | `buttons=2`, `mtk_cpufreq=` |
| `bsd/dev/redwood_mali.c` | the Mali-G52 brought up: power, clocks, MMU, jobs | `mali=1` |
| `iokit/Kernel/RedwoodMali.cpp` | the GPU for programs: panfrost's DRM interface as an IOKit user client | `mali=1` |
| `bsd/dev/redwood_scp.c` | the SCP sensor hub: accelerometer, `kern.redwood.accel` | `scp_step=7` |
| `bsd/dev/redwood_connsys.c` | CONNSYS (Wi-Fi/Bluetooth): power, firmware, BTIF, STP/WMT | `connsys_step=1..8` |

They are not a kernel on their own: they hook into the common kernel (its `bsd/conf/files`, the input
device `/dev/redwood-touch`, the display driver, `IOFindBSDRoot` for `emmc_root=1`), which lives in the
common repository. Until they are merged there they keep the project's former names (`redwood_*`,
`kern.redwood.*`); the merge renames them to `hakios_*`.

They are merged into the common repository's branch `mtk` as `mtk/mt6768/drivers/` (`hakios.*`, `kern.hakios.*`), where
`make x6826b_defconfig && make kernel` builds them; these copies are kept as they were.
