# sbdv-00120-esphome
Modification of smart Wifi lamp Sber SBDV-00120 for work with esphome with replacement of controller esp-01f (esp8266ex)
Модификация умной лампы Sber от SberDevices SBDV-00120 (LED A70 E27) для работы с ESPHome и Home Assistant.
Обзор лампы в заводском исполнении представлен на хабре (https://habr.com/ru/companies/lamptest/articles/829536/).

Лампа основана на микроконтроллере Beken BK7231N. Управление светодиодами осуществляется по i2c led-драйвером BP5758D (5 каналов, до 90 mA на канал). Порядок каналов: Blue, Green, Red, White Cold, White Warm.
Модель управляющей платы (MCU) - CB2L. На управляющей плате есть тестовые точки для подключения UART (TX1, RX1, TX2); SPI (CSN), RST (CEN), GPIO (P24, P26, P6, P7, P8). Имеющиеся фото внутренностей находятся в репозитории в папке photos.

Внимание: контакты MOSI, MISO, CLK на плате не разведены. Подпаяться и считать/перепрограммировать прошивку через spi программатор у меня не удалось.

Разборка аналогична другой лампе от сбера, подробная инструкция выложена пользователем @esnet146 (https://github.com/esnet146/rgb-cw-lamp-sber).

Я решил заменить плату микроконтроллера на подходящую по размерам ESP01.
Микроконтроллер esp8266 перед впайкой на плату был заранее прошит через переходник usb-uart на базе ft232. Подключение осуществлял тонким многожильным проводом типа МГТФ. Крепление на двухстороннем скотче.
На плате ESP01 имеется два светодиода: светодиод POWER был удален, светодиод статуса (GPIO1) оставлен, настроен в конфиге как status_led.

Подключение к плате лампы по следующей схеме (порядок от края платы лампы):
| ЛАМПА | CRL2 | ESP01 |
| ----- | ---- | ----- |
| 3.3V  | 3V3  | VCC   |
| GND   | GND  | GND   |
| ?     | P24  | NC    |
| ?     | P26  | NC    |
| ?     | P6   | NC    |
| CLK   | P7   | GPIO0 |
| DAT   | P8   | GPIO2 |


Нужно вставить свой "encryption key" в разделе api. Возможно потребуется внести в файл secrets вашего esphome значения параметров ota_password (пароль для обновления по воздуху) и wifi_password (пароль для подключения к точке доступа wifi). Можете поправить конфиг напрямую вписав необходимые вам пароли.

Калибровки тока (current) на каждом канале сделаны на глаз, возможно вам потребуется внести корректировки. 
Больше всего меня смущают текущие калибровки тока RGB каналов. 
Включен режим "color interlock" когда одновременно может работать только либо rgb, либо ccw. 

Лампа терпимо греется, насколько стало ярче/тусклее по сравнению с заводским исполнением проверить не могу, т.к. имею всего одну лампу.

Конфиг для esphome: 
```
esphome:
  name: sbdv-00120
  friendly_name: SBDV-00120

# Esp8266ex с флешкой на 1MB, если флешка установлена на 4MB, то заменить board на d1.
esp8266:
  board: esp01_1m

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "Ключ сгенерированный ESPHome"

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "SberLamp AP"
    password: "12345678"

captive_portal:

status_led:
  pin:
    number: GPIO1
    inverted: true

bp5758d:
  data_pin: GPIO2
  clock_pin: GPIO0

output:
  - platform: bp5758d
    id: output_blue
    channel: 1
    current: 20
    min_power: 0.001
    zero_means_zero: true
  - platform: bp5758d
    id: output_green
    channel: 2
    current: 8
    min_power: 0.001
    zero_means_zero: true
  - platform: bp5758d
    id: output_red
    channel: 3
    current: 15
    min_power: 0.001
    zero_means_zero: true
  - platform: bp5758d
    id: output_cold
    channel: 4
    current: 24
    min_power: 0.001
    zero_means_zero: true
  - platform: bp5758d
    id: output_warm
    channel: 5
    current: 26
    min_power: 0.001
    zero_means_zero: true

light:
  - platform: rgbww
    name: SberLamp
    red: output_red
    green: output_green
    blue: output_blue
    cold_white: output_cold
    warm_white: output_warm
    cold_white_color_temperature: 6200 K
    warm_white_color_temperature: 2700 K 
    color_interlock: true
    constant_brightness: false
    restore_mode: RESTORE_DEFAULT_OFF
    default_transition_length: 400ms
    gamma_correct: 2.8

sensor:
  - platform: wifi_signal
    name: Wifi Signal
    update_interval: 10min

```


