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
| `bsd/dev/redwood_connsys.c` | CONNSYS (Wi-Fi/Bluetooth): power, firmware, the MCU patch (PDA), BTIF, STP/WMT, `connsys_sw_init`; Bluetooth's function on, HCI over STP channel 0, the kernel control `com.apple.uart.sk.bluetooth` and the answers `bluetoothd` needs changed | `connsys_step=1..11`, `connsys_bt=1` |
| `bsd/dev/redwood_btsk.c` | Bluetooth's Skywalk kernel pipe: the channel `bluetoothd` of iOS 16.5 reads and writes H4 packets on | (with Bluetooth) |
| `iokit/Kernel/RedwoodBluetooth.cpp` | the registry's service `bluetooth` with `local-mac-address` (the Wi-Fi address plus one) | (with Bluetooth) |
| `bsd/dev/redwood_mtkbt_skel.c` | Bluetooth for another MediaTek phone: the same path with the phone as five operations (`struct mtkbt_platform`); not built here, compiles, has not run on a phone (`../docs/bluetooth-mtk-porting.md`) | `mtkbt_skel=1` |
| `bsd/dev/redwood_mtkwifi.c` | the Wi-Fi itself (gen4m CONNAC on AXI): firmware, scan, WPA2 join, `en0`; the network is listed and joined in Settings | `connsys_step=12..17` |
| `bsd/dev/redwood_wpa.c`, `bsd/sys/redwood_wpa.h` | the WPA2 supplicant: the 4-way handshake and the keys, built by the host | (with `connsys_step=17`) |
| `bsd/dev/redwood_mtkcam.c`, `bsd/sys/redwood_camera.h` | the main camera (S5K JN1): power, sensor, CAMSV_1 DMA, and `/dev/redwood-camera` for iOS (the ABI is the header) | `mtkcam=5` |

They are not a kernel on their own: they hook into the common kernel (its `bsd/conf/files`, the input
device `/dev/redwood-touch`, the display driver, `IOFindBSDRoot` for `emmc_root=1`), which lives in the
common repository. Until they are merged there they keep the project's former names (`redwood_*`,
`kern.redwood.*`); the merge renames them to `hakios_*`.

They are merged into the common repository's branch `mtk` as `mtk/mt6768/drivers/` (`hakios.*`, `kern.hakios.*`), where
`make x6826b_defconfig && make kernel` builds them; these copies are kept as they were.

`bsd/dev/redwood_mtkcam.c` shares the PMIC wrapper with `redwood_mtkkeys.c` and `redwood_connsys.c` through `redwood_pwrap_lock()`, which lives in the common
tree's `redwood_input.c`: `patches/redwood_pwrap_lock.patch` is that change. It also includes `redwood_mtkcam_jn1.h` and `redwood_mtkcam_rx.h` (the sensor's and the receiver's register tables),
which is the vendor's and is made on your machine from your own phone's capture (`tools/camtable.py`, `tools/camrx.py`), never kept in git. Its user-space side is `../userland/camera/`.

Bluetooth (`../docs/bluetooth-x6826b.md`): `redwood_connsys.c` calls `redwood_btsk.c` and `RedwoodBluetooth.cpp`; the Wi-Fi side gives the address (`RedwoodBluetoothAddress()`, called from the common tree's `redwood_net.c`), and `redwood_connsys_init()` is called from the common tree's `redwood_input.c`. Its user-space half (the jobs, `libredwood` inside `bluetoothd`, `BlueTool` and `Preferences`, `bt_test`) is the common repository's `userland/`.
