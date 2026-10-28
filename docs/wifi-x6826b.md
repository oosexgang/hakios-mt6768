# Wi-Fi (и Bluetooth) на Infinix Hot 20 (X6826B, MT6768): что переносится, что писать заново

План, а не описание готового: на 27 октября 2026 года в ядре X6826B нет ни одной строки драйвера радио
MediaTek. Здесь записано, где проходит граница, что из Wi-Fi ветки `main` (Qualcomm) встаёт без правок, что
придётся написать и что нужно знать до первой строки кода. Всё, что помечено **проверить**, — не установлено,
а предположено, и должно быть сверено с исходниками производителя или с телефоном.

## 1. Что берётся из ветки `main` без изменений

В ветке `main` Wi-Fi разделён на две половины, и граница между ними не знает, какое под ней радио.

Сверху (до iOS) — общее для любого телефона:

| файл | что делает |
| --- | --- |
| `bsd/dev/redwood_apple80211.c` | отвечает службе Wi-Fi iOS на запросы Apple80211 (сканирование, сети, подключение) |
| `bsd/dev/redwood_sta.c`, `bsd/sys/redwood_sta.h` | один поток, на котором делается каждый вызов в радио, по одному за раз; список того, что услышало последнее сканирование; подключение к сети из файла при старте |
| `bsd/sys/redwood_net.h` (+ реализация) | кадры: сетевой интерфейс XNU, «связь есть/нет» |
| `wifi/lx/src/lx_wpa.c` | рукопожатие WPA2 с паролем; проверяется на компьютере (`tools/wpa-test/run.sh`) |

Снизу в `main` — ath11k из Linux (WCN6750), собранный на заголовках LinuxKPI из FreeBSD (`wifi/lx/`,
`bsd/dev/redwood_lx.c`, `tools/mk-ath11k.sh`), и наша станция над ним (`wifi/lx/src/lx_80211.c`). Это
**не переносится**: другое железо.

## 2. Что должен дать новый драйвер радио

Ровно то, что даёт станция в `main`, — и больше ничего: верхняя половина уже есть.

```c
/* bsd/sys/redwood_sta.h */
struct redwood_sta_ops {
	int  (*power)(bool on);                                   /* включить/выключить радио */
	int  (*scan)(const struct redwood_sta_scan *what);
	void (*heard)(void (*each)(const struct redwood_sta_bss *, void *), void *);
	int  (*join)(const struct redwood_sta_join *what);        /* открытая сеть или WPA2 с паролем */
	int  (*leave)(void);
	void (*state)(struct redwood_sta_state *state);           /* не спит, с любого потока */
};
redwood_sta_attach(&ops);           /* когда радио готово */
redwood_sta_lost(error);            /* сеть пропала сама */

/* bsd/sys/redwood_net.h */
struct redwood_net_link { bool (*transmit)(mbuf_t, size_t); unsigned (*poll)(void); ... };
redwood_net_attach(&link); redwood_net_input(frame, length); redwood_net_link_state(up);
```

Прошивка MediaTek сама делает сканирование, подключение и ключи (gen4m — драйвер «полного MAC», см. 4),
поэтому ни станция `lx_80211.c`, ни рукопожатие `lx_wpa.c` здесь не нужны: `scan` и `join` — команды прошивке.

## 3. Железо X6826B

- Wi-Fi и Bluetooth (и GPS, FM) — подсистема **CONNSYS** внутри MT6768 со своим процессором и радиочастью
  **MT6631**. На Android её поднимают модули `wmt_drv.ko` (питание, загрузка прошивки, общий канал),
  `wmt_chrdev_wifi.ko` и `wlan_drv_gen4m.ko` (Wi-Fi), лежащие в `/vendor/lib/modules` телефона, с прошивками
  из `/vendor/firmware` (точный список — ниже, снят с телефона 28 октября).
