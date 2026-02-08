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
   To calculate the multiplier value, you need to measure the resistance between the ADC and the + or - point of the battery.
```
- platform: adc
    pin: GPIO2
    name: "Akkufeszültség"
    id: batt_v
    attenuation: 11db
    update_interval: never # csak akkor frissiti, ha meghivjuk Update-el
    retain: true
    filters:
      - multiply: 1.7  # (8.7k+12,3k)/8,7k
      - sliding_window_moving_average:
          window_size: 5
          send_every: 5
    unit_of_measurement: "V"
    accuracy_decimals: 2
```


Calculated entities:</br >
Sea level air pressure:
m = local height value ( View with GPS )
```
 - platform: template
    name: "Légnyomás (tengerszintre korrigált)"
    id: p_sl
    unit_of_measurement: "hPa"
    icon: "mdi:gauge"
    accuracy_decimals: 1
    retain: true
    update_interval: never
    lambda: |-
      float p = id(bme_p).state;   // hPa
      float t = id(bme_t).state;   // °C
      const float h = 268.0f;      // m 
      if (isnan(p) || isnan(t)) return NAN;

      float p0 = p * powf(
        1.0f - (0.0065f * h) / (t + 0.0065f * h + 273.15f),
        -5.257f
      );
      return p0;
```

Dew point:
```
- platform: template
    name: "Harmatpont"
    id: dewpoint
    unit_of_measurement: "°C"
    icon: "mdi:thermometer-alert"
    accuracy_decimals: 1
    retain: true
    update_interval: never
    lambda: |-
      float t = id(bme_t).state;
      float rh = id(bme_rh).state;
      if (isnan(t) || isnan(rh) || rh <= 0.0f) return NAN;

      // Magnus formula
      const float a = 17.67f;
      const float b = 243.5f;
      float gamma = logf(rh / 100.0f) + (a * t) / (b + t);
      return (b * gamma) / (a - gamma);
```



# HA Card<br />
U need for this card:
 - card-mod / http://homeassistant.local:8123/hacs/repository/190927524
 - expander-card / http://homeassistant.local:8123/hacs/repository/677140532
 - button-card / http://homeassistant.local:8123/hacs/repository/146194325
  
 Card close:<br />
 ![HA WS card close](/pictures/ws_card_close.jpg)

 Card open:<br />
 ![HA WS card close](/pictures/ws_card_open.jpg)


