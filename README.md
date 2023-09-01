# HCPBridge with MQTT + HomeAssistant Support
![image](https://user-images.githubusercontent.com/14005124/215204028-66bb0342-6bc2-48dc-ad8e-b08508bdc811.png)

Emulates Hörmann UAP1-HCP board using an ESP32 and a RS485 converter, and exposes garage door controls over web page and MQTT.

**Compatible with the following motors (UAP1-HCP / HCP2-Bus / Modbus):**

* SupraMatic E/P Serie 4
* ProMatic Serie 4

It is **not** compatible with E3 series motors. Previous generations have different protocol, different pin layout, and already have another supporting project.

## Functions

* Get current status (door open/close/position, light on/off)
* Trigger the actions
  * light on/off
  * gate open, close, stop
  * gate position: half, ventilation, custom (MQTT set_position compatible)
* Web Interface
* Web Service (GET)
* OTA Update (with username and password)
* Set WiFi and other user Settings setup in Web Interface
<!-- * AsyncWifiManger (hotspot when disconnected) -->
* DS18X20, BME280 or DHT22 sensor (with threshold)
* HCSR04 Proximity sensor (to know if a car is below)
* Efficient MQTT messages (send only MQTT Message if Door state changed)

## Known Bugs
* None

## Notes by original author

Eigentlich war das Ziel, die Steuerung komplett nur mit einem ESP8266 zu realisieren, allerdings gibt es durch die WLAN und TCP/IP-Stackumsetzung Timeoutprobleme, die zum Verbindungsabbruch zwischen dem Antrieb und der Steuerung führen können. Durch die ISR-Version konnte das Problem zwar reduziert aber nicht komplett ausgeschlossen werden. Daher gibt es zwei weitere Versionen, die bisher stabil laufen. Eine Variante nutzt den ESP32 statt ESP8266, welcher über 2 Kerne verfügt und so scheinbar besser mit WLAN-Verbindungsproblemen zurecht kommt. Die andere Option ist ein zweiter MCU, der die MODBUS Simulation übernimmt, sodass sich der ESP8266 nur noch um die Netzwerkkommunikation und das WebInterface kümmern muss.

## Web Interface

***http://[deviceip]***

![alt text](Images/webinterface.PNG)

## Web Service

### Commands

***http://[deviceip]/command?action=[id]***

| id | Function | Other Parameters
|--------|--------------|--------------|
| 0 | Close | |
| 1 | Open | |
| 2 | Stop | |
| 3 | Ventilation | |
| 4 | Half Open | |
| 5 | Light toggle | |
| 6 | Restart | |
| 7 | Set Position | position=[0-100] |



### Status report

***http://[deviceip]/status***

Response (JSON)

```
{
"valid": true,
"doorstate": 64,
"doorposition": 0,
"doortarget": 0,
"lamp": false,
"temp": 19.94000053,
"lastresponse": 0,
"looptime": 1037,
"lastCommandTopic": "hormann/garage_door/command/door",
"lastCommandPayload": "close"
}
```

### Wifi Status

***http://[deviceip]/sysinfo***

### OTA Firmware update (AsyncElegantOTA)

***http://[deviceip]/update***

![image](https://user-images.githubusercontent.com/14005124/215216505-8c5abe46-5d40-402b-963a-e3825c63d417.png)

## Pinout RS485 (Plug)

![alt text](Images/plug-min.png)

1. GND (Blue)
2. GND (Yellow)
3. B- (Green)
4. A+ (Red)
5. \+24V (Black)
6. \+24V (White)

## RS485 Adapter

![alt text](Images/rs485board-min.png)  
Pins A+ (Red) and B- (Green) need a 120 Ohm resistor for BUS termination. Some RS485 adapters provide termination pad to be soldered.

## DS18X20 Temperature Sensor

![DS18X20](Images/ds18x20.jpg) <br/>
DS18X20 connected to GPIO4.

## HC-SR04 Ultra sonic proximity sensor

To use set the appropriate build env in platformio.ini. The pin can either by set in the configuration.h or in the Web UI.

It will send an mqtt discovery for two sensor one for the distance in cm available below the sensor and the other informing if the car park is available. It compare if the distance below is less than the maximal measured distance then car park is not available. The hcsr04_maxdistanceCm is initialised with 150cm in main.cpp, This setting work for me to get the right status if I restart the esp32 with the car below.

## Circuit

![alt text](Images/esp32.png)

ESP32 powering requires a Step Down Module such as LM2596S DC-DC, but any 24VDC ==> 5VDC will do, even the tiny ones with 3 pin.
Please note that the suggested serial pins for serial interfacing, on ESP32, are 16 RXD and 17 TXD.

It is possible to implement it with protoboard and underside soldering:

![alt text](Images/esp32_protoboard.jpg)

![alt text](Images/esp32_protoboard2.jpg)

## 3D Case

![Body](Images/body.jpg)
Body with optional BME280 Sensor

![Lid](Images/lid.jpg)
Another prototype

## Flashing instruction
See flashing_instructions.md

## Installation

![alt text](Images/antrieb-min.png)

* Connect the board to the BUS
* Run a BUS scan: 
  * Old HW-Version / Promatic4: BUS scan is started through flipping (ON - OFF) last dip switch. Note that BUS power  (+24v) is removed when no devices are detected. In case of issues, you may find useful to "jump start" the device using the +24V provision of other connectors of the motor control board.
  * New HW version: with newer HW versions, the bus scan is carried out using the LC display in menu 37. For more see: [Supramatic 4 Busscan](https://www.tor7.de/news/bus-scan-beim-supramatic-serie-4-errorcode-04-avoid)

# HCPBridge MQTT (HomeAssistant topics)

This is just a quick and dirty implementation and needs refactoring, but it is working.
Using the Shutter Custom Card (from HACS) it is also possible to get a representation of the current position of the door, and slide it to custom position (through set_position MQTT command).

The switch to put garage in venting position wors with a small hack. Based on the analyses of dupas.me the motor should gave a status 0A in venting position. As this was not the case the variable VENT_POS in hciemulator.h was default with value '0x08' which correspond to the position when my garage door is in venting position (position available under ***http://[deviceip]/status***). When the door is stopped in this position the doorstate is set as venting.

![image](https://user-images.githubusercontent.com/14005124/215218504-bddf65e2-6c88-4d0a-83bd-de3cacb63c88.png)
![alt text](Images/HA.png)
