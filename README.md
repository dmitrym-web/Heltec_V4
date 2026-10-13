# Heltec V4 — подключение к удалённому MeshMonitor

## Содержание

- [1. Введение](#1-введение)
- [2. Характеристики Heltec V4](#2-характеристики-heltec-v4)
- [3. Шаг 1 — Прошивка Heltec V4](#3-шаг-1--прошивка-heltec-v4)
- [4. Шаг 2 — Подготовка компьютера](#4-шаг-2--подготовка-компьютера)
- [5. Шаг 3 — Настройка Heltec V4](#5-шаг-3--настройка-heltec-v4)
- [6. Шаг 4 — Настройка компьютера](#6-шаг-4--настройка-компьютера)
- [7. Шаг 5 — Запуск и проверка](#7-шаг-5--запуск-и-проверка)
- [8. Прошивки и приложения для Heltec V4](#8-прошивки-и-приложения-для-heltec-v4)

---

## 1. Введение

Для реализации интеграции платы Heltec V4, расположенной внутри локальной сети, к сервису MeshMonitor, необходимо её подключение и настройка. В отличие от T114, **Heltec V4 имеет встроенный модуль Wi-Fi** (ESP32-S3), что позволяет подключаться к сети напрямую, без обязательного использования компьютера-посредника.

**Ключевое отличие:** Heltec V4 может работать с MeshMonitor двумя принципиально разными способами:

1. **Напрямую по Wi-Fi** — плата сама подключается к локальной сети и отправляет MQTT-данные в MeshMonitor. Компьютер не нужен.
2. **Через компьютер** — по USB или Bluetooth, как T114. Используется, если Wi-Fi недоступен или нужна проводная надёжность.

Отсюда пять сценариев:

    Heltec V4 → Wi-Fi → MQTT → MeshMonitor
    Heltec V4 → USB → Компьютер → TCP или MQTT → MeshMonitor
    Heltec V4 → Bluetooth → Компьютер → TCP или MQTT → MeshMonitor

| # | Heltec V4 → | Посредник | → MeshMonitor | Что запущено |
|:---:|---|---|---|---|
| 1 | Wi-Fi | — | MQTT | Ничего (плата сама) |
| 2 | USB | Компьютер | TCP | Serial Bridge |
| 3 | USB | Компьютер | MQTT | Serial Bridge + MQTT Proxy |
| 4 | Bluetooth | Компьютер | TCP | Bluetooth Bridge |
| 5 | Bluetooth | Компьютер | MQTT | Bluetooth Bridge + MQTT Proxy |

### Когда какой сценарий использовать

| Сценарий | Когда использовать |
|---|---|
| 1 (Wi-Fi → MQTT) | Плата в зоне Wi-Fi. Самый простой путь, компьютер не нужен. |
| 2 (USB → TCP) | MeshMonitor и компьютер в одной сети. Wi-Fi недоступен. |
| 3 (USB → MQTT) | MeshMonitor удалён, связь через MQTT-брокер. |
| 4 (Bluetooth → TCP) | То же, что сценарий 2, но без кабеля. |
| 5 (Bluetooth → MQTT) | То же, что сценарий 3, но без кабеля. |

[↑ Наверх](#содержание)

---

## 2. Характеристики Heltec V4

| Параметр | Значение |
|---|---|
| Микроконтроллер | ESP32-S3 (dual-core Xtensa LX7, 240 МГц) |
| ОЗУ | 512 КБ SRAM |
| PSRAM | 2 МБ (S3R2) или 8 МБ (S3R8) — зависит от ревизии |
| Флеш-память | 16 МБ (внешняя) |
| LoRa-чип | Semtech SX1262 + внешний PA (KCT8103L в V4.3.1) |
| **Беспроводная связь** | **Wi-Fi 802.11 b/g/n + Bluetooth 5.0 (BLE)** |
| LoRa-диапазоны | 863–870 МГц / 902–928 МГц |
| Мощность передачи (TX) | до 28 ± 1 дБм |
| Чувствительность (RX) | до ~ −137 дБм (SF12/125 кГц) |
| Питание | USB-C (5 В), Li-Ion (3.7 В), солнечная панель (SH1.25-2P) |
| GPS-интерфейс | SH1.25-8Pin (дополнительно) |
| Размеры | 50.80 × 22.86 мм (стандартный форм-фактор) |

### Распиновка и интерфейсы

| Интерфейс | Кол-во | Назначение |
|---|:---:|---|
| SPI | 2 | LoRa-чип, дисплей, флеш-память |
| I2C | 2 | Часы реального времени (RTC) и др. |
| UART | 2 | GPS-модуль и др. |
| GPIO | 40-pin | АЦП (ADC), ШИМ (PWM). Пин-разъём расширен с 36 до 40. |
| USB-UART | — | Встроенный (CP2102 удалён в V4) |
| GNSS | 1 | SH1.25-8Pin |
| Solar | 1 | SH1.25-2P |
| 2.4G Antenna | 1 | FPC + IPEX connector |

[↑ Наверх](#содержание)

---

## 3. Шаг 1 — Прошивка Heltec V4

Heltec V4 прошивается через веб-флешер или ESP32 Flash Tool. **Важное отличие от T114:** в V4 удалён чип CP2102, поэтому для входа в режим прошивки нужно вручную перевести плату в Bootloader mode.

### Важно перед прошивкой

**Для hardware version V4.3 используйте прошивку 2.7.20 или выше.** OLED-версия и touchscreen-версия требуют **разных файлов прошивки**.[reference:4]

### Способ 1: Веб-флешер (Chrome / Edge)

| Шаг | Действие |
|---|---|
| 1 | Подключи Heltec V4 к компьютеру по USB-C |
| 2 | **Нажми и удерживай кнопку USER**, затем нажми **RST** один раз, отпусти USER |
| 3 | Открой `flasher.meshtastic.org` **в Chrome или Edge** |
| 4 | Выбери плату **Heltec V4** |
| 5 | Выбери версию прошивки **2.7.20 или выше** |
| 6 | Нажми **Flash** → выбери **Full Erase and Install** |
| 7 | Выбери COM-порт и нажми **Connect**. Флешер сотрёт память и запишет прошивку |
| 8 | После записи плата перезагрузится |

**Версия 2.7.20** по умолчанию использует **горизонтальную** ориентацию touchscreen, а **2.7.21 и выше** — **вертикальную**.[reference:5]

### Способ 2: ESP32 Flash Tool

| Шаг | Действие |
|---|---|
| 1 | Скачай **ESP32 Flash Tool** с сайта Heltec |
| 2 | Подключи V4 по USB-C, войди в Bootloader mode (USER + RST) |
| 3 | Загрузи `.bin` файл прошивки |
| 4 | Нажми **Start**. Дождись завершения |

### Траблшутинг прошивки

| Проблема | Причина | Решение |
|---|---|---|
| **Плата не входит в Bootloader** | Неверная последовательность кнопок | Удерживай **USER**, нажми **RST**, отпусти **USER**. Попробуй несколько раз |
| **COM-порт не появляется** | Кабель только для зарядки | Замени USB-кабель на **data-кабель** |
| **Прошивка записывается, но плата не загружается** | Неверная версия для hardware revision | Проверь версию платы (V4.2 / V4.3) и выбери правильную прошивку |
| **Плата зависла в Bootloader** | Bootloader не загрузил приложение | Нажми **RST** один раз. Если не помогает — прошей заново с Full Erase |
| **OLED не работает после прошивки** | Прошита touchscreen-версия на OLED-плату | Скачай правильный файл (OLED vs touchscreen) |

### Обновление по воздуху (OTA)

Heltec V4 поддерживает **Bluetooth OTA** через приложение **nRF Device Firmware Update** (для Meshtastic) или через веб-интерфейс (для MeshCore). Требуется `.zip` файл.

| Шаг | Действие |
|---|---|
| 1 | Установи **nRF Device Firmware Update** в App Store / Play Store |
| 2 | Скачай **ZIP**-версию прошивки с `flasher.meshtastic.org` |
| 3 | Выбери ZIP-файл и устройство Heltec V4 в DFU-приложении |
| 4 | Нажми **Upload**. Процесс занимает 2–5 минут |
| 5 | Держи экран телефона включённым, не уводи приложение в фон |

[↑ Наверх](#содержание)

---

## 4. Шаг 2 — Подготовка компьютера

На компьютере должна быть установлена утилита **Meshtastic Python CLI**. Без неё команд `meshtastic --set ...` в системе нет.

### Установка CLI

| ОС | Команды |
|---|---|
| **Windows** | 1. Установи Python 3.9+ с python.org, галочка **Add Python to PATH**<br>2. `pip3 install --upgrade meshtastic`<br>3. Проверка: `meshtastic --version` |
| **macOS** | `brew install python`<br>`brew install pipx`<br>`pipx ensurepath`<br>`pipx install "meshtastic[cli]"` |
| **Linux** | `sudo apt install python3-pip`<br>`pip3 install --upgrade "meshtastic[cli]"`<br>`sudo usermod -aG dialout $USER`<br>перелогиниться |

### Определение порта Heltec V4

| ОС | Как узнать порт |
|---|---|
| **Windows** | Диспетчер устройств → Порты (COM и LPT) → например `COM3` |
| **Linux / macOS** | `ls /dev/tty*` → `/dev/ttyUSB0` или `/dev/ttyACM0` |

Все команды ниже подаются с флагом `--port`, например:

    meshtastic --port COM3 --set mqtt.enabled true

[↑ Наверх](#содержание)

---

## 5. Шаг 3 — Настройка Heltec V4

### Сценарий 1: Wi-Fi → MQTT (без компьютера)

Heltec V4 подключается к Wi-Fi напрямую и публикует данные в MQTT. Компьютер не нужен.

**Настройка на самой плате (через MUI или приложение Meshtastic):**

| Параметр | Значение |
|---|---|
| Wi-Fi SSID | Название твоей сети |
| Wi-Fi Password | Пароль |
| MQTT Enabled | Да |
| MQTT Address | Адрес брокера (если MeshMonitor использует встроенный — IP сервера MeshMonitor) |
| MQTT Username | Логин |
| MQTT Password | Пароль |
| MQTT Root Topic | `msh/твой_регион/2` |
| Uplink Enabled | Да |
| Downlink Enabled | Да |

> **Важно:** Wi-Fi на Heltec V4 можно использовать **только для MQTT-загрузки данных**. Подключение к мобильному приложению через Wi-Fi **невозможно** — для этого используется Bluetooth.[reference:6]

**Через CLI (если плата подключена по USB):**

    meshtastic --port COM3 --set network.wifi_enabled true
    meshtastic --port COM3 --set network.wifi_ssid "твой_ssid"
    meshtastic --port COM3 --set network.wifi_psk "твой_пароль"
    meshtastic --port COM3 --set mqtt.enabled true
    meshtastic --port COM3 --set mqtt.address адрес_брокера
    meshtastic --port COM3 --set mqtt.username логин
    meshtastic --port COM3 --set mqtt.password пароль
    meshtastic --port COM3 --set mqtt.root msh/твой_регион/2
    meshtastic --port COM3 --ch-index 0 --ch-set uplink_enabled true
    meshtastic --port COM3 --ch-set downlink_enabled true

### Сценарии 2–5: Через компьютер

| Сценарий | Serial-режим | MQTT Client Proxy | Uplink/Downlink |
|---|:---:|:---:|:---:|
| 2 (USB → TCP) | Да | Нет | Нет |
| 3 (USB → MQTT) | Да | Да | Да |
| 4 (Bluetooth → TCP) | Нет | Нет | Нет |
| 5 (Bluetooth → MQTT) | Нет | Да | Да |

**Serial-режим (сценарии 2, 3):**

    meshtastic --port COM3 --set serial.enabled true --set serial.echo false --set serial.mode SIMPLE --set serial.baud BAUD_115200

**MQTT Client Proxy (сценарии 3, 5):**

    meshtastic --port COM3 --set mqtt.enabled true
    meshtastic --port COM3 --set mqtt.proxy_to_client_enabled true

`proxy_to_client_enabled` — обязательно. Без него плата не будет передавать MQTT-трафик через компьютер.

**Uplink/Downlink (сценарии 3, 5):**

    meshtastic --port COM3 --ch-index 0 --ch-set uplink_enabled true --ch-set downlink_enabled true

[↑ Наверх](#содержание)

---

## 6. Шаг 4 — Настройка компьютера

### Сценарий 1: Wi-Fi → MQTT

Компьютер **не участвует**. Плата подключается к Wi-Fi и сама отправляет MQTT-данные в брокер MeshMonitor.

**MeshMonitor:** данные приходят автоматически, если брокер и root topic совпадают. Ничего дополнительно настраивать не нужно.

### Сценарий 2: USB → TCP

    Heltec V4 → USB → Serial Bridge (TCP:4403) → MeshMonitor

**Плата:** только Serial-режим.
**Компьютер:** один контейнер.

    # docker-compose.yml
    services:
      serial-bridge:
        image: ghcr.io/yeraze/meshtastic-serial-bridge:latest
        container_name: meshtastic-serial-bridge
        devices:
          - /dev/ttyUSB0:/dev/ttyUSB0
        ports:
          - "4403:4403"
        restart: unless-stopped
        environment:
          - SERIAL_DEVICE=/dev/ttyUSB0
          - BAUD_RATE=115200
          - TCP_PORT=4403

**MeshMonitor:** `Dashboard → Sources → Add Source` → тип `meshtastic_tcp` → адрес `serial-bridge` → порт `4403`.

### Сценарий 3: USB → MQTT

    Heltec V4 → USB → Serial Bridge (TCP:4403) → MeshMonitor → Virtual Node (4404) ← MQTT Proxy → Embedded Broker

**Плата:** Serial-режим + MQTT Client Proxy + Uplink/Downlink.
**Компьютер:** два контейнера.

    # docker-compose.yml
    services:
      serial-bridge:
        image: ghcr.io/yeraze/meshtastic-serial-bridge:latest
        container_name: meshtastic-serial-bridge
        devices:
          - /dev/ttyUSB0:/dev/ttyUSB0
        ports:
          - "4403:4403"
        restart: unless-stopped
        environment:
          - SERIAL_DEVICE=/dev/ttyUSB0
          - BAUD_RATE=115200
          - TCP_PORT=4403

      mqtt-proxy:
        image: ghcr.io/ln4cy/mqtt-proxy:latest
        container_name: mqtt-proxy
        restart: unless-stopped
        depends_on:
          - serial-bridge
        environment:
          - INTERFACE_TYPE=tcp
          - TCP_HOST=serial-bridge
          - TCP_PORT=4403

**MeshMonitor:** включить Virtual Node (порт `4404`), убедиться, что Embedded MQTT Broker добавлен как источник.

### Сценарий 4: Bluetooth → TCP

    Heltec V4 → Bluetooth → Bluetooth Bridge (TCP:4403) → MeshMonitor

**Компьютер:** один контейнер.

    # docker-compose.yml
    services:
      ble-bridge:
        image: ghcr.io/yeraze/meshtastic-ble-bridge:latest
        container_name: meshtastic-ble-bridge
        privileged: true
        ports:
          - "4403:4403"
        restart: unless-stopped
        volumes:
          - /var/run/dbus:/var/run/dbus
          - /var/lib/bluetooth:/var/lib/bluetooth:ro
        environment:
          - BLE_ADDRESS=AA:BB:CC:DD:EE:FF

**MeshMonitor:** `Dashboard → Sources → Add Source` → тип `meshtastic_tcp` → адрес `ble-bridge` → порт `4403`.

### Сценарий 5: Bluetooth → MQTT

    Heltec V4 → Bluetooth → Bluetooth Bridge (TCP:4403) → MeshMonitor → Virtual Node (4404) ← MQTT Proxy → Embedded Broker

**Компьютер:** два контейнера.

    # docker-compose.yml
    services:
      ble-bridge:
        image: ghcr.io/yeraze/meshtastic-ble-bridge:latest
        container_name: meshtastic-ble-bridge
        privileged: true
        ports:
          - "4403:4403"
        restart: unless-stopped
        volumes:
          - /var/run/dbus:/var/run/dbus
          - /var/lib/bluetooth:/var/lib/bluetooth:ro
        environment:
          - BLE_ADDRESS=AA:BB:CC:DD:EE:FF

      mqtt-proxy:
        image: ghcr.io/ln4cy/mqtt-proxy:latest
        container_name: mqtt-proxy
        restart: unless-stopped
        depends_on:
          - ble-bridge
        environment:
          - INTERFACE_TYPE=tcp
          - TCP_HOST=ble-bridge
          - TCP_PORT=4403

**MeshMonitor:** включить Virtual Node (порт `4404`), убедиться, что Embedded MQTT Broker добавлен как источник.

[↑ Наверх](#содержание)

---

## 7. Шаг 5 — Запуск и проверка

    docker compose up -d
    docker compose logs -f

| Что проверяем | Команда |
|---|---|
| TCP (сценарии 2, 4) | `nc -zv localhost 4403` |
| MQTT (сценарии 1, 3, 5) | `mosquitto_sub -h localhost -p 1883 -t 'msh/#' -v` |

Если пошли пакеты — данные от Heltec V4 доходят до MeshMonitor.

[↑ Наверх](#содержание)

---

## 8. Прошивки и приложения для Heltec V4

### Прошивки (устанавливаются на саму плату)

| Прошивка | Описание | Ссылка |
|---|---|---|
| **Meshtastic** | Стандартная прошивка. MUI (touchscreen UI) или OLED-версия. | [flasher.meshtastic.org](https://flasher.meshtastic.org) |
| **MeshCore** | Альтернативная прошивка для mesh-сетей. | [flasher.meshcore.io](https://flasher.meshcore.io) |
| **MC-Term** | Ретро-прошивка на базе MeshCore. Собственный экран, карта, чат. Устанавливается на Heltec V4 с дисплеем. | [GitHub - dabeani/meshcoreterm](https://github.com/dabeani/meshcoreterm) |
| **wadamesh** | Touch-UI прошивка на базе MeshCore. Карта, чат, контакты, каналы. Для Heltec V4 TFT. | [GitHub - ALLFATHER-BV/wadamesh](https://github.com/ALLFATHER-BV/wadamesh) |

### Приложения (работают на компьютере или телефоне)

| Приложение | Платформа | Ссылка | Тип подключения | Настройка |
|---|---|---|---|---|
| **Meshtastic Desktop** | Windows / macOS / Linux | [GitHub Releases](https://github.com/meshtastic/Meshtastic-Android/releases) (артефакты `desktop`) | USB Serial, Bluetooth | Скачать `.msi`/`.dmg`/`.AppImage`, запустить, подключить по USB, выбрать порт |
| **Meshtastic Web** | Chrome / Edge | [client.meshtastic.org](https://client.meshtastic.org) | WebSerial (USB) | Только Chrome/Edge. **New Connection → Serial**, выбрать порт |
| **mc-webui** | Windows / macOS / Linux | [DeepWiki - MarekWo/mc-webui](https://deepwiki.com/MarekWo/mc-webui) | Serial (USB), Bluetooth | Flask-приложение. Подключается к Heltec V3/V4 по Serial или BLE. |
| **Meshy** | Linux | [Codeberg - sesivany/meshy](https://codeberg.org/sesivany/meshy) | USB Serial | Flatpak-пакет. GUI для Meshtastic |

> **Для MUI-версии Heltec V4:** на самой плате есть встроенный экран (touchscreen или OLED). Wi-Fi настраивается прямо через **Settings → WiFi**. Однако Wi-Fi работает **только для MQTT**, не для подключения к мобильному приложению. **В MUI Bluetooth не работает.**[reference:7]

[↑ Наверх](#содержание)