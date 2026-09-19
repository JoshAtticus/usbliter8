# usbliter8

Тетхерд-эксплойт BootROM для Apple A12, S4/S5 и A13 SoC (A12X/Z теоретически возможны, но не реализованы).

> Скомпилированные **UF2**-образы доступны для [плат RP2350](https://github.com/JoshAtticus/usbliter8/releases/tag/rp2350) и [плат RP2040](https://github.com/JoshAtticus/usbliter8/releases/tag/rp2040). Прошиваются через mass storage protocol BootROM RP2350 или через [picotool](https://github.com/raspberrypi/picotool).

## Использование

### Требования к железу

Эксплойт использует очень низкоуровневую ошибку USB-контроллера. Это значит, что стандартный USB-стек Mac/PC не может до неё добраться. Поэтому используются микроконтроллерные платы на базе Raspberry Pi RP2350.

Плата, которую мы используем: [**Waveshare RP2350 USB-A**](https://www.waveshare.com/wiki/RP2350-USB-A) с кабелем Lightning → USB-A и опционально удалённым резистором R13.

Другие платы на RP2350 также можно использовать, если отрезать кабель Lightning и припаять его напрямую к соответствующим пинам.

Обычно GPIO12 и GPIO13 используются для сигналов D+ и D- соответственно, но это настраивается. **НЕ используйте USB-C кабели** — у них часто совсем другое расположение контактов. И держите оставшийся кабель (с концом Lightning) относительно коротким.

Список плат, на которых мы тестировали эксплойт:

* Waveshare RP2350 USB-A
* Waveshare RP2350 Zero
* Pimoroni TINY2350
* Raspberry Pi Pico 2

RP2040 теоретически тоже можно использовать, но это **НЕ** очень стабильно, и Apple A13 SoC **НЕ** работает вообще.

### Прошивка

Скомпилированные **UF2**-образы доступны для [плат RP2350](https://github.com/JoshAtticus/usbliter8/releases/tag/rp2350) и [плат RP2040](https://github.com/JoshAtticus/usbliter8/releases/tag/rp2040).

Если вашей платы нет среди поддерживаемых, вы можете создать свой файл конфигурации платы (в папке `/boards`) и обратиться к инструкциям по сборке ниже.

### Эксплуатация

1. Войдите в [DFU-режим](https://theapplewiki.com/wiki/DFU_Mode). Сделайте это, пока устройство подключено к Mac/PC.
    * **НЕ** входите в DFU путём прерывания LLB — это не сработает.
2. Отключите устройство от Mac/PC и подключите его к плате RP2350.
3. Через несколько секунд эксплойт завершится.

Есть 2 способа наблюдать за процессом:

1. RP2350 появляется как виртуальный COM-порт — логи эксплойта печатаются там.
2. Через встроенный светодиод.

Если светодиод RGB:

1. Мигает оранжевым — RP2350 загружается (~2 секунды)
2. Горит оранжевым — простой, готов к эксплойту
3. Синий — эксплойт выполняется
4. Зелёный — эксплойт успешен!
5. Красный — эксплойт не удался

Если светодиод одноцветный:

1. Медленное мигание (200 мс) — RP2350 загружается
2. Дыхание — простой, готов
3. Быстрое мигание (100 мс) — эксплойт выполняется
4. Горит — эксплойт успешен!
5. Выключен — эксплойт не удался

Эксплойт занимает 0.7–1.2 секунды. После успеха или неудачи нужно перезагрузить плату — кнопкой, через `picotool` или просто переподключив питание.

### После эксплуатации

Переподключите устройство обратно к Mac/PC.

В серийном номере USB вы увидите строку `PWND` в конце, например:

CPID:8020 CPRV:11 CPFM:03 SCEP:01 BDID:0E ECID:XXXXXXXXXXXXXXXX IBFL:3C SRTG:[iBoot-3865.0.0.4.7] PWND:[usbliter8]

В репозитории есть Python-инструмент управления (`usbliter8ctl`), который позволяет понизить production mode или загрузить сырой iBoot.

Инструмент управления зависит от `pyusb`, который можно получить через **pip**.
➜ usbliter8 git:(main) ✗ ./usbliter8ctl
usage: usbliter8ctl [-h] {boot,demote} ...

Love is Control

positional arguments:
{boot,demote}
boot boot raw iBoot
demote demote production mode

options:
-h, --help show this help message and exit

## Сборка

Если вы хотите скомпилировать прошивку самостоятельно, вам нужны: CMake, [Pico SDK](https://github.com/raspberrypi/pico-sdk), [picotool](https://github.com/raspberrypi/picotool) (зависит от `libusb`) и [ARM toolchain](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads).

Из-за «гоночной» природы эксплойта изменения кода (даже казалось бы несвязанные) могут снизить надёжность. Мы считаем, что это связано с тем, что RP2350 выполняет код из внешней QSPI-флеш-памяти. Реальные инструкции могут попадать в небольшой кэш, и затем вытеснения из этого кэша ломают тайминги, и всё ломается.

Вот точные версии Pico SDK и ARM toolchain, которые мы использовали:

* Pico SDK — SDK Release 2.2.0 (commit `a1438dff1d38bd9c65dbd693f0e5db4b9ae91779`)
* ARM toolchain — 15.2.Rel1

cd usbliter8
cmake -S . -B build -DPICO_BOARD=
P
I
C
O
B
O
A
R
D
−
D
P
I
C
O
S
D
K
P
A
T
H
=
PICO 
B
​
 OARD−DPICO 
S
​
 DK 
P
​
 ATH=PICO_SDK_PATH -DPICO_TOOLCHAIN_PATH=$PICO_TOOLCHAIN_PATH
cmake --build build

Замените `$PICO_SDK_PATH` и `$PICO_TOOLCHAIN_PATH` на ваши пути. `$PICO_BOARD` — один из следующих:

* waveshare_rp2350_usb_a
* waveshare_rp2350_zero
* pimoroni_tiny2350
* pico2

Хотя использование RP2040 не рекомендуется, вы всё ещё можете попробовать:

* adafruit_feather_rp2040
* pico

Вы также можете добавить свой файл определения платы в каталог `/board`. Имя платы должно совпадать с именем платы в Pico SDK.

Или используйте «unknown» (выбирается по умолчанию для неподдерживаемых плат). Он использует GPIO12/13 для сигналов USB и отключает светодиод.

## Авторы

* [@__gsch](https://x.com/__gsch) из Paradigm Shift — баг и эксплуатация
* [@hdesk@infosec.exchange](https://infosec.exchange/@hdesk/) из Paradigm Shift — эксплуатация
* **REDACTED** из Paradigm Shift — пост-эксплуатация и разработка

## Благодарности

* **sekigon-gonnoc** — за [PIO USB library](https://github.com/sekigon-gonnoc/Pico-PIO-USB)
