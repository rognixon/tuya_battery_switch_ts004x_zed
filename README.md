## IHSENO 6-Button Scene Switch (`TS0046` / PCB `6KEY-V04` 2024.05.21)

This variant uses a **Tuya ZT3L** (Telink TLSR8258) module with a different GPIO pinout than the standard `_TZ3000_iszegwpd` (`model_6`) build.

### 1. Hardware GPIO Pinout (`ZT3L` Module)

| Function | ZT3L Module Pad | Telink TLSR8258 GPIO | Notes |
| :--- | :--- | :--- | :--- |
| **Status LED** | `C4` (Pin 2) | `GPIO_PC4` | Active-High (drives transistor `Q1` for all 6 LEDs) |
| **Button 1** | `B5` (Pin 14) | `GPIO_PB5` | Active-Low (`PM_PIN_PULLUP_1M`) |
| **Button 2** | `A0` (Pin 12) | `GPIO_PA0` | Active-Low (`PM_PIN_PULLUP_1M`) |
| **Button 3** | `C2` (Pin 7) | `GPIO_PC2` | Active-Low (`PM_PIN_PULLUP_1M`) |
| **Button 4** | `D2` (Pin 5) | `GPIO_PD2` | Active-Low (`PM_PIN_PULLUP_1M`) |
| **Button 5** | `C3` (Pin 6) | `GPIO_PC3` | Active-Low (`PM_PIN_PULLUP_1M`) |
| **Button 6** | `D7` (Pin 4) | `GPIO_PD7` | Active-Low (`PM_PIN_PULLUP_1M`) |

---

### 2. Source Code Changes (`src/common/board_ts004x.h`)

Update the LED and Button GPIO definitions in `src/common/board_ts004x.h` as follows (note that 6-button switches use `BUTTON4_6_GPIO` for Button 4, while `BUTTON4_GPIO` is moved to `GPIO_PB4` to avoid colliding with Button 5 on `GPIO_PC3`):

```c
/***************************** Configure LED  ***************************************/
#define LED_ON                  1
#define LED_OFF                 0
#define LED1                    GPIO_PC4
#define PC4_FUNC                AS_GPIO
#define PC4_OUTPUT_ENABLE       ON
#define PC4_INPUT_ENABLE        OFF
#define PC4_DATA_OUT            LED_OFF

/************************* Configure BUTTON GPIO ***************************************/
#define MAX_BUTTON_NUM          6

#define BUTTON1_GPIO            GPIO_PB5
#define PB5_INPUT_ENABLE        ON
#define PB5_OUTPUT_ENABLE       OFF
#define PB5_FUNC                AS_GPIO
#define PULL_WAKEUP_SRC_PB5     PM_PIN_PULLUP_1M

#define BUTTON2_GPIO            GPIO_PA0
#define PA0_INPUT_ENABLE        ON
#define PA0_OUTPUT_ENABLE       OFF
#define PA0_FUNC                AS_GPIO
#define PULL_WAKEUP_SRC_PA0     PM_PIN_PULLUP_1M

#define BUTTON3_GPIO            GPIO_PC2
#define PC2_INPUT_ENABLE        ON
#define PC2_OUTPUT_ENABLE       OFF
#define PC2_FUNC                AS_GPIO
#define PULL_WAKEUP_SRC_PC2     PM_PIN_PULLUP_1M

/* Unused on 6-key model (moved to PB4 to avoid conflict with BUTTON5 on PC3) */
#define BUTTON4_GPIO            GPIO_PB4
#define PB4_INPUT_ENABLE        ON
#define PB4_OUTPUT_ENABLE       OFF
#define PB4_FUNC                AS_GPIO
#define PULL_WAKEUP_SRC_PB4     PM_PIN_PULLUP_1M

/* Key 4 (6-button model) */
#define BUTTON4_6_GPIO          GPIO_PD2
#define PD2_INPUT_ENABLE        ON
#define PD2_OUTPUT_ENABLE       OFF
#define PD2_FUNC                AS_GPIO
#define PULL_WAKEUP_SRC_PD2     PM_PIN_PULLUP_1M

/* Key 5 */
#define BUTTON5_GPIO            GPIO_PC3
#define PC3_INPUT_ENABLE        ON
#define PC3_OUTPUT_ENABLE       OFF
#define PC3_FUNC                AS_GPIO
#define PULL_WAKEUP_SRC_PC3     PM_PIN_PULLUP_1M

/* Key 6 */
#define BUTTON6_GPIO            GPIO_PD7
#define PD7_INPUT_ENABLE        ON
#define PD7_OUTPUT_ENABLE       OFF
#define PD7_FUNC                AS_GPIO
#define PULL_WAKEUP_SRC_PD7     PM_PIN_PULLUP_1M
```

