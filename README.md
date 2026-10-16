# Quadro P3000 (MXM) в iMac 27" Mid 2010: как включить внутренний экран в Windows

Проверено на iMac11,3 (27" Mid 2010) с Quadro P3000 Mobile (GP104, 10de:1bb8). Windows установлена в режиме UEFI, OpenCore 1.0.8. macOS и OCLP нет. Одинаковое поведение было на драйверах R470 и R580.

## Что нужно заранее

Карта должна быть прошита vBIOS для iMac с драйвером GOP и включённой внутренней панелью. Оригинальный vBIOS ноутбука эту панель не зажигает.

Загрузчик OpenCore установлен на iMaс или рабочая загрузочная флешка с ним.

С таким vBIOS и двумя параметрами OpenCore ниже внутренний экран работает в меню OpenCore и в Windows до загрузки драйвера NVIDIA.

### vBIOS

vBIOS прошивается аппаратным программатором. Образы прошивок и описание процесса прошивок смотреть в постах на MacRumors:

**Santa's Little Helper** - https://forums.macrumors.com/threads/imac-2011-maxwell-and-pascal-gpu-upgrade.2300989/page-7?post=30349621#post-30349621

**m0bil** - https://forums.macrumors.com/threads/imac-2011-maxwell-and-pascal-gpu-upgrade.2300989/page-9?post=30638999#post-30638999

### OpenCore

Windows нужно загружать через OpenCore. Прямая загрузка Windows не применяет правку ACPI, и драйвер снова выключит панель.

В `UEFI → Quirks`:

- `ForgeUefiSupport` = `True`
- `ReloadOptionRoms` = `True`

Эти два параметра заставляют прошивку 2010 года выполнить GOP с карты, поэтому панель будет включена ещё до операционной системы.

## Сама правка ACPI

После загрузки драйвера NVIDIA внутренняя панель гаснет, работает только внешний монитор. vBIOS и разводка при этом исправны. Драйвер отключает панель из-за ACPI этого iMac.

В заводском DSDT видеокарта описана как старый Radeon. Метод `GFX0._DOD` возвращает один идентификатор дисплея, `0x0110`. Драйвер NVIDIA сверяет с этим списком, какой выход считать внутренней панелью, не находит среди них свой eDP и отключает панель.

Нужно переименовать `_DOD` в `XDOD`. Драйвер больше не увидит метод и определит панель сам. Устройство `LCD` с методами яркости (`_BCL`, `_BCM`, `_BQC`) остаётся на месте. На этом iMac яркостью управляет SMC, а не видеокарта.

Добавьте запись в `ACPI → Patch`:

```xml
<dict>
    <key>Base</key>
    <string></string>
    <key>BaseSkip</key>
    <integer>0</integer>
    <key>Comment</key>
    <string>GFX0 _DOD to XDOD</string>
    <key>Count</key>
    <integer>1</integer>
    <key>Enabled</key>
    <true/>
    <key>Find</key>
    <data>FA1fRE9EAA==</data>
    <key>Limit</key>
    <integer>0</integer>
    <key>Mask</key>
    <data></data>
    <key>OemTableId</key>
    <data></data>
    <key>Replace</key>
    <data>FA1YRE9EAA==</data>
    <key>ReplaceMask</key>
    <data></data>
    <key>Skip</key>
    <integer>0</integer>
    <key>TableLength</key>
    <integer>0</integer>
    <key>TableSignature</key>
    <data>RFNEVA==</data>
</dict>
```

`Find` — это байты `14 0D 5F 44 4F 44 00`, заголовок AML метода `Method (_DOD, 0)`. `Replace` меняет имя на `XDOD`. `TableSignature` ограничивает правку таблицей DSDT. В заводской прошивке iMac11,3 такая последовательность встречается один раз.

## После правки

- Внутренняя панель 2560×1440 остаётся включённой с драйвером NVIDIA.
- Ползунок яркости работает.
- Изображение возвращается после сна.
- Внешний монитор работает одновременно с внутренним.

Отдельный SSDT и дополнительные kext не нужны.
