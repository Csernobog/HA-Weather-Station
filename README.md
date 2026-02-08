# HA-Weather-Station
Solar - Battery powered ESP32+BME280 weather station MQTT integrated HA.
![HA WS](/pictures/weather_station.jpg)
3D Case / THX the super modell!: https://www.thingiverse.com/thing:2282869 

# Part list and circuit<br />
 - XIAO ESP32-C3 Super Mini  https://www.aliexpress.com/item/1005007663345442.html
 - BME280 I2C temp, humidity, pressure sensor https://www.aliexpress.com/item/1005008511564094.html
 - MCP1700-3302E low power LDO https://www.aliexpress.com/item/1005008740983678.html
 - CN3065 Mini Solar Lithium Battery Charger Board 500mA https://www.aliexpress.com/item/1005008965334813.html
 - Solar Panel 5V 250-500 mA
 - 18650 Battery with case

![HA WS](/pictures/WS_circuit_image.png)
# ESPHOME code

## Operating model: 
 - 5-minute deep-sleep cycles
 - measurement in every cycle
 - sending measurement data to the HA every 3 cycles via MQTT protocol

## Base entities:
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


## Calculated entities:</br >
### Sea level air pressure:
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

### Dew point:
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
### Pressure trend: </br >
Air pressure change over a 3-hour average
```
 - platform: template
    name: "Nyomástrend (hPa/3h)"
    id: p_trend_3h
    unit_of_measurement: "hPa/3h"
    icon: "mdi:chart-line"
    accuracy_decimals: 1
    retain: true
    update_interval: never
    lambda: |-
      float p = id(p_sl).state;
      if (isnan(p)) return NAN;

      // 15 percenként frissül valójában (küldős ciklus)
      // gyors EMA: kb. "rövid táv"
      if (isnan(id(p_sl_ema))) id(p_sl_ema) = p;
      id(p_sl_ema) = 0.25f * p + 0.75f * id(p_sl_ema);

      // lassú EMA: kb. "hosszabb táv"
      if (isnan(id(p_sl_slow))) id(p_sl_slow) = id(p_sl_ema);
      id(p_sl_slow) = 0.05f * id(p_sl_ema) + 0.95f * id(p_sl_slow);

      // különbség -> trend jelleg; skálázás 3 órára (heurisztika)
      float dp = id(p_sl_ema) - id(p_sl_slow);
      return dp * 6.0f;
```
### Weather type: </br >
Weather change calculated based on pressure trend
- Rapidly rising (improving)
- Rising (improving)
- Stabile
- Decreasing (deteriorating)
- Decreasing rapidly (front/storm chance)

```
text_sensor:
  - platform: template
      name: "Időjárás jelleg (nyomás)"
      id: wx_trend
      icon: "mdi:weather-cloudy-clock"
      update_interval: never
      lambda: |-
        float tr = id(p_trend_3h).state;
        if (isnan(tr)) return {"n/a"};
  
        if (tr > 3.0f) return {"Gyorsan emelkedik (javuló)"};
        if (tr > 1.0f) return {"Emelkedik (javuló)"};
        if (tr < -3.0f) return {"Gyorsan csökken (front/vihar esély)"};
        if (tr < -1.0f) return {"Csökken (romló)"};
        return {"Stabil"};
```

### Warning: combination of pressure drop + "moist" air </br >
 - High: rapid pressure drop + humid air (chance of showers/storms)
 - Medium: pressure drop + humid air (chance of precipitation)
 - Watch out: rapid pressure drop (front may be approaching)
 - Humid: fog/humidity chance (small spread / high RH)
 - No Alert
   
```
  # Figyelmeztetés: nyomásesés + "nyirkos" levegő kombinációja
  - platform: template
    name: "Figyelmeztetés"
    id: wx_alert
    icon: "mdi:alert-outline"
    update_interval: never
    lambda: |-
      float tr = id(p_trend_3h).state;   // hPa/3h jelleg
      float t  = id(bme_t).state;
      float dp = id(dewpoint).state;
      float rh = id(bme_rh).state;

      if (isnan(tr) || isnan(t) || isnan(dp) || isnan(rh)) return {"n/a"};

      float spread = t - dp; // dewpoint depression

      // Nedvességi jelző:
      bool humid_air = (rh >= 85.0f) || (spread <= 2.0f);

      // Erős romlás jelző:
      bool fast_drop = (tr <= -3.0f);
      bool drop      = (tr <= -1.5f);

      if (fast_drop && humid_air) return {"Magas: gyors nyomásesés + nedves levegő (zápor/vihar esély)"};
      if (drop && humid_air)      return {"Közepes: nyomásesés + nedves levegő (csapadék esély)"};
      if (fast_drop)              return {"Figyeld: gyors nyomásesés (front közeleghet)"};
      if (humid_air)              return {"Nyirkos: köd/pára esély (kis spread / magas RH)"};
      return {"Nincs"};
```
### Last seen counter</br >
Heartbeat, increases by 1 in each deep-sleep cycle, and the HA side always increases by 3 (sleep-sleep-connect)</br > 
( simple diagnostics, "Have it, it works, it wasn't taken away :)" )
```
- platform: template
    name: "Last seen counter"
    id: last_seen_counter
    unit_of_measurement: "boot"
    accuracy_decimals: 0
    retain: true
    update_interval: never
    lambda: |-
      return (float) id(boot_count);
```
## MQTT Settings
Empty birth and will messages are needed so that the values ​​on the HA side are preserved even if the device is in deep sleep (otherwise everything will be unavailable during deep sleep!!)
'''
mqtt:
  broker: !secret mqtt_host
  username: !secret mqtt_user
  password: !secret mqtt_pass
  discovery: true
  discovery_retain: true

  topic_prefix: weather-station

  birth_message: 
  will_message: 
  on_message:
    - topic: weather-station/ota_mode
      payload: "ON"
      then:
        - lambda: |-
            id(ota_mode) = true;
        - wifi.enable         
        - logger.log: "OTA MODE: ON (no deep sleep)"
    - topic: weather-station/ota_mode
      payload: "OFF"
      then:
        - lambda: |-
            id(ota_mode) = false;
        - logger.log: "OTA MODE: OFF (normal deep sleep)"
'''
On the HA side, commands to turn OTA mode on and off: from MQTT broker the ON or OFF message sent to the weather-station/ota_mode topic in retain mode
![HA WS MQTT](/pictures/WS_MQTT.jpg)

# HA Card<br />
U need for this card:
 - card-mod / http://homeassistant.local:8123/hacs/repository/190927524
 - expander-card / http://homeassistant.local:8123/hacs/repository/677140532
 - button-card / http://homeassistant.local:8123/hacs/repository/146194325
  
 Card close:<br />
 ![HA WS card close](/pictures/ws_card_close.jpg)

 Card open:<br />
 ![HA WS card close](/pictures/ws_card_open.jpg)


