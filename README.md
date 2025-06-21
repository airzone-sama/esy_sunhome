# ESY Sunhome HA Integration

This is set up for a Linux based appliance install of HA. 

## Pre-reqs

You need some basic Linux skills. You need the MQTT Integration installed. It is ideal to have a file editor and shell as part of your HA integration, but not necessary.

I'm sure there are better ways to do this, but I did the minimal amount of effort to get my scenario to work.

## Basic principles
There is 2 main parts.. Shell scripts and HA configuration. Shell scripts interact with the web api to enact configuration updates to your battery, and HA extracts information from an MQTT stream. The trick is that the MQTT stream only broadcasts your battery's status in a digestible form when the app requests it, so we need to trick it.

Also you need MQTT filtering otherwise you're getting everyone else's battery status from the unauthenticated MQTT server.. The 'S' in IoT stands for security.... :|

## Shell Scripts (basic integration)
Create a directory in your homeassistant root directory called battery_api
(e.g. /root/homeassistant/battery_api)

Create a text file called login.txt with the following content:
### login.txt
```json
{
  "password":"xxxxxxxxxxxxxx",
  "clientId":"",
  "requestType":1,
  "loginType":"PASSWORD",
  "userType":2,
  "userName":"xxxxxxxx@xxxxxxx.xxxx"
}
```
Substitute in your own credentials, and make sure you aren't sharing those credentials with any other system.

Test your login file with the following command run from your battery_api folder
```bash
curl --header "Content-Type: application/json" --request POST --data "@login.txt" "http://esybackend.esysunhome.com:7073/login?grant_type=app"
```
It should a json reply with the code:0 value. If you get code:1, you stuffed up. Check your login credentials. 

Create the following files:

### get_inverter_id.sh
```bash
#!/bin/bash

ACCESS_TOKEN=`curl --silent --header "Content-Type: application/json" --request POST --data "@login.txt" "http://esybackend.esysunhome.com:7073/login?grant_type=app" | jq -r '.data.access_token'`

INVERTER_ID=`curl --silent --header "Content-Type: application/json" --header "Authorization: bearer $ACCESS_TOKEN" --request GET "http://esybackend.esysunhome.com:7073/api/lsydevice/page?current=1&size=1" | jq -r '.data.records.[].id'`

echo $INVERTER_ID
```

### transmit_mqtt.sh
```bash
#!/bin/bash

ACCESS_TOKEN=`curl --silent --header "Content-Type: application/json" --request POST --data "@battery_api/login.txt" "http://esybackend.esysunhome.com:7073/login?grant_type=app" | jq -r '.data.access_token'`

INVERTER_ID=`curl --silent --header "Content-Type: application/json" --header "Authorization: bearer $ACCESS_TOKEN" --request GET "http://esybackend.esysunhome.com:7073/api/lsydevice/page?current=1&size=1" | jq -r '.data.records.[].id'`

curl --silent --header "Content-Type: application/json" --header "Authorization: bearer $ACCESS_TOKEN" --request GET "http://esybackend.esysunhome.com:7073/api/param/set/obtain?val=3&deviceId=$INVERTER_ID" 
```

### read_custom_schedule.sh
```bash
#!/bin/bash

ACCESS_TOKEN=`curl --silent --header "Content-Type: application/json" --request POST --data "@battery_api/login.txt" "http://esybackend.esysunhome.com:7073/login?grant_type=app" | jq -r '.data.access_token'`

INVERTER_ID=`curl --silent --header "Content-Type: application/json" --header "Authorization: bearer $ACCESS_TOKEN" --request GET "http://esybackend.esysunhome.com:7073/api/lsydevice/page?current=1&size=1" | jq -r '.data.records.[].id'`

curl --silent --header "Content-Type: application/json" --header "Authorization: bearer $ACCESS_TOKEN" --request GET "http://esybackend.esysunhome.com:7073/api/lsydevicechargedischarge/info?deviceId=$INVERTER_ID"
```

Make the scripts executable with a chmod 755 *.sh in the battery_api folder