---

### 3. Compiling via GitHub Actions (`.github/workflows/build.yml`)

To compile automatically in the cloud using Telink's official TC32 container without installing a local toolchain, add `.github/workflows/build.yml`:

```yaml
name: Build TS0046 Custom Firmware

on:
  workflow_dispatch:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    container:
      image: ghcr.io/telink-semi/telink-dev-tc32:latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Link Telink TC32 toolchain & compile Model 6
        run: |
          apt-get update && apt-get install -y python3
          git config --global --add safe.directory "\$GITHUB_WORKSPACE"
          GCC_PATH=\$(which tc32-elf-gcc || find / -name tc32-elf-gcc 2>/dev/null | head -n 1)
          mkdir -p /opt/tc32
          ln -sfn "(dirname "GCC_PATH")" /opt/tc32/bin
          mkdir -p out
          make clean
          make PROJECT_MODEL=_model_6 DEBUG="-DUART_PRINTF_MODE=OFF"

      - name: Upload compiled firmware
        uses: actions/upload-artifact@v4
        with:
          name: ts0046-ihseno-6key-v04
          path: |
            bin/*.bin
            bin/*.zigbee
```

Download the **`ts0046-ihseno-6key-v04`** artifact from the **Actions** tab after each build.

---

### 4. Flashing & Zigbee2MQTT Setup

