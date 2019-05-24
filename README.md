#################################################################################################
#
# stm32f4 + sim868 (gsm,gps,bluetooth) + ssd1306(i2c) + bmp280(i2c) + bh1750(i2c) + SDcard(spi)
#
#################################################################################################


## Состав рабочего оборудования:

```
* stm32f4 (STM32F407GDISC1 board) - плата микроконтроллера
* ssd1306 - OLED дисплей 0.96" 128x64 (интерфейс I2C)
* bmp280 - датчик атмосферного давления и температуры воздуха (интерфейс I2C).
* bh1750 - датчик освещенности (интерфейс I2C).
* sim868 - плата с модулем SIM868 (SIMCOM).
* sdcard - плата адаптера SD Card (SPI).
```


# Средства разработки:

```
* STM32CubeMX - графический пакет для создание проектов (на языке Си) под микроконтроллеры семейства STM32
  (https://www.st.com/en/development-tools/stm32cubemx.html).
* System Workbench for STM32 - IDE среда разработки ПО для микроконтроллеров семейства STM32
  (https://www.st.com/en/development-tools/sw4stm32.html).
```


# Функционал:

* Устройство использует программные средства freeRTOS :
  - StartGpsTask - нитка (задача), работающая с данными GPS (NMEA) модуля SIM868.
  - StartAtTask - нитка (задача), работающая портом команд модуля SIM868.
  - STartSensTask - нитка (задача), работающая с датчиками bmp280, bh1750.
  - StartDefTask - нитка (задача), выполняющая функцию main.
  - mailQueue - очередь для передачи данных от bmp280 и bh1750 из задачи StartSensTask в задачу StartDefTask,
    для вывода данных на дисплей ssd1306 и для вывода в последовательный порт (USART3).
  - binSemHandle - бинарный семафор на доступ к порту usart3 (по записи).
* Устройство инициализирует некоторые интерфейсы микроконтроллера :
  - GPIO : подключены четыре сетодиода : PD12..PD15, один пин (PA2) статус sim868 (on/off), один пин (PA3) вкл./выкл. sim868
  - I2C1 : режим мастера с частотой 400Кгц (шина ослуживает ssd1306, bmp280, bh1750).
  - USART3 : параметры порта 500000 8N1 - порт для логов и передачи AT команд модулю SIM868, если подключен комп.
  - UART4 : параметры порта 9600 8N1 - порт AT команд модуля SIM868.
  - USART2 : параметры порта 115200 8N1 - порт для приема данных GPS (NMEA) от модуля SIM868.
  - TIM2 : таймер-счетчик временных интервалов в 250 мс. и 1 секунду, реализован в callback-функции.
  - SPI1 : обслуживает SD card c fatfs; линии интерфейса : CS(PB5), SCK(PA5), MISO(PA6), MOSI(PA7).
  - RTC : часы реального времени, могут быть установлены с помощью команды DATE:
* Системный таймер (TIM1) считает миллисекунды от начала работы устройства.
* Прием данных по всем последовательным портам (USART3, UART4, USART2) выполняется в callback-функции обработчика прерывания.
  Принятые данные передаются в соответствующие задачи (нитки) через структурные очереди.
* Каждые 10 секунд считываются данные с датчиков bmp280 и bh1750, выполняется пересчет атмосферного
  давления в мм ртутного столба и температуры в градусы Цельсия, полученные данные выдаются
  в USART3, например :

```
22.05.2019 14:54:54 | [gsmONOFF] GSM_KEY set to 0
22.05.2019 14:54:55 | [gsmONOFF] GSM_KEY set to 1
NORMAL POWER DOWN
22.05.2019 14:54:58 | [gsmONOFF] GSM_KEY set to 0
22.05.2019 14:54:59 | [gsmONOFF] GSM_KEY set to 1

RDY
+CFUN: 1
+CPIN: NOT INSERTED
AT
OK
AT+CMEE=0
OK
AT+GMR
Revision:1418B03SIM868M32_BT
OK
AT+GSN
868183030452648
OK
AT+CCLK?
+CCLK: "04/01/01,05:03:17+02"
OK
AT+CGNSPWR=1
OK
AT+CGNSPWR?
+CGNSPWR: 1
OK
AT+CSQ
+CSQ: 0,0
OK
AT+CREG?
+CREG: 0,4
OK
AT+CGNSINF
+CGNSINF: 1,0,19800105235947.000,,,,0.00,0.0,0,,,,,,0,0,,,,,
OK
```

22.05.2019 14:55:05 | +CGNSINF: 1,0,19800105235947.000,,,,0.00,0.0,0,,,,,,0,0,,,,,

```
{
    "MsgType": "+CGNSINF",
    "DevName": "STM32_SIM868",
    "DevTime": 446,
    "EpochTime": 1558536905,
    "UTC": "05.01.1980 23:59:47.000",
    "Run": 1,
    "Status": "Invaid",
    "Latitude": 0.0,
    "Longitude": 0.0,
    "Altitude": 0,
    "Speed": 0.0,
    "Dir": 0.0,
    "Mode": 0,
    "HDOP": 0.0,
    "PDOP": 0.0,
    "VDOP": 0.0,
    "SatGPSV": 0,
    "SatGNSSU": 0,
    "SatGLONASSV": 0,
    "dBHz": 0,
    "HPA": 0.0,
    "VPA": 0.0
}
```

22.05.2019 14:55:08 | $GNRMC,235951.868,V,,,,,0.00,0.00,050180,,,N*50

```
{
    "MsgType": "$GNRMC",
    "DevName": "STM32_SIM868",
    "DevTime": 449,
    "EpochTime": 1558536908,
    "UTC": "05.01.80 23:59:51.868",
    "Status": "Invaid",
    "Latitude": 0.0,
    "Longitude": 0.0,
    "Speed": 0.0,
    "Dir": 0.0,
    "Mode": "N",
    "CRC": "50"
}
```

22.05.2019 14:55:11 | BMP280: Press=753.26 mmHg, Temp=26.90 DegC; BH1750: Lux=39.16 lx

```
{
    "MsgType": "SENSOR",
    "DevName": "STM32_SIM868",
    "DevTime": 452,
    "EpochTime": 1558536911,
    "Press": 753.269287,
    "Temp": 26.905843,
    "Lux": 39.166667
}
```

  Эти же данные (время работы, напряжение питания, атмосферное давление, температура воздуха и освещенность)
отображаются на дисплей ssd1306.

* Через usart3 можно отправлять команды на модуль SIM868, например :

```
AT+GMR
Revision:1418B03SIM868M32_BT
OK
```

* Порт usart2 предназначен для приема данных GPS (NMEA) от модуля SIM868.

* Функционал проекта в процессе пополнения.