This is enough for very basic integration. We can discuss advances scheduling later.

## Inverter ID

All the MQTT comms we care about have the inverter ID as part of the topic. The inverter ID is a numberic serial number that is different to the serial number on your sticker. We need this. In your shell, change to the battery_api directory and run the get_inverter_id.sh command. It will present you with the inverter ID associated with your account. 

```bash
[core-ssh homeassistant]$ cd battery_api
[core-ssh battery_api]$ ./get_inverter_id.sh
16xxxxxxxxxxxxxxx2
[core-ssh battery_api]$
```

## MQTT Plug-in configuration
If you are using the MQTT plugin for other things, you may need to set up an intermediate broker.

The following configuration is required for the MQTT Plugin:
Broker: 99.83..178.210
Port: 1883
Username: anything
Password: anything
Client ID: leave blank
Use a certificate: Off
Ignore Broker Certificate Validation: On
MQTT Protocol: 3.1.1
MQTT Transport: TCP

To test, you can use the following topic to listen to: /APP/Your Inverter ID here/NEWS

(Open the phone app - it should start to show communications)

The message should look like this
```json
Message 13 received on /APP/xxxxxx/NEWS at 2:53 PM:
{"val":"{\"ratedPower\":6,\"flowOne\":[0,0,0,0,0,0,0,1,0,1,0,1,1,0,0,1],\"gridPower\":0,\"loadPower\":740,\"code\":5,\"wifiStrength\":-64,\"batterySoc\":99,\"flowTwo\":[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],\"loadLine\":1,\"pv1Power\":714,\"batteryNum\":4,\"flowThree\":[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],\"pv2Power\":0,\"gridLine\":0,\"deviceId\":INVERTER_ID,\"deviceSn\":\"DEVICE_SN\",\"pvPower\":714,\"heatingState\":0,\"dailyPowerGeneration\":17.934,\"batteryStatus\":5,\"batteryLine\":1,\"systemRunStatus\":5,\"pvLine\":1,\"batteryPower\":26}","msgType":0,"valType":0,"msgId":"xxxxxxx","msgOperation":1}
QoS: 0 - Retain: false
```
If you get stuff like this, then you're good.