1. **Initial Flash (CP2102 USB-to-TTL via Web Flasher):**
   * Remove the battery from the switch and wire your **CP2102** adapter to the ZT3L module pads:
     * `3V3` -> `3V3` (Pin 8) ideally via a switch so it can be booted during activation
     * `GND` -> `GND` (Pin 9)
     * `TXD` -> `SWS` (bottom edge pad)
   * Open [pvvx's TLSR825x USB-COM Flash Writer](https://github.io) in a WebSerial-capable browser (Chrome/Edge).
   * Click **Open**, select your CP2102 COM port, select the compiled `bin/tuya_battery_switch_ts004x_zed_model_6_V*.bin` file, set baud as 115200 and activation time of 1 second.  Click **Write to Flash** (toggle/switch-on `3V3` power during the 1sec activation so the chip enters SWS mode).
   * *(Future firmware updates can be pushed wirelessly via Zigbee2MQTT OTA using the `.zigbee` file.)*
2. **Pairing:** Press any button **5 times quickly**, then **press and hold on the 6th press for ~2 seconds** until the LEDs flash steadily to join Zigbee2MQTT (recognized as **`TS0046-z-SlD`**).
3. **Configure Button Actions:**
   * Press any button **4 times quickly** to wake the sleepy device for 20 seconds.
   * In Z2M **Exposes**, set **`Switch type`** on Endpoints `1`–`6` to **`multifunction`** (enables `single`, `double`, `triple`, `hold`, and `release`).

<img width="627" height="354" alt="image" src="https://github.com/user-attachments/assets/519bbc86-dac4-44ca-9c3a-b21d91ffd167" />



# <a id="Top">LoraTap Wireless switch Zigbee with custom firmware</a>

<img src="doc/images/LoraTap_TS004x.jpg"/>

## [Ссылка на магазин](https://ali.click/09xe01e)

---

> [!WARNING]
> Описание сделано на примере 4-х кнопочного пульта.
>
> **Автор не несет никакой ответственности, если вы, воспользовавшись этим проектом, превратите свой умный пульт (выключатель) в полоумный.**


<!--
| Custom Zigbee Model | Original Zigbee Model | Z2M Model | Original Zigbee Manufacturer| Update method | Photo    |
|:-------------------:|:---------------------:|:---------:|:---------------------------:|:-------------:|:--------:|
| TS0044-z-SlD        | TS0044                | TS0043    | _TZ3000_ee8nrt2l            | OTA file      | :camera: |
| TS0044-z-SlD        | TS0044                | TS0043    | _TZ3000_ee8nrt2l            | OTA file      | :camera: |
-->

| Custom Zigbee Model | Original Zigbee Model | Z2M Model | Original Zigbee Manufacturer| Update method | Photo    |
|:-------------------:|:---------------------:|:---------:|:---------------------------:|:-------------:|:--------:|
| TS0041-z-SlD | TS0041 | TS0041 | --- | OTA file | [:camera:](doc/images/TS0041-z-SlD.png) |
| TS0042-z-SlD | TS0042 | [TS0042](https://www.zigbee2mqtt.io/devices/TS0042.html) | _TZ3000_fkvaniuu | [OTA file](https://github.com/slacky1965/tuya_battery_switch_ts004x_zed/raw/refs/heads/main/bin/1141-d3a3-1111114b-tuya_battery_switch_ts004x_zed_model_2.zigbee) | [:camera:](doc/TS0042.md) |
| TS0043-z-SlD | TS0043 | [TS0043](https://www.zigbee2mqtt.io/devices/TS0043.html) | _TZ3000_famkxci2 | [OTA file](https://github.com/slacky1965/tuya_battery_switch_ts004x_zed/raw/refs/heads/main/bin/1141-d3a3-1111114b-tuya_battery_switch_ts004x_zed_model_3.zigbee) | [:camera:](doc/TS0043.md) |
| TS0044-z-SlD | TS0044 | [TS0043](https://www.zigbee2mqtt.io/devices/TS0043.html) | _TZ3000_ee8nrt2l | [OTA file](https://github.com/slacky1965/tuya_battery_switch_ts004x_zed/raw/refs/heads/main/bin/1141-d3a3-1111114b-tuya_battery_switch_ts004x_zed_model_4.zigbee) | [:camera:](doc/TS0044.md) |
| TS0046-z-SlD | TS0046 | [TS0046](https://www.zigbee2mqtt.io/devices/TS0046.html) | _TZ3000_iszegwpd | [OTA file](https://github.com/slacky1965/tuya_battery_switch_ts004x_zed/raw/refs/heads/main/bin/1141-d3a3-1111114b-tuya_battery_switch_ts004x_zed_model_6.zigbee) | [:camera:](doc/TS0046.md) |


Проверялся только в zigbee2mqtt. Требует всестороннего тестирования.

## Зачем. 

Пульт с оригинальной прошивкой умел присылать только одиночное, двойное нажатии и удержание. Биндинга не было. С группами не работал.

## Как обновить.
<details>
    <summary>Раскрыть</summary>

С выходом новой версии `zigbee2mqtt` обновление стало в разы легче. 

<img src="doc/images/z2m-version.jpg"/>

Нужно только переключиться в новый интерфейс - `zigbee2mqtt-windfront`.

> [!WARNING]
> Внимание!!! На момент выхода прошивки конвертор не был добавлен в репозиторий `zigbee-herdsman-converters`, поэтому в этой версии `zigbee2mqtt` [внешний конвертор](zigbee2mqtt/TS0044-z-SlD.js) нужно положить в директорию `external_converters` и перегрузить `zigbee2mqtt`. В последующих версиях внешний конвертор будет не нужен.

Итак, скачиваем из репозитория нужный файл обновления (какой именно смотрите в таблице вверху). Заходим в устройство. И справа видим в `Firmware version` значок облака. Нам сюда.

<img src="doc/images/z2m-ota_device.jpg"/>

Далее выбираем `Custom firmware` из вываливающегося списка.

<img src="doc/images/z2m-ota_custom_firmware.jpg"/>

После этого выбираем файл.

<img src="doc/images/z2m-ota_file_selection.jpg"/>

Далее нажимаем кнопку на пульте, т.е. будим его и жмем обновить.

<img src="doc/images/z2m-ota_update.jpg"/>

Чтобы понять, пошло обновление или нет, смотрим на изображение пульта, там должен появиться вращающийся кружок со стрелками. И в `Recent activity` будет отображаться оставшееся время в секундах и сколько загрузили в процентах.

<img src="doc/images/z2m-ota_process.jpg"/>

Ну и еще это все будет фиксироваться в логе.

<img src="doc/images/z2m-ota_log.jpg"/>

После завершения обновления пульт готов к спариванию, нужно только разрешить это в `zigbee2mqtt`. Старую версию пульта просто удаляем. 

   </details>
   
> [!WARNING]
> Чтобы ввести пульт в режим сопряжения, нужно 5 раз подряд нажать любую кнопку. Каждое нажатие будет сопровождаться вспышкой светодиода. После 5-го нажатия светодиод загорится на 3 секунды. За эти 3 секунды нужно успеть нажать кнопку и удерживать ее нажатой 1-2 секунды. Понять, вошел пульт в режим сопряжения можно по моргающему светодиоду с частотой примерно 1 раз в секунду. Время, отведенное на сопряжение, не превышает полторы минуты. После чего светодиод прекращает моргать, а пульт переходит в ждущий режим.

---

## Возможности.

<img src="doc/images/z2m-exposes.jpg"/>

- `Battery` - заряд батарейки в процентах.
- `Switch actions (Endpoint: X)` - устанавливает, какую команду посылать `On`, `Off` или `Toggle` при `Switch Type` выбранным `toggle` или `momentery` (при других выбранных параметрах эта настройка игнорируется).
- `Switch type (Endpoint: X)` - выбор действия при нажатии на кнопку.
	- `toggle` - посылает команды `OnOff` при нажатии кнопки. Команда зависит от выбранной настройки `Switch actions (Endpoint: X)`. Если выбрана настройка `On`, то при нажатии всегда уходит команда `On`, если выбрана настройка `Off`, то при нажатии всегда уходит команда `Off`. С настройкой `toggle` - тоже самое.
	- `momentary` - посылает команды `OnOff` при нажатии и при отпускании кнопки. Команда зависит от выбранной настройки `Switch actions (Endpoint: X)`. Если выбрана команда `On`, то при нажатии уходит команда `On`, а при отпускании команда `Off`. Если выбрана настройка `Off`, то при нажатии уходит команда `Off`. Если выбрана настройка `toggle`, то команда `toggle` уходит и при нажатии, и при отпускании.
	- `multifunction` - высылает `actions` - 1 нажатие, 2 нажатия, 3 нажатия, удержание и отпускание кнопки.
	- `brightness_level` - устанавливает режим работы кнопки для попеременного увеличения или уменьшения яркости.
		- Одно нажатие - посылает команду `Toggle` на устройство, для изменения его состояния на противоположное.
		- Удержание - посылает команду `Move to level`. Устройство должно плавно увеличивать или уменьшать яркость, пока последняя не станет равна значению `level`, который берется из атрибута `minLevel` или `maxLevel`. Скорость увеличения/уменьшения яркости берется из атрибута `onOffTransitionTime`, который можно изменить через dev-консоль. По умолчанию это значение равно 25. Если устройство выключено, то при команде на увеличение яркости устройство включается. При достижении минимального значения яркости устройство не выключается.
	- `brightness_level_up` (**недоступно для однокнопочного пульта**) - устанавливает режим работы кнопки для увеличения яркости.
		- Одно нажатие - посылает команду `On` на устройство.
		- Двойное нажатие - посылает команду `Step` на устройство (прибавляет к текущему значению яркости примерно 25 единиц). Если устройство выключено, оно включается.
		- Удержание - посылает команду `Move to level`. Устройство должно плавно увеличивать яркость, пока последняя не станет максимальной - значение максимальной яркости берется из атрибута `maxLevel`. Увеличение продолжается, пока удерживается кнопка и не достигнуто значение `maxLevel`. Скорость увеличения яркости берется из атрибута `onOffTransitionTime`, который можно изменить через dev-консоль. По умолчанию это значение равно 25. Если устройство было выключено, оно включается.
	- `brightness_level_down` (**недоступно для однокнопочного пульта**) - устанавливает режим работы кнопки для уменьшения яркости.
		- Одно нажатие - посылает команду `Off` на устройство.
		- Двойное нажатие - посылает команду `Step` на устройство (отнимает от текущего значения яркости примерно 25 единиц). Если устройство включено, а яркость достигла минимального предела, оно выключается.
		- Удержание - посылает команду `Move to level`. Устройство должно плавно уменьшать яркость, пока последняя не станет минимальной - значение минимальной яркости берется из атрибута `minLevel`. Уменьшение продолжается, пока удерживается кнопка и не достигнуто значение `minLevel`. Скорость уменьшения яркости берется из атрибута `onOffTransitionTime`, который можно изменить через dev-консоль. По умолчанию это значение равно 25. Устройство не выключается если яркость достигла минимального предела.
	- `scene` - работа со сценами. Сцены создаются в управляемом устройстве во вкладке `Scene` или в группе. При нажатии на кнопку посылается команда `Recall scene` на устройство или в группу.
- `Scene id (Endpoint: X)` - номер сцены (от `0` до `255`).
- `Group id (Endpoint: X)` - номер группы, если сцена была создана в группе (от `1` до `0xfff7`). Если сцена в группе не создавалась, это поле должно оставаться со значением `0`.
- `Min level` - значение, ниже которого яркость не может измениться.
- `Max level` - значение, выше которого яркость не может измениться.
- `Action` - показывает название присланных команд и атрибутов в символьном виде.


`(Endpoint: X)` - X - номер эндпоинта. Зависит от количества кнопок. Может быть от `1` до `6`. Соответственно и полей с настройками `Switch actions`, `Switch type`, `Scene id` , `Group id`, `Min level`, `Max level` тоже будет от `1` до `6`.

> [!WARNING]
> Для настройки каких-либо параметров выше, а так же репортинга, биндинга и прочего, прежде чем нажать в `Web`-интерфейсе подтверждение, нужно разбудить пульт, нажав подряд 4 раза любую кнопку. Пульт проснется на 20 секунд.

---

## Репортинг

Привычный репортинг есть только у батарейки и все равно его пришлось немного модифицировать (в угоду сбережения заряда батарейки). Cчитывание показаний батарейки и отправка репорта зависят от настройки `Max rep interval`.

<img src="doc/images/z2m-battery_reporting.jpg"/>

Если на пульте не нажимать кнопки, то он уснет на время, указанное в параметре `Max rep interval`. Т.е. исходя из скриншота - на 14400 секунд, что соответствует 4-м часам. По истечению этого времени пульт проснется, измерит батарейку, вышлет репорт и опять уснет на это же время. **Чтобы принудительно отправить репортинг батарейки, нужно 4 раза подряд нажать на любую кнопку.**

## Биндинг

Биндинг нужен для того, чтобы пульт знал, куда именно ему нужно высылать команды. И если для команд `OnOff` достаточно прибиндить только кластер `genOnOff`, то для управления светом и яркость нужно забиндить уже два кластера - `genOnOff` и `genLevelCtrl`. Лампочки, как правило, являются роутерами, и поэтому связка пульт - лапочка при прямом биндинге должна работать, даже если будет какая-то проблема с сетью (например отключился координатор).

Также можно биндить не только конкретное устройство, но и целиком **`группу`**.

Также можно забиндить какое-нибудь устройство, которое поддерживает процедуру `Find and bind`. Чтобы ввести пульт в режим поиска нужно 4 раза быстро нажать нужную кнопку и затем в течение 3 секунд нажать и удерживать эту же кнопку.

## Группы

Еще одна возможность управлять светом без настройки биндинга - это группы. Допустим мы настроили пульт таким образом, что 2-я кнопка отвечает за включение и увеличения яркости, а 3-я за выключение и уменьшения яркости.

<img src="doc/images/z2m-dev_settings.jpg"/>

Для того, чтобы все работало без настройки биндинга, создаем группу с каким-нибудь именем и присваиваем ей номер, допустим `10`. Номер или `ID` этой группы также является ее сетевым групповым адресом. Теперь в эту группу добавляем `Endpoint 1` лампочки и `Endpoint 2`, `Endpoint 3` пульта, где мы как раз настроили управление светом.

<img src="doc/images/z2m-group.jpg"/>

Все, больше никаких настроек не требуется. Теперь, если мы нажимаем на пульте кнопку 2 или 3, лампочка начинает выполнять команды.

---

Связаться со мной можно в **[Telegram](https://t.me/slacky1965)**.

### Если захотите отблагодарить автора, то это можно сделать через [ЮMoney](https://yoomoney.ru/to/4100118300223495)

## История версий
- 1.0.01
	- Добавлен 4-х кнопочный пульт `TS0044 _TZ3000_ee8nrt2l`.
- 1.0.02
	- Добавлен 2-х кнопочный пульт `TS0042 _TZ3000_fkvaniuu`.
	- При настройке `momentary` команда при отпускании кнопки не успевала отработать, пульт засыпал. Исправлено.
- 1.0.03
	- Добавлен 3-х кнопочный пульт `TS0043 _TZ3000_famkxci2`.
	- Добавлен 6-х кнопочный пульт `TS0046 _TZ3000_iszegwpd`.
	- Устранена проблема, когда группа отрабатывала с любой кнопки.
- 1.0.04
	- Изменена процедура отправки команд и репортов. По идее это должно устранить проблему, когда команда не всегда отправлялась на прибинженное устройство.
- 1.0.05
	- Добавлена возможность отправлять чип в сон при длительном удержании кнопки.
	- Уменьшено время "не сна" после нажатия кнопки.
- 1.0.06
	- Изменена работа с командами изменения яркости.
	- Добавлена возможность забиндить пульт на другое устройство с помощью процедуры `Find and bind`.
	- Добавлена возможность перезапустить пульт. Для этого нужно нажать быстро 10 раз подряд любую кнопку.
	- Добавлена проверка доставки команды через `aps ack`. Команда отправляется первый раз, если `aps ack` вернул ошибку, то команда отправляется повторно.
	- Убраны лишние `endpoint'ы`. Раньше отражались все 6. Теперь, если это пульт с двумя кнопками, то отражаются только два `endpoint'а` и т.д.
- 1.0.07
    - Увеличен период "бодрствования" при сопряжении.

[Наверх](#Top)
