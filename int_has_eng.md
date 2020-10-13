# Integration with Home-Assistant

The SLS Gateway can be easily integrated with the [Home Assistant](www.home-assistant.io) home automation system. For integration, the software [zigbee2mqtt](https://www.zigbee2mqtt.io) can be used together with various versions of zigbee dongles, or the ready-made Smart Logic System (SLS) Zigbee BLE gateway.

![koridor](/img/koridor.png)


# Preparatory activities

The module works through MQTT.
Install mosqutto on raspberry or linux:

[link 1](https://robot-on.ru/articles/ystanovka-mqtt-brokera-mosquitto-raspberry-orange-pi)

[link 2](https://smartideal.net/ustanovka-i-zapusk-mqtt-brokera-mosquitto/)

Mosqutto for windows can be downloaded [here](https://mosquitto.org/download/)





# Discovery
Discovery mode automatically adds new devices to the system. Devices in HAS are created when the state is first recorded. If the devices were paired before enabling this setting, it is necessary to re-pair.

You can enable Discovery mode from the ZIgbee-> Config menu (checkbox Home Assistant MQTT Discovery)

![int_has_disc](/img/int_has_disc.png)




# Manually adding devices
SLS Zigbee Gateway devices can be manually added to the Home-Assistant. To do this, add the appropriate settings for the device type to the configuration.yaml configuration file. Below are tested examples of clippings from the configuration file:

*The following examples show the ZigBeeCA20 starting topic - replace it with yours, which you specified in the MQTT settings of the SLS gateway. 


### Leakage Sensor (binary_sensor) SJCGQ11LM
```
- platform: mqtt
  name: bathroom_leak
  availability_topic: "/ZigBeeCA20/bridge/state"
  state_topic: "/ZigBeeCA20/bathroom_leak_1"
  value_template:> -
    {% if value_json.water_leak == true%}
      {{'ON'}}
    {% else%}
      {{'OFF'}}
    {% endif%}

# Leak Sensor # 1 (charge level) SJCGQ11LM
- platform: mqtt
  name: bathroom_leak_1_battery
  icon: mdi: battery-high
  unit_of_measurement: "%"
  availability_topic: "/ZigBeeCA20/bridge/state"
  state_topic: "/ZigBeeCA20/bathroom_leak_1"
  value_template: "{{value_json.battery}}"
```
### Temperature / humidity sensor (xiaomi round, normal sensor) WSDCGQ01LM
```
- platform: mqtt 
# Temperature
  name: bathroom_temperature
  icon: mdi: thermometer
  unit_of_measurement: "° C"
  availability_topic: "/ZigBeeCA20/bridge/state"
  state_topic: "/ZigBeeCA20/bathroom_sensor"
  value_template: "{{value_json.temperature | round (2)}}"
- platform: mqtt # Humidity
  name: bathroom_humidity
  icon: mdi: water-percent
  unit_of_measurement: "%"
  availability_topic: "/ZigBeeCA20/bridge/state"
  state_topic: "/ZigBeeCA20/bathroom_sensor"
  value_template: "{{value_json.humidity | round (2)}}"
- platform: mqtt # Charge level
  name: bathroom_sensor_battery
  icon: mdi: battery-high
  unit_of_measurement: "%"
  availability_topic: "/ZigBeeCA20/bridge/state"
  state_topic: "/ZigBeeCA20/bathroom_sensor"
  ```
  
### Square sensor with pressure (in addition to the previous one) WSDCGQ11LM:
```
- platform: mqtt # Pressure
  name: loggia_pressure
  icon: mdi: gauge
  unit_of_measurement: "mmHg"
  availability_topic: "/ZigBeeCA20/bridge/state"
  state_topic: "/ZigBeeCA20/loggia_sensor"
  value_template: "{{(value_json.pressure | float * 7.501) | round | int}}"
```
### Square xiaomi button (binary_sensor) WXKG11LM
```
- platform: mqtt
  name: bathroom_button
  availability_topic: "/ZigBeeCA20/bridge/state"
  state_topic: "/ZigBeeCA20/bathroom_button"
  value_template:> -
    {% if value_json.click == ''%}
      {{'OFF'}}
    {% else%}
      {{'ON'}}
    {% endif%}
  expire_after: 5
 ```
A note on a button - since this is a button, not a switch, binary_sensor changes its state for a very short time. To work with it, you can use automation such as this (in this case, when pressed, the fan turns on / off):
 
 
 
### Gateway backlight (light)
```
- platform: mqtt
  name: gateway
  availability_topic: "/ZigBeeCA20/bridge/state"
  command_topic: "/ZigBeeCA20/led"
  rgb_command_topic: "/ZigBeeCA20/led"
  rgb_command_template:> -
    {
      "mode": "manual",
      "hex": "# {{'% 02x% 02x% 02x' | format (red, green, blue)}}"
    }
  on_command_type: "brightness"
  payload_off: '{"mode": "off"}'
```

### Attribute Gateway Status (binary_sensor)
```
- platform: mqtt
  name: sls_state
  unique_id: cee1d05d-205a-4334-b257-723540c5d578
  state_topic: "ZigBeeGW/bridge/state"
  device_class: connectivity
  payload_on: online
  payload_off: offline
  json_attributes_topic: "ZigBeeGW/bridge/config"
  json_attributes_template: "{{value_json | tojson}}"

```
### ZigBee / Bluetooth gateway pairing mode (switch)
```
- platform: mqtt
  name: gateway_join
  availability_topic: "/ZigBeeCA20/bridge/state"
  state_topic: "/ZigBeeCA20/bridge/config"
  value_template: "{{value_json.permit_join}}"
  state_on: true
  state_off: false
  command_topic: "/ZigBeeCA20/bridge/config/permit_join"
  payload_on: "true"
  payload_off: "false"
```
### Gateway uptime (sensor)
```
- platform: mqtt
  name: gateway_uptime
  icon: mdi: timeline-clock
  unit_of_measurement: "%"
  availability_topic: "/ZigBeeCA20/bridge/state"
  state_topic: "/ZigBeeCA20/bridge/config"
  value_template: "{{value_json.UptimeStr}}"
```
! [permit] (/img/permit.png)


### Two-channel relay xiaomi switch LLKZMK11LM

```
### boiler
- platform: mqtt
  name: gas_boiler
  availability_topic: "/ZigBeeCA20/bridge/state"
  state_topic: "/ZigBeeCA20/gas_heating"
  value_template: "{{value_json.state_l1}}"
  command_topic: "/ZigBeeCA20/gas_heating/set/state_l1"
  

### floor heating pump switch

- platform: mqtt
  name: warm_floor
  availability_topic: "/ZigBeeCA20/bridge/state"
  state_topic: "/ZigBeeCA20/gas_heating"
  value_template: "{{value_json.state_l2}}"
  command_topic: "/ZigBeeCA20/gas_heating/set/state_l2"
```
respectively everywhere name and topic addresses change to your


### Turn ventilation on / off by pressing a button
```
- alias: toggle_bathroom_fan_when_button_pushed
  trigger:
    - platform: state
      entity_id: binary_sensor.bathroom_button
      to: "on"
  action:
    - service: fan.toggle
      entity_id: fan.bathroom
```

### Aqara LED Light Bulb Tunable White Model ZNLDP12LM
```
- platform: mqtt
    name: GardenBulbLeft
    state_topic: "/ZigBeeCA20/GardenBulbLeft"
    availability_topic: "/ZigBeeCA20/bridge/state"
    brightness: true
    color_temp: true
    schema: json
    command_topic: "/ZigBeeCA20/GardenBulbLeft/set"

  - platform: mqtt
    name: GardenBulbRight
    state_topic: "/ZigBeeCA20/GardenBulbRight"
    availability_topic: "/ZigBeeCA20/bridge/state"
    brightness: true
    color_temp: true
    schema: json
    command_topic: "/ZigBeeCA20/GardenBulbRight/set”
```

### Xiaomi RTCGQ11LM motion / light sensor
```
# Sensor Motion Corridor
  - platform: mqtt
    name: "Motion koridor battery"
    state_topic: "SLS/Sensor_Motion_Koridor"
    unit_of_measurement: '%'
    value_template: "{{value_json.battery}}"
  - platform: mqtt
    name: "Motion koridor linkquality"
    state_topic: "ZigBeeCA20/Sensor_Motion_Koridor"
    value_template: "{{value_json.linkquality}}"
  - platform: mqtt
    name: "Motion koridor dvigenie"
    state_topic: "ZigBeeCA20/Sensor_Motion_Koridor"
    value_template: "{{value_json.occupancy}}"
    expire_after: 10
  - platform: mqtt
    name: "Motion koridor yarkost"
    state_topic: "ZigBeeCA20/Sensor_Motion_Koridor"
    value_template: "{{value_json.illuminance}}"
    unit_of_measurement: 'lux'
```

### Xiaomi window open sensor MCCGQ01LM
```
# Sensor Door Street
  - platform: mqtt
    name: "Door Uliza battery"
    state_topic: "ZigBeeCA20/Sensor_Door_Uliza"
    unit_of_measurement: '%'
    value_template: "{{value_json.battery}}"
  - platform: mqtt
    name: "Door Uliza linkquality"
    state_topic: "ZigBeeCA20/Sensor_Door_Uliza"
    value_template: "{{value_json.linkquality}}"
  - platform: mqtt
    name: "Door Uliza"
    state_topic: "ZigBeeCA20/Sensor_Door_Uliza"
    value_template: "{{value_json.contact}}"
```


### The script sets the status of the sensors to off after the required timeout.
Create a file
```
#Victor Enot, 04/06/20 6:02 PM
# ================================================== ===========================================
# python_scripts / set_state.py
# ================================================== ===========================================
inputEntity = data.get ('entity_id')
if inputEntity is None:
    logger.warning ("===== entity_id is required if you want to set something.")
else:
    inputStateObject = hass.states.get (inputEntity)
    inputState = inputStateObject.state
    inputAttributesObject = inputStateObject.attributes.copy ()

    for item in data:
        newAttribute = data.get (item)
        logger.debug ("===== item = {0}; value = {1}". format (item, newAttribute))
        if item == 'entity_id':
            continue # already handled
        elif item == 'state':
            inputState = newAttribute
        else:
            inputAttributesObject [item] = newAttribute
        
    hass.states.set (inputEntity, inputState, inputAttributesObject)
```

In automations.yaml you need to write the following code
```
- id: '1579606187576'
  alias: Tualet pir off
  description: ''
  trigger:
  - entity_id: binary_sensor.tualet_pir
    for: 00:02:00
    platform: state
    to: 'on'
  condition: []
  action:
  - data_template:
      entity_id: Binary_sensor.tualet_pir
      state: 'off'
    service: python_script.set_state
```


      


* PS: section in development. *
