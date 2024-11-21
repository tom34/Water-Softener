# Water-Softener
## Dumb water softener with home assistant & espHome

The goal of this project is to describe how to wire a D1 Mini NodeMCU (ESP8266-12) to a water softener (CPED BiO 22 liters) and to configure espHome.


| Hardware required  |  |  |
| ------------- | ------------- | ------------- |
| D1 Mini NodeMCU (ESP8266-12)  | [link to Amazon](https://www.amazon.fr/gp/product/B01N9RXGHY/ref=pe_3044141_189395771_pd_te_s_qp_im?_encoding=UTF8&pd_rd_i=B01N9RXGHY&pd_rd_r=AZ70N9HMVFQYPZTPVFX5&pd_rd_w=o2N3j&pd_rd_wg=VCi3Y)  | ![](https://github.com/tom34/Water-Softener/blob/33341fb78fcdb5e3516713293c75eb1e442d207a/pics-small/NodeMCU%20-%20D1%20Mini-XS.png)|
| 4 Octocoupled relays board  | [link to Amazon](https://www.amazon.fr/gp/product/B078Q8S9S9/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1) | ![](https://github.com/tom34/Water-Softener/blob/c4f95d90308fbb6db4f89fb76a1948137767a7ac/pics-small/4%20relays%20module-XS.png)|


## How softener work ? Basic description 

A softener use 3 solenoid valves to operate, each valve is dedicated to a specfic function:
* SV1 : decompression (EV3 sur desc.)
* SV2 : moving train  (EV1 sur desc.)
* SV3 : brine suction (EV2 sur desc.)

```                                                                                 ┏━━━━━━━━━━━━┓   ━Open
                                                                                 ┃            ┃
 ━┫SV1┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━...━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    1min    ┗━  ━Closed
                                                                                 :
                                                                                 : 
          ┏━━━━━━━━━━━━━━━━━━━━━━━━━...━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓                ━Open
          ┃                                                                      ┃
 ━┫SV2┣━━━┛<                            Regeneration                            >┗━━━━━━━━━━━━━━━ ━Closed 
          :                                                                      : 
          :                                                                      :
          ┏━━━━━━━━━━━━┓                              ┏━━━━━━━━━━━━━━━━━━━━━━━━━━┓                ━Open
          ┃            ┃                              ┃                          ┃
 ━┫SV3┣━━━┛    1min    ┗━━━━━━━━━━━━...━━━━━━━━━━━━━━━┛           2min           ┗━━━━━━━━━━━━━━━ ━Closed 
          |< Backwash >|< Brine suction & Slow Rince >|<       Quick Rince      >|   
```
## Wiring 

![](https://github.com/tom34/Water-Softener/blob/main/pics-large/Wiring%20ESP32%20Relays%20-%20XL.png)


## espHome

```
#    ┏━━━━━━━━━━━━━┓
#    ┃   ON_BOOT   ┃
#    ┗━━━━━━━━━━━━━┛
esphome:
  name: compteur-eau
  friendly_name: compteur-eau
  on_boot:  # even if relays are declared with the correct logic (see switch section), in the event of a sudden power outtage during a regeneration
    then:   # the execution of the Regeneration_Stop script will stop the water spillage
      - script.execute: 
          id: waterSoftener_Regeneration_Stop # Reconfigure l'ensemble des ElectroVannes
          minutes: 0.20 # 10sec
      - binary_sensor.template.publish:
          id: regeneration_sensor
          state: OFF
      # lors du boot D2,D3,D4 passent HIGH => Ecoulement Egout dans ce cas donc decompression de la chambre obligatoire
      # voir si le probleme peut etre résolu en changeant les sorties
      # - switch.turn_off: relay_3
      # - switch.turn_off: relay_2
      # - switch.turn_on: relay_1 # lors du boot D2,D3,D4 passent HIGH => Ecoulement Egout dans ce cas donc decompression de la chambre obligatoire
      # - delay: 10s              # voir si le probleme peut etre résolu en changeant les sorties
      # - switch.turn_off: relay_1


substitutions:
  device_name: "Compteur d'eau"
  output_name_1: "ElectroVanne 1" # Decompression de la chambre
  output_name_2: "ElectroVanne 2" # Déplacement Train Mobile
  output_name_3: "ElectroVanne 3" # Aspiration Saumure
  regenerationDuration: '50.0' # "50.0" pour 50 min
  detassage: 1min             # 1min
  rincageRapide: 2min         # 2min
  regenerationInterval: '2500.0'
  regenerationStartTime: '1.0'  # for 01:00 AM

esp8266: 
  board: d1_mini
  restore_from_flash: True # allow restoring counters after esp boot 

preferences:
  flash_write_interval: 30min # decrease flash memory wearing 

# Enable logging
logger:
  level: VERBOSE

# Enable Home Assistant API
api:
  encryption:
    key: "xxxxxxxxxxx"

ota:
  - platform: esphome
    password: "xxxxxxxxxxx"

wifi:
  networks:
  - ssid: xxxxxx
    password: xxxxxx

  manual_ip:
    # Set this to the IP of the ESP
    static_ip: 192.168.xx.xx
    # Set this to the IP address of the router. Often ends with .1
    gateway: 192.168.xx.xx
    # The subnet of the network. 255.255.255.0 works for most home networks.
    subnet: 255.255.255.0

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "compteur-eau"
    password: "xxxxxxxxxxxx"


#    ┏━━━━━━━━━━━━━━━━━━━━━┓
#    ┃      Variables      ┃
#    ┗━━━━━━━━━━━━━━━━━━━━━┛
globals:
  - id: regInterval
    type: double
    restore_value: no
    initial_value: ${regenerationInterval}

  - id: regStartTime
    type: double
    restore_value: no
    initial_value: ${regenerationStartTime}

  - id: softenedWater_total
    type: double
    restore_value: yes # restore water counter after boot

  - id: softenedWater_flow
    type: double
    restore_value: no

  - id: last_regeneration
    type: long long int
    restore_value: yes # restore regeneration last date after boot
    initial_value: '33616800'

#                                                                                                                        ┏━━━━━━━━━━━━━┓       ━Open
#                                                                                                                        ┃             ┃
# ━━━┫EV1 Decompression┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╍┅┅━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛     1min    ┗━━━━━  ━Closed (EV3 sur desc.)
#                                                                                                                        ╎
#                                                                                                                        ╎
#                          ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╍┅┅━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓                     ━Open
#                          ┃                                                                                             ┃
# ━━━┫EV2-Moving Train ┣━━━┛                                    Regeneration                                             ┗━━━━━━━━━━━━━━━━━━━━ ━Closed (EV1 sur desc.)
#                          ╎                                                                                             ╎
#                          ╎                                                                                             ╎
#                          ┏━━━━━━━━━━━━━┓                                                    ┏━━━━━━━━━━━━━━━━━━━━━━━━━━┓                     ━Open
#                          ┃             ┃                                                    ┃                          ┃
# ━━━┫EV3-Brine suction┣━━━┛    1min     ┗━━━━━━━━━━━━━━━━━━━━━━╍┅┅━━━━━━━━━━━━━━━━━━━━━━━━━━━┛           2min           ┗━━━━━━━━━━━━━━━━━━━━ ━Closed (EV2 sur desc.)
#                          ╎<  Backwash >╎<             Brine suction + Slow Rince           >╎<       Quick Rince      >╎      


#    ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━┳━━━━━━━┳━━━━━━━┳━━━━━━━┓
#    ┃ Pressure - BAR                   ┃  2,5  ┃  3,0  ┃  3,5  ┃  4,0  ┃
#    ┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━╋━━━━━━━╋━━━━━━━╋━━━━━━━┫
#    ┃ Regeneration duration - minutes  ┃  62   ┃  52   ┃  42   ┃  41   ┃
#    ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┻━━━━━━━┻━━━━━━━┻━━━━━━━┻━━━━━━━┛
#
#    ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━┳━━━━━━━━┳━━━━━━━━┳━━━━━━━━┳━━━━━━━━┳━━━━━━━━┳━━━━━━━━┓
#    ┃ Water hardness - °f                 ┃   20°  ┃   25°  ┃   30°  ┃   35°  ┃   40°  ┃   45 ° ┃   50°  ┃   60°  ┃
#    ┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━╋━━━━━━━━╋━━━━━━━━╋━━━━━━━━╋━━━━━━━━╋━━━━━━━━╋━━━━━━━━╋━━━━━━━━┫
#    ┃ Interval btw regeneration - liters  ┃  5500  ┃  4200  ┃  3650  ┃  3100  ┃  2750  ┃  2450  ┃  2200  ┃  1800  ┃
#    ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┻━━━━━━━━┻━━━━━━━━┻━━━━━━━━┻━━━━━━━━┻━━━━━━━━┻━━━━━━━━┻━━━━━━━━┻━━━━━━━━┛
#      ↳ Water Softener CPED BiO (22 liters)  


#    ┏━━━━━━━━━━━━━━━━━━━━━━━━━┓
#    ┃   Interval subroutine   ┃
#    ┗━━━━━━━━━━━━━━━━━━━━━━━━━┛
interval:
  - interval: 60s
    then:
      # ESP_LOGD("custom", ">>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> softenedWater_total is: %f", id(softenedWater_total)); 
      - lambda: |-
          if (id(softenedWater_total) > id(regInterval) ) {
            if ((id(homeassistant_time).now().hour + id(homeassistant_time).now().minute/60.0) > id(regStartTime) && (id(homeassistant_time).now().hour + id(homeassistant_time).now().minute/60.0 < (id(regStartTime) +3.0))) {
              id(softenedWater_total) = 0.0;
              id(regeneration).turn_on();
             }
          }


#    ┏━━━━━━━━━━━━━┓
#    ┃   SCRIPTS   ┃
#    ┗━━━━━━━━━━━━━┛
script:
  - id: waterSoftener_Regeneration
    parameters:
      minutes: float # regeneration duration
    then:

      - binary_sensor.template.publish:
          id: regeneration_sensor
          state: ON

      # ensure that all valves are set to positions: EV2+EV3=OFF with EV1=ON for X minutes then EV1=ON / final state is EV1+EV2+EV3=OFF
      - script.execute: 
          id: waterSoftener_Regeneration_Stop # Reconfigure l'ensemble des ElectroVannes
          minutes: 0.20 # 10sec

      # regeneration starting
      - switch.turn_on: relay_2

      # backwash / detassage  
      - switch.turn_on: relay_3
      - delay: ${detassage}
      - switch.turn_off: relay_3

      # regeneration duration in minutes
      - delay: !lambda return minutes*60000; 
          # unite ms : x 60000

      # quick rinse
      - switch.turn_on: relay_3
      - delay: ${rincageRapide}

      - script.execute:
          id: waterSoftener_Regeneration_Stop # EV2+EV3=OFF EV1=ON 1'
          minutes: 1.0 # decompression duration 1 minute
  
      # RAZ du compteur eau adoucie 
      - lambda: |-
          id(softenedWater_total) = 0.0;
          id(last_regeneration) = id(homeassistant_time).now().timestamp;
 
      - binary_sensor.template.publish:
          id: regeneration_sensor
          state: OFF

  - id: waterSoftener_Regeneration_Stop
    parameters:
      minutes: float # decompression duration
    then:
      - switch.turn_off: relay_3
      - switch.turn_off: relay_2
      - switch.turn_on: relay_1 
      - delay: !lambda return minutes*60000; 
          # unite ms : x 60000              
      - switch.turn_off: relay_1

#    ┏━━━━━━━━━━━━━┓
#    ┃   SWITCHS   ┃
#    ┗━━━━━━━━━━━━━┛
switch:
  # >>>>>>>> Régéneration
  - platform: template
    name: "Régéneration"
    id: "regeneration"
    icon: "mdi:water-sync"
    turn_on_action:
      - script.execute:
          id: waterSoftener_Regeneration
          minutes: ${regenerationDuration}
    turn_off_action:
      - script.execute:
          id: waterSoftener_Regeneration_Stop
          minutes: 1.0    

  # >>>>>>>> Decompression de la chambre
  - platform: gpio
    id: "relay_1" 
    name: "${output_name_1}" 
    pin: D2
    inverted: yes
    icon: "mdi:electric-switch"
    restore_mode: ALWAYS_ON           # Due to relay wiring and as inverted is TRUE restore mode must be set to ALWAYS_ON
  #----------------------------------------
  # >>>>>> Déplacement Train Mobile
  - platform: gpio
    id: "relay_2"
    name: "${output_name_2}" 
    pin: D3
    inverted: yes
    icon: "mdi:electric-switch"
    restore_mode: ALWAYS_ON           # Due to relay wiring and as inverted is TRUE restore mode must be set to ALWAYS_ON
  #----------------------------------------
  # >>>>>>>> Aspiration Saumure
  - platform: gpio
    id: "relay_3"
    name: "${output_name_3}" 
    pin: D4
    inverted: yes
    icon: "mdi:electric-switch"
    restore_mode: ALWAYS_ON           # Due to relay wiring and as inverted is TRUE restore mode must be set to ALWAYS_ON
  
#    ┏━━━━━━━━━━━━━┓
#    ┃   SENSORS   ┃
#    ┗━━━━━━━━━━━━━┛
sensor:
#    ┏━━━━━━Water counter━━━━━━┓
  - platform: pulse_counter
    state_class: measurement
    name: "water_flow_meter"
    id: water_flow_meter
    pin: D1
    update_interval: 1s
    filters:
    - lambda: return (x / 400.0); #Flow pulse: F=(6.68Q)±5% with Q=L/min
    unit_of_measurement: "L/min"

  - platform: integration
    device_class: water
    state_class: total_increasing
    name: "total en m³"
    unit_of_measurement: 'm³'
    accuracy_decimals: 4
    sensor: water_flow_meter
    time_unit: min
    icon: "mdi:water"
    filters:
        - lambda: return (x / 1000);

  - platform: total_daily_energy
    name: "total du jour"
    power_id: water_flow_meter
    unit_of_measurement: "L"
    accuracy_decimals: 2
    id: daily_total_eau
    filters:
        - lambda: return (x * 60);

  - platform: integration
    device_class: water
    state_class: total_increasing
    name: "total en litres"
    unit_of_measurement: 'L'
    accuracy_decimals: 2
    sensor: water_flow_meter
    time_unit: min
    icon: "mdi:water"

#    ┏━━━━━━Softened Water counter━━━━━━┓
  - platform: pulse_counter # Reed sensor between GPIO and GND
    pin:
      number: D6
      inverted: true
      mode:
        input: true
        pullup: true
    unit_of_measurement: 'L/min'
    name: 'Débit Eau Adoucie'
    internal_filter: 100ms
    update_interval: 1s 
    count_mode:
      rising_edge: DISABLE
      falling_edge: INCREMENT
    filters:
      # valeur precedente 1770.0 -2% / 1804.0 +3.5%
       - lambda: |-
            id(softenedWater_total) += x/1782.0; 
            id(softenedTotal).publish_state(id(softenedWater_total));
            id(softenedWater_flow) = x/40;
            return x/40 ;
 
  - platform: template
    id: softenedTotal
    name: "Total eau Adoucie" 
    icon: "mdi:water-opacity"
    unit_of_measurement: "L"
    accuracy_decimals: 2
    lambda: return (id(softenedWater_total));
  
  - platform: template
    id: consumption
    name: "Consommation avant régéneration" #consumption before regeneration 
    icon: "mdi:water-percent"
    unit_of_measurement: "%"
    accuracy_decimals: 1
    lambda: return ((id(softenedWater_total)/id(regInterval))*100.0);

  # Uptime sensor.
  - platform: uptime
    name: Uptime
  
  # WiFi Signal sensor.
  - platform: wifi_signal
    name: WiFi Signal
    update_interval: 60s

text_sensor:
  - platform: template
    name: 'Dernière régéneration'
    icon: mdi:calendar-clock
    lambda: |-
      char str[17];
      strftime(str, sizeof(str), "%d.%m.%Y %H:%M", localtime(&id(last_regeneration)));
      return  {str};

binary_sensor:
  - platform: template
    name: "Régéneration en cours"
    id: regeneration_sensor

# Enable time component to reset energy at midnight
time: # Get the time from Home Assistant to sync the onboard real-time clock.
  - platform: homeassistant
    id: homeassistant_time

```