## configuration.yaml
Add this to your configuration.yaml file. Substitute INVERTER_ID for your inverter ID. When done, restart your HA system (not just reload). 
```yaml
shell_command:
  battery_mqtt_heartbeat: bash battery_api/transmit_mqtt.sh
  battery_get_custom_schedule: bash battery_api/read_custom_schedule.sh

homeassistant:
  customize:
    sensor.energy_to_battery:
      device_class: energy
      unit_of_measurement: kWh
    sensor.energy_from_pv:
      device_class: energy
      unit_of_measurement: kWh
    sensor.energy_from_battery:
      device_class: energy
      unit_of_measurement: kWh
    sensor.energy_from_grid:
      device_class: energy
      unit_of_measurement: kWh
    sensor.energy_to_grid:
      device_class: energy
      unit_of_measurement: kWh  
    sensor.energy_to_home:
      device_class: energy
      unit_of_measurement: kWh        

sensor:
  - platform: filter
    name: "Home Battery Export Rolling Average"
    entity_id: sensor.home_battery_battery_export
    unique_id: INVERTER_ID_BERA
    filters:
      - filter: time_simple_moving_average
        window_size: "00:15"
        precision: 0
mqtt:
  sensor:
    - name: State of Charge
      state_topic: "/APP/INVERTER_ID/NEWS"
      unit_of_measurement: "%"
      unique_id: INVERTER_ID_SOC
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.batterySoc }}
        {% endif %}
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"        
    - name: Grid Power
      state_topic: "/APP/INVERTER_ID/NEWS"
      unit_of_measurement: "W"
      device_class: power
      unique_id: INVERTER_ID_GP
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.gridPower }}
        {% endif %}        
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"       
    - name: Grid Import
      state_topic: "/APP/INVERTER_ID/NEWS"
      unit_of_measurement: "W"
      device_class: power
      unique_id: INVERTER_ID_GIMP
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {% if nested_json.gridLine == 2 %}
        {{ nested_json.gridPower }}
        {% else %}
        0
        {% endif %}
        {% endif %}        
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"   
    - name: Grid Export
      state_topic: "/APP/INVERTER_ID/NEWS"
      unit_of_measurement: "W"
      device_class: power
      unique_id: INVERTER_ID_GEXP
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {% if nested_json.gridLine == 1 %}
        {{ nested_json.gridPower }}
        {% else %}
        0
        {% endif %}
        {% endif %}        
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"  
    - name: Load Power
      state_topic: "/APP/INVERTER_ID/NEWS"
      unit_of_measurement: "W"
      unique_id: INVERTER_ID_LP
      device_class: power
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.loadPower }}
        {% endif %}        
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"               
    - name: PV Power
      state_topic: "/APP/INVERTER_ID/NEWS"
      unit_of_measurement: "W"
      device_class: power
      unique_id: INVERTER_ID_PP
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.pvPower }}
        {% endif %}           
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"               
    - name: Battery Power
      state_topic: "/APP/INVERTER_ID/NEWS"
      unit_of_measurement: "W"
      device_class: power
      unique_id: INVERTER_ID_BP
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.batteryPower }}
        {% endif %}              
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"               
    - name: Battery Import
      state_topic: "/APP/INVERTER_ID/NEWS"
      unit_of_measurement: "W"
      device_class: power
      unique_id: INVERTER_ID_BIMP
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {% if nested_json.batteryLine == 2 %}
        {{ nested_json.batteryPower }}
        {% else %}
        0
        {% endif %}
        {% endif %}        
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"   
    - name: Battery Export
      state_topic: "/APP/INVERTER_ID/NEWS"
      unit_of_measurement: "W"
      device_class: power
      unique_id: INVERTER_ID_BEXP
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {% if nested_json.batteryLine == 1 %}
        {{ nested_json.batteryPower }}
        {% else %}
        0
        {% endif %}
        {% endif %}        
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"             
    - name: Grid Active
      state_topic: "/APP/INVERTER_ID/NEWS"
      unique_id: INVERTER_ID_GA
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.gridLine }}
        {% endif %}        
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"               
    - name: Load Active
      state_topic: "/APP/INVERTER_ID/NEWS"
      unique_id: INVERTER_ID_LA
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.loadLine }}
        {% endif %}       
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"               
    - name: PV Active
      state_topic: "/APP/INVERTER_ID/NEWS"
      unique_id: INVERTER_ID_PA
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.pvLine }}
        {% endif %}          
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"               
    - name: Battery Active
      state_topic: "/APP/INVERTER_ID/NEWS"
      unique_id: INVERTER_ID_BA
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.batteryLine }}
        {% endif %}       
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"               
    - name: Schedule Mode
      state_topic: "/APP/INVERTER_ID/NEWS"
      unique_id: INVERTER_ID_SM
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {% if nested_json.code == 5 %}
        Custom Schedule
        {% elif nested_json.code == 1 %}
        Normal Schedule
        {% else %}
        Unknown Schedule
        {% endif %} 
        {% endif %}  
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"               
    - name: Heater State
      state_topic: "/APP/INVERTER_ID/NEWS"
      unique_id: INVERTER_ID_HS
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.heatingState }}
        {% endif %}        
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"               
    - name: Battery Status
      state_topic: "/APP/INVERTER_ID/NEWS"
      unique_id: INVERTER_ID_BS
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.batteryStatus }}
        {% endif %}        
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"               
    - name: System Run Status
      state_topic: "/APP/INVERTER_ID/NEWS"
      unique_id: INVERTER_ID_RS
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.systemRunStatus }}
        {% endif %} 
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"               
    - name: Daily Power Generation
      state_topic: "/APP/INVERTER_ID/NEWS"
      unit_of_measurement: "kWh"
      unique_id: INVERTER_ID_PG
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.dailyPowerGeneration }}
        {% endif %} 
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"               
    - name: Rated Power
      state_topic: "/APP/INVERTER_ID/NEWS"
      unit_of_measurement: "kW"
      unique_id: INVERTER_ID_RP
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {{ nested_json.ratedPower }}
        {% endif %}       
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"              
    - name: Battery Status Text
      state_topic: "/APP/INVERTER_ID/NEWS"
      unique_id: INVERTER_ID_BST
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 0 %}
        {% set nested_json = value_json.val|replace("\\", "")|from_json %}
        {% if nested_json.batteryStatus == 0 %}
        Idle
        {% elif nested_json.batteryStatus == 1 %}
        Charging
        {% elif nested_json.batteryStatus == 5 %}
        In Use
        {% else %}
        Unknown {{nested_json.batteryStatus}}
        {% endif %} 
        {% endif %}  
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"                  
    - name: Inverter Temperature
      state_topic: "/APP/INVERTER_ID/NEWS"
      unit_of_measurement: "C"
      unique_id: INVERTER_ID_IT
      value_template: >
        {% if value_json.msgType == 0 and value_json.valType == 7 %}
        {% set nested_json = ((value_json.val|replace("\\", ""))[1:(nested_json|length-1)])|from_json %}
        {{ nested_json.dataList[5].dataList[0].val }}
        {% endif %}       
      device:
        name: "Home Battery"
        identifiers:
          - "hb_INVERTER_ID"                
```
**Observations:** The return data is json nested in json. This was the way I could figure out how to deal with it. If you monitor the return data using an MQTT explorer, you can find out other parameters that you may find interesting, such as SOH, cell balancing, heater status, etc. It's up to you.

