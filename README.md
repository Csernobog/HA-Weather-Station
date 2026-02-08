# HA-Weather-Station
Solar - Battery powered ESP32+BME280 weather station MQTT integrated HA.
![HA WS](/pictures/weather_station.jpg)
3D Case: https://www.thingiverse.com/thing:2282869

# Part list and circuit<br />
 - XIAO ESP32-C3 Super Mini  https://www.aliexpress.com/item/1005007663345442.html
 - BME280 I2C temp, humidity, pressure sensor https://www.aliexpress.com/item/1005008511564094.html
 - MCP1700-3302E low power LDO https://www.aliexpress.com/item/1005008740983678.html
 - CN3065 Mini Solar Lithium Battery Charger Board 500mA https://www.aliexpress.com/item/1005008965334813.html
 - Solar Panel 5V 250-500 mA
 - 18650 Battery with case

![HA WS](/pictures/WS_circuit_image.png)
# ESPHOME code

Operating model: 
 - 5-minute deep-sleep cycles
 - measurement in every cycle
 - sending measurement data to the HA every 3 cycles via MQTT protocol

Base entities:
BMP280:
 - Outside temperature
 - Absolute humidity
 - Station air pressure value
 - Dew point
 - Battery voltage: </br >
   To calculate the multiplier value, you need to measure the resistance between the ADC and the + or - point of the element.
'''
multiply: 1.7  # (8.7k+12,3k)/8,7k
'''



# HA Card<br />
U need for this card:
 - card-mod / http://homeassistant.local:8123/hacs/repository/190927524
 - expander-card / http://homeassistant.local:8123/hacs/repository/677140532
 - button-card / http://homeassistant.local:8123/hacs/repository/146194325
  
 Card close:<br />
 ![HA WS card close](/pictures/ws_card_close.jpg)

 Card open:<br />
 ![HA WS card close](/pictures/ws_card_open.jpg)