- Драйвера CONNSYS нет в основном дереве Linux (mt76 — для отдельных чипов MediaTek, не для встроенных в
  SoC); есть только исходники производителя под GPL в ядрах устройств на MT6768. **Проверить**, выложил ли
  Transsion исходники ядра X6826B; если нет — взять ядро другого телефона на MT6768 с тем же CONNSYS.

### Исходники (скачаны 27 октября 2026)

`xnu-deps/build/connsys-src/` (в `.gitignore`): `connectivity/` ветки `merlin-r-oss` репозитория
[MiCode/MTK_kernel_modules](https://github.com/MiCode/MTK_kernel_modules) — модули ядра MediaTek, которые
Xiaomi выложила для Redmi 9 / POCO M2 (Helio G80/G85, платформа MT6768, как у X6826B), коммит `ffe6bc5`
от 6 июля 2021, 48 МБ, лицензия в репозитории не указана. В ней:

| путь | что это |
| --- | --- |
| `common/common_main/platform/mt6768.c`, `include/mt6768.h` | WMT под MT6768: питание, такты, EMI, загрузка ROM patch |
| `conninfra/` | общая инфраструктура CONNSYS |
| `wlan/core/gen4m/` | драйвер Wi-Fi; `os/linux/plat/mt6768/plat_priv.c`, шина `os/linux/hif/axi` |
| `bt/mt66xx/` | Bluetooth для этого семейства |

## 4. Что выяснено и что ещё надо выяснить

Из исходников (27 октября):

- **Включение CONNSYS** (`mt6768.c`): `consys_hw_power_ctrl` (домен питания в SPM), `consys_clock_buffer_ctrl`,
  `consys_ahb_clock_ctrl`, `consys_hw_reset_bit_set`, `consys_emi_mpu_set_region_protection` (область EMI под
  прошивку), `polling_consys_chipid` (признак, что подсистема ожила), `consys_acr_reg_setting`,
  `consys_afe_reg_setting`, затем `consys_dl_rom_patch`. Адреса: `CONN_HIF_ON` 0x18007000, AFE 0x180B3000,
  такты 0x180C1200, EMI у процессора 0x80068000 (у прошивки 0xF0068000), выводы 0x10005000 и 0x10420000.
  Часть берётся из дерева устройств (`consys_read_reg_from_dts`, `consys_clk_get_from_dts`,
  `consys_pmic_get_from_dts`) — её надо снять с DTB телефона.
- **Прошивка**: ROM patch с суффиксом `_soc1_0` (`mt66xx_patch_hdr.bin`), чип — «Talbot SoC 1.0» с MT6631.
  Точные имена файлов — из `/vendor/firmware` телефона.
- **Wi-Fi**: gen4m, шина AXI; это драйвер «полного MAC» — `mtk_cfg80211_scan`, `mtk_cfg80211_connect`,
  `mtk_cfg80211_add_key` отдают работу прошивке.

Из дерева устройств Android (27 октября; DTB из `backup/boot_a_magisk.img`, тот же, что LK отдаёт загрузчику;
`mtkloader/mtkconnsys.h` печатает его при каждой загрузке XNU). Узел `consys@18002000`, `compatible =
"mediatek,mt6768-consys"`, блоки `reg` по индексам `enum CONSYS_BASE_ADDRESS_INDEX`
(`common_main/platform/include/mtk_wcn_consys_hw.h`):

| индекс | блок | адрес | размер |
| --- | --- | --- | --- |
| 0 | MCU (конфигурация процессора CONNSYS; `CONSYS_IP_VER` по смещению 0x10) | 0x18002000 | 0x1000 |
| 1 | TOP_RGU (сброс) | 0x10007000 | 0x100 |
| 2 | INFRACFG_AO | 0x10001000 | 0x1000 |
| 3 | SPM (питание) | 0x10006000 | 0x1000 |
| 4 | MCU_CONN_HIF_ON | 0x18007000 | 0x1000 |
| 5 | MCU_TOP_MISC_OFF | 0x180B1000 | 0x1000 |
| 6 | MCU_CFG_ON | 0x180A3000 | 0x1000 |
| 7 | MCU_CIRQ | 0x180A5000 | 0x800 |
| 8 | MCU_TOP_MISC_ON | 0x180C1000 | 0x1000 |
| 9 | MCU_CONN_HIF_PDMA | 0x18004000 | 0x1000 |

Прерывания (GIC SPI): 0x11C (284), 0x4E (78), 0x11F (287). Узел `wifi@18000000` (`mediatek,wifi`): регистры
0x18000000, 1 МБ, прерывание SPI 0x11D (285). Зарезервированная память: `consys-reserve-memory` 4 МБ и
`wifi-reserve-memory` (shared-dma-pool) 3 МБ, обе где угодно в 0x40000000–0xC0000000 — ядро Linux выбирает
место само, так что XNU должен выделить их сам. Регулятор PMIC: `mt6358regulator/ldo_vcn33_wifi`.

Ещё надо выяснить:

1. **Такты и регуляторы**: какие такты и выводы PMIC (VCN18, VCN28, VCN33) включает `mt6768.c`
   (`consys_hw_vcn18_ctrl` и соседние) — по исходникам и по MT6358 (`mtkloader/mtkpwrap.h` уже его читает).
2. **Файлы прошивки и порядок загрузки**. На телефоне (27 октября, `ls` из adb) они такие:
   `soc1_0_patch_mcu_1a_1_hdr.bin`, `soc1_0_ram_mcu_1a_1_hdr.bin`, `soc1_0_ram_wifi_1a_1_hdr.bin`,
   `soc1_0_ram_bt_1a_1_hdr.bin`, `WIFI_RAM_CODE_soc1_0_1a_1.bin`, `WMT_SOC.cfg`, `wifi.cfg`, `BT_FW.cfg`, модули
   `wmt_drv.ko`, `wmt_chrdev_wifi.ko`, `wlan_drv_gen4m.ko`, `bt_drv_connac1x.ko`, `connfem.ko`. Снять их через
   `adb pull` не вышло: SELinux не даёт пользователю shell читать `/vendor/firmware` и `/vendor/lib/modules`
   (нужен root или раздел vendor из прошивки производителя). Ещё не выяснено, формат заголовка ROM patch
   (`consys_dl_rom_patch`), загружает ли RAM code WMT или сам gen4m по AXI.
3. **Канал команд gen4m по AXI** (`wlan/core/gen4m/os/linux/hif/axi`, `nic/`): формат команд и событий,
   кольца приёма и передачи — то, что заменит здесь шину Linux.

### Рецепт шага 2 (включение), по исходникам

Порядок общего кода WMT (`mtk_wcn_consys_hw_reg_ctrl`, `platform/mtk_wcn_consys_hw.c`), шаги платформы из
`mt6768.c`:

1. **VCN18** вкл. (`consys_hw_vcn18_ctrl`; регулятор PMIC MT6358).
2. **Выводы** (`consys_set_if_pinmux`): 0x10005000+0x3F0 (маска 0xFFFFFF88, значение 0x11), +0x3E0 (маска
   0x8888888F, значение 0x11111110); сила выводов 0x10420000 (маска 0xF8FFFFFF).
3. **Буфер тактового сигнала** (`consys_clock_buffer_ctrl`) или режим VCN28 — в зависимости от типа
   «co-clock»; затем **VCN28** вкл.
4. **Сброс** (`consys_hw_reset_bit_set`): TOP_RGU 0x10007000+0x18.
5. **Питание** (`consys_hw_power_ctrl`). На MT6768 Android делает это через домен питания Linux
   (`clk_scp_conn_main`, `CONSYS_PWR_ON_OFF_API_AVAILABLE = 1`); ручной вариант того же в `#else`:
   SPM+0x000 = (старое & 0xFFFF) | 0x0B160001 (ключ записи); SPM+0x32C |= бит 2, ждать SPM+0x180 бит 1;
   |= бит 3, ждать SPM+0x184 бит 1; &= ~бит 4 (такт шины), 1 мкс; &= ~бит 1 (изоляция); ждать бит 30 в
   0x180 и 0x184; снять программный сброс TOP_RGU+0x18; снять защиту шины INFRACFG_AO 0x10001000+0x220
   (биты 13, 14, состояние +0x228) и +0x250 (бит 18, состояние +0x258); SPM+0x6F8 = 0x5, SPM+0x6FC = 0x50505.
   В исходнике ожидания бесконечные — у нас каждое с таймаутом и отчётом.
6. **Такт AHB** (`consys_ahb_clock_ctrl`).
7. **Chip ID**: 0x180B1000+0x10 должно стать 0x10020501 — это и есть признак, что CONNSYS ожил и шаг 2
   удался. В исходнике сразу за ним идёт загрузка ROM patch (`consys_dl_rom_patch`) — это уже шаг 3, и для
   него нужна прошивка.

Для шагов 1 и 3 нужны регистры LDO VCN18/VCN28/VCN33 MT6358 (из `mt6358-regulator` Linux); MT6358 мы уже
читаем через PWRAP (`mtkloader/mtkpwrap.h`, кнопки в `redwood_mtkkeys.c`).

## 5. Порядок работы

Каждый шаг включается своим boot-arg и ничего не трогает без него, как `wifi_step` в `main`.

1. Только чтение: CONNSYS после LK (включён ли, что в его регистрах), зарезервированная память.
2. Включение CONNSYS и загрузка прошивки; признак — прошивка ответила.
3. Канал команд: одна безвредная команда (версия прошивки) и её ответ.
4. Сканирование → `redwood_sta_ops.scan/heard`: сети видны в `kern.redwood`/логе и в Настройках iOS.
5. Подключение к открытой сети, затем WPA2; кадры через `redwood_net`.

Шаги 1–3 — работа «загрузки и ядра», шаги 4–5 подключают уже готовую верхнюю половину.

## 6. Что нужно от пользователя

- ~~Разрешение скачать исходники~~ — получено, скачаны (см. 3).
- Разрешение снять с телефона (в Android, когда он не занят прогоном) список и сами файлы
  `/vendor/lib/modules/{wmt,wlan}*.ko` и `/vendor/firmware` — только чтение.
- Разрешение на прогоны на телефоне, начиная с шага 1. Без него — только чтение исходников и сборка.

## 7. Bluetooth

В проекте Bluetooth нет совсем — ни в `main`, ни здесь. На X6826B он в том же CONNSYS, что и Wi-Fi, так что
шаги 1–2 общие. Сверху нужна вторая половина, которой ещё нет нигде: транспорт HCI, которого ждёт
`bluetoothd` iOS. Это отдельная работа после Wi-Fi.

## Прошивка и модули сняты с телефона (28 октября 2026)

Через root (Magisk, `su`), только чтение: файлы скопированы из `/vendor/firmware` и
`/vendor/lib/modules` во временную папку, забраны `adb pull`, временная папка удалена. Лежат в
`xnu-deps/build/connsys-fw/` (в .gitignore: это прошивка производителя), контрольные суммы в
`SHA256SUMS` там же.

- firmware: `soc1_0_patch_mcu_1a_1_hdr.bin` (25 710), `soc1_0_ram_mcu_1a_1_hdr.bin` (105 648),
  `soc1_0_ram_wifi_1a_1_hdr.bin` (197 628), `soc1_0_ram_bt_1a_1_hdr.bin` (571 025),
  `WIFI_RAM_CODE_soc1_0_1a_1.bin` (638 904), `WMT_SOC.cfg`, `wifi.cfg`, `BT_FW.cfg`.
- modules: `wmt_drv.ko`, `wmt_chrdev_wifi.ko`, `wlan_drv_gen4m.ko`, `bt_drv_connac1x.ko`, `connfem.ko`.

## CONNSYS включён из XNU (28 октября 2026)

`bsd/dev/redwood_connsys.c`, boot-arg `connsys_step`:

| Шаг | Что | Прогон |
| --- | --- | --- |
| 2 | только чтение: VCN18, VCN28, VCN33 выключены, VCN33 выставлен на 3,5 В | 304 |
| 3 | VCN18 и VCN33 в программный режим, VCN33 на 3,3 В, все три включены (через PWRAP) | 305 |
| 4 | выводы, сброс CPU, домен питания SPM по последовательности Linux (`spm_mtcmos_ctrl_conn_pwr`: PWR_ON, PWR_ON_2ND, ждать бит 1, CLK_DIS и ISO снять, **PWR_RST_B**), снятие защиты шины через регистры CLR (`INFRA+0x2A4`, `+0x2AC`), SPM apsrc/ddr | 306, 307 |

Прогон 307: идентификатор CONNSYS `0x10020501`, это MT6768, чип жив. Процессор CONNSYS ещё в сбросе
(CPUPCR 0), как и должно быть до прошивки.

Отличия от ручной ветки WMT: она не снимает PWR_RST_B (бит 0 `TOP1_PWR_CTRL`), и без этого вторая
часть не подтверждается. А подтверждение второй части на этом телефоне приходит битом 31 `PWR_STATUS`,
а не 30, как в заголовке WMT. Буфер тактового сигнала (co-clock, `clk_buf`) ещё не включён.

Дальше: загрузка ROM patch (`soc1_0_patch_mcu_1a_1_hdr.bin`) и RAM-прошивки MCU (`consys_dl_rom_patch`,
`wmt_step`), затем канал команд WMT и Wi-Fi gen4m.

## Прошивка загружена, процессор CONNSYS работает (28 октября 2026)

Шаг 5 (`connsys_step=5`), в потоке ядра после монтирования корня: файлы `soc1_0_*_hdr.bin` из системного
образа (`/fw/connsys`, кладёт `userland/build.sh` из `xnu-deps/build/connsys-fw`) копируются без 48-байтного
заголовка `struct wmt_rom_patch` в общую память CONNSYS по их `u4PatchAddr` (0xF0000000 у CONNSYS = начало
памяти): MCU RAM на 0, ROM patch на 0x1C000, BT на 0x80000, Wi-Fi на 0x140000. Память — 4 МБ на
`0x51000000` (выравнивание 16 МБ для регистра переадресации). Затем `INFRA+0x380` и `+0x388`
(переадресация), сброс процессора CONNSYS снимается (`TOP_RGU+0x18` бит 12).

Прогон 312 (через `kern.redwood.connsys`): 4 файла, `MCU+0x600 = 0x1D1E` — ROM CONNSYS принял патчи, как ждёт
Android; CPUPCR меняется (0x26E2, 0xB04) — процессор CONNSYS исполняет код. Вместе с акселерометром и
ориентацией, без паник.

По дороге: `ml_io_map` 4 МБ разом не помещается в карту ввода-вывода ядра (прогон 308), `ml_io_unmap`
паникует на постоянных записях (310); окна по файлу (~900 КБ) и буфер датчиков 256 КБ помещаются вместе.

Дальше: канал команд WMT с работающим CONNSYS (у SoC — через общую память и прерывания, `wmt_step`/STP по
BTIF), затем Wi-Fi gen4m (`WIFI_RAM_CODE_soc1_0_1a_1.bin`) и Bluetooth.

## Следующий слой: канал команд к CONNSYS (BTIF, STP, WMT)

После загрузки патчей через память Android говорит с CONNSYS командами WMT, обёрнутыми в протокол STP поверх
аппаратного моста BTIF (`mtk_wcn_soc_sw_init` в `core/wmt_ic_soc.c`: полный режим BTIF, досылка патчей
через STP, команда сброса, таблицы инициализации, затем включение функций Wi-Fi/BT).

Из дерева устройств (прогон 313, отчёт загрузчика): `btif@1100c000`, `mediatek,btif`; регистры BTIF
0x1100C000 (0x1000), DMA передачи 0x11000B80, DMA приёма 0x11000C00 (по 0x80); прерывания SPI 0x85, 0x73,
0x8D. BTIF устроен как UART (`drivers/misc/mediatek/btif/common/plat_inc/btif_priv.h`): RBR/THR +0x0, IER
+0x4, IIR +0x8, LSR +0x14 (бит 0 — данные есть), SLEEP_EN +0x48, DMA_EN +0x4C, TRI_LVL +0x60, WAK +0x64,
HANDSHAKE +0x6C.

План:
1. BTIF в режиме PIO из ядра: такт BTIF в INFRA, чтение того, что CONNSYS пишет после старта.
2. STP: кадр (синхробайт, номер, подтверждение, тип, длина, CRC), подтверждения, повторы.
3. WMT: досылка патчей, сброс, таблицы инициализации, включение функции Wi-Fi — по `wmt_ic_soc.c`.
4. Wi-Fi gen4m (`wlan_drv_gen4m`, FullMAC, `WIFI_RAM_CODE_soc1_0_1a_1.bin`) под XNU и `redwood_sta_ops`;
   Bluetooth — HCI поверх того же STP для bluetoothd.

### BTIF и первая команда WMT (шаги 6–7)

- Шаг 6 (`connsys_step=6`, прогон 314): такт BTIF (IFR2, бит 31) и настройка как в `hal_btif_hw_init` до
  включения CONNSYS. После старта CONNSYS сам ничего не шлёт (LSR 0x60): первым говорит WMT.
- Шаг 7 (`connsys_step=7`, прогоны 315–316): то, что WMT спрашивает первым (`wmt_core_hw_check`), —
  чтение регистра 0x80000008 (GEN_HCR). Кадр STP «обязательного» режима (`MTKSTP_BTIF_MAND_MODE`):
  `80 40 14 00`, 20 байт `WMT_SET_REG_CMD` с op 2, CRC `00 00`; перед ним — импульс пробуждения BTIF_WAK
  (+0x64). **CONNSYS ответил** (прогон 316):

  ```
  80 40 10 00 | 02 08 0c 00 00 00 00 01 08 00 00 80 32 66 00 00 | 00 00
  ```

  Кадр STP (WMT, 16 байт), событие WMT 0x08 со статусом 0, адрес 0x80000008, значение 0x6632 (на SoC
  WMT потом подменяет его чип-ID платформы). Канал команд к CONNSYS работает.

CONNSYS отвечает на первую команду не всегда сразу: в прогоне 316 сразу, в 317 только после второго
пробуждения, в 318 не ответил за 3 с. Поэтому первая команда посылается до 5 раз, каждый раз с импульсом BTIF_WAK.

### Полный режим STP (шаг 8, прогон 319)

`connsys_step=8` повторяет `_wmt_soc_mode_ctrl` целиком — всё ответило как ожидалось:

```
WMT read 0x80000008:     02 08 0c 00 00 00 00 01 08 00 00 80 32 66 00 00
STP defaults:            02 04 06 00 00 04 11 00 00 00
STP set:                 02 04 02 00 00 03
(хост — в полный режим)
STP asked again: [ack 0] 02 04 06 00 00 04 db 0e 68 01
```

В полном режиме CONNSYS подтверждает наш кадр (`[ack 0]` — кадр без данных), а его ответ проходит с
контрольной суммой заголовка и CRC-16. STP работает в том же режиме, что на Android.

Было в плане: `init_table_1_2` (запрос STP по умолчанию, `01 04 01 00 04`),
`init_table_4` (`01 04 05 00 03 DB 0E 68 01`), переход хоста в полный режим STP (номер, подтверждение,
контрольная сумма заголовка, CRC16 данных, ACK-кадры `0x80+ack, 0, 0, sum`), `init_table_5`; затем
таблицы `init_table_3`, функция Wi-Fi.

Дальше: досылка патчей через STP (если на MT6768 она нужна при загрузке в память — `pda_dl_patch_flag`),
`init_table_3` (`WMT_RESET_CMD` `01 07 01 00 04`), таблицы инициализации и включение функции Wi-Fi
(`WMT_FUNC_CTRL`), затем драйвер gen4m.