## MQTT Heartbeat
As alluded to earlier, your MQTT stream will only operate while the phone app is open. Your phone app calls a webapi that causes your system to start transmitting MQTT for a short period. We need to hijack this. 

Add an automation with the following properties:

**When:** Triggers every 15 seconds of every minute of every hour. 

**Then do:** Perform action 'shell_command.battery_mqtt_heartbeat'

Turn on the automation. You should see it run every 15 seconds or so.

## The end?
You now have enough basic functionality to show the current battery status (charging / discharging / idle), state of charge, scheduling mode, and power from the current transformers. You also can see the power flow parameters. 

Power flows are shown discretely - i.e. import and export are different flows.

But that's not really enough to do much with. 



## Adding to a energy meter
By default, you will be getting power. This works fine in power_flow_card_plus to show your immediate power flows. But some people may want to see energy. 

You will need to add a helper sensor of the integral type for each power flow named:
- Energy To Battery
- Energy From PV
- Energy From Battery
- Energy From Grid
- Energy To Grid
- Energy To Home

Use the corresponding power sensor (e.g. for Energy To Battery, use Battery Import). Use Left Riemann Sum calculation, a precision of 2, and a sub-interval of 1 minute. Once you have all 6 Integral sensors defined, you can add this to the energy dashboard.


## Advanced scheduling

You should already know the app provides the following modes:
- Regular Mode
- Emergency Mode
- Electricity Sale Mode
- Battery Energy Management

Battery Energy Management allows you to set a custom schedule to buy, use, and sell grid power. You can have multiple times per section. This is useful if you have time-of-use metering. e.g. my metering schedule is as follows:
- 00:00 - 05:59: Cheap c/kwh
- 06:00 - 10:59: Expensive c/kwh
- 11:00 - 13:59: Free c/kwh
- 14:00 - 23:59: Expensive c/kwh

I therefore set my mode to "Battery Energy Management" and a custom schedule to "buy" power from 11am until 2pm, with the intent to use it through the day and night. Depending on many factors, I will run out of power before the next free period. Therefore I want to have the option of ensuring that come 6am, my battery is always at a fixed SoC so it has enough energy to run through the morning peak. My default percentage is 30%, but I increase that to 40% if it's raining, or 50% if it's cold and I have my heater set to run in the early AM. So how to interact with this capability?

### Step 1 - Understanding the custom schedule format
Configure the approximate schedule in your BEM mode. 


