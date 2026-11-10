# hakios-mt6768

[English](README.md) · Русский

HakiOS для **Infinix Hot 20 (X6826B)**: MediaTek Helio G85 (**MT6768**), GPU **Mali-G52 MC2**, eMMC.
Родной userland iOS 16.5 (launchd → backboardd → SpringBoard → приложения) на открытом ядре Apple XNU
(xnu-8796.121.2), со своим загрузчиком и драйверами этого телефона. Первый MediaTek-телефон проекта;
всё остальное в организации — Qualcomm.

На телефоне ничего не заменяется навсегда: `mtkloader` встаёт в `boot_a` перед родным ядром Android,
по запросу из Android один раз запускает XNU, и после каждого прогона телефон возвращается в Android.
Образ системы iOS лежит в отдельном разделе (`xnusys`, выкроен из конца `userdata`), и ядро читает его
с eMMC по мере надобности.

## Что работает

| | | |
| --- | --- | --- |
| загрузка | работает | mtkloader (`mtk/loader/` общего дерева): MMU выключен, eMMC через MSDC0, дерево устройства для XNU; 4 ядра |
| CPU | работает | большие ядра подняты по частоте (`mtk_cpufreq=3`) |
| накопитель | работает, только чтение | образ системы читается с eMMC по мере надобности (`hakios.mtk_emmc.c`, `emmc_root=1`): ~2.5 ГБ из 4 для iOS (docs/x6826b-emmc.md) |
| экран | работает | 720×1640, iOS в масштабе 2 |
| тач | работает | OmniVision TD4160 по SPI (`hakios.tcm.c`; прошивку шлёт загрузчик) |
| кнопки | работает | питание, громкость (`hakios.mtk_keys.c`) |
| GPU | работает | Mali-G52 поднят из ядра (`hakios.mali.c`, `hakios.mali.cpp`), panfrost в Mesa, драйвер Metal: CoreAnimation рисует через Metal, родные обои iOS 16 |
| датчики | частично | акселерометр через сенсорный хаб SCP (`hakios.mtk_scp.c`), поворот экрана (docs/sensors-x6826b.md) |
| Wi-Fi | работает | драйвер gen4m: скан, подключение WPA2 из Настроек, интернет и Safari через него (`hakios.mtk_connsys.c`, docs/wifi-x6826b.md) |
| Bluetooth | частично | `bluetoothd` из iOS 16.5 на MT6631 (HCI через STP): включён, приложения видят настоящие BLE-устройства, в Настройках переключатель, имя и устройства; подтверждённого сопряжения, профилей и звука пока нет (docs/bluetooth-x6826b.md; для другого телефона MediaTek docs/bluetooth-mtk-porting.md) |
| звук, камера, вибро, сотовая | нет | |

## Что здесь лежит

- `mtkloader/` — bare-metal-загрузчик (MIT): встаёт перед ядром Android в `boot_a`, поднимает то, что XNU
  не умеет (прошивка тача, PMIC, тактирование), читает eMMC, строит дерево устройства и boot-аргументы XNU.
- `kernel/` — драйверы этого телефона в XNU (APSL 2.0, см. `kernel/README.md`).
- `tools/` — запуск: `xnu-cycle.sh` (один прогон: сборка, прошивка `boot_a`, старт XNU, забор лога),
  `boot-xnu.sh`, `chainboot.py`, `system-flash.sh` (образ системы в `xnusys`), `gpt-carve.py`,
  пробы (`gpu-probe.sh`, `touch-probe.sh`, `msdc-test.sh`), читалки логов (`fbshot.py`, `frames.py`,
  `cache-name.py`, `rwm-recipes.py`).
- `configs/` — `x6826b_defconfig` и `x6826b.config`: конфигурация ядра для этого телефона в общем дереве.
- `pull-firmware.sh` — прошивки телефона из **твоего** телефона в `fw-*/` (нужен root в Android).
- `docs/` — как каждая часть нашлась и что показал каждый прогон.
- `device.conf` — телефон и состояние каждого компонента.

У общего дерева в ветке `mtk` ([hakios-xnu-kernel, ветка mtk](https://github.com/oosexgang/hakios-xnu-kernel/tree/mtk))
те же загрузчик и скрипты (`mtk/loader/`, `mtk/tools/`) и эти драйверы под именами `hakios` (`mtk/mt6768/`), и собираются и
запускаются через `make`.

## Чего здесь нет, намеренно

Ни одного файла iOS, ни одной прошивки телефона, ни одного снимка экрана (LEGAL.md). Сборка загрузчика делает два
заголовка, и git игнорирует оба: `touchfw.h` из `hdl_firmware.img` твоего телефона (`pull-firmware.sh`) и `font8x16.h` —
текстовый шрифт загрузчика, растрированный на твоём Mac.

## Запуск

В общем дереве, ветка `mtk` (что нужно:
- Infinix Hot 20 (X6826B) с разлоченным загрузчиком, рутованным Android (Magisk) и сохранённым штатным
  образом `boot_a` (`backup/boot_a_magisk.img` в корне дерева);
- Mac и свой IPSW Darwin userland 16.5 (для `xnuloader/identity.h`)).

```sh
git clone -b mtk https://github.com/oosexgang/hakios-xnu-kernel.git hakios && cd hakios
tools/hakios-device.py sync x6826b      # этот репозиторий, в devices/x6826b
export HAKIOS_SERIAL=...                # твой телефон, как его показывает `adb devices`
devices/x6826b/pull-firmware.sh         # fw-touch/, fw-connsys/
xnuloader/mkidentity.sh your.ipsw
make x6826b_defconfig && make kernel    # hakios-kernel: mali mtk_keys mtk_scp tcm mtk_emmc mtk_connsys
make phone SECS=300 RAMDISK=...         # boot_a прошит, XNU один раз, Android назад, лог в logs/
```

Userland, который работает на этом телефоне (panfrost в Mesa, драйвер Metal), всё ещё использует прежние имена проекта
(`kern.redwood.*`, `RedwoodMali`) и пока не в общем дереве; `RAMDISK=` и образ системы в `xnusys` — его.

Со своими `tools/` этого репозитория, в раскладке дерева, где они писались (`mtkloader/` и `tools/` в его корне, рядом
с `xnuloader/`, `xnu-deps/` и `userland/`):

```sh
./pull-firmware.sh                   # fw-touch/, fw-connsys/ рядом с mtkloader/ этого репозитория
XNU_SECS=300 XNU_ARGS="cpus=4 touchscreen=2 buttons=2 mali=1 mtk_cpufreq=3 metal=1 camtl=1" \
    tools/xnu-cycle.sh                # один прогон; лог ложится в logs/
```

## Лицензии

`kernel/` — APSL 2.0 (`LICENSE.APSL`), как код, встроенный в XNU; всё остальное — MIT (`LICENSE`).
См. `NOTICE` и `LEGAL.md`.
