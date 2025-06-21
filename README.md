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


## Advanced scheduling with anti-EV "integration"

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

I also have an EV and although will usually charge during the Free period, if I need a top-up for tomorrow morning, I will charge in the cheap period starting from 00:00. I don't want the battery to dump it's energy into the car - it's inefficient. So I also want it to detect when my EVSE starts to charge the car, and to shut off power delivery until it's finished. My EVSE is a Tesla Wall Connector G3 and I already have it integrated in my HA.

I therefore set my mode to "Battery Energy Management" and a custom schedule to "buy" power from 11am until 2pm, with the intent to use it through the day and night. Depending on many factors, I will run out of power before the next free period. Therefore I want to have the option of ensuring that come 6am, my battery is always at a fixed SoC so it has enough energy to run through the morning peak. My default percentage is 30%, but I increase that to 40% if it's raining, or 50% if it's cold and I have my heater set to run in the early AM. So how to interact with this capability?

### Step 1 - Understanding the custom schedule format
Configure the approximate schedule in your BEM mode. 

Then use the HA shell to run the command from your HA root directory:
```bash
[core-ssh homeassistant]$ battery_api/read_custom_schedule.sh
{json stuff returned here}
[core-ssh homeassistant]$
```
Grab the JSON output and put it in a json pretty formatter (can use online if you like - I use [JSON Formatter](https://jsonformatter.org/json-pretty-print)). It will resemble this:
```json
{
  "updateTime": xxxxxxx,
  "chargeTimeQuantum": [
    {
      "end": 839,
      "sort": 0,
      "start": 660
    },
    {
      "end": 359,
      "sort": 1,
      "start": 330
    }
  ],
  "deviceId": "INVERTER_ID",
  "chargeCutOff": 100,
  "dischargeCutOff": 0,
  "dischargeTimeQuantum": [],
  "releaseSwitch": 1,
  "createTime": xxxxxxx,
  "dischargeSwitch": 0,
  "releaseTimeQuantum": [
    {
      "end": 330,
      "sort": 0,
      "start": 0
    },
    {
      "end": 659,
      "sort": 1,
      "start": 360
    },
    {
      "end": 1439,
      "sort": 2,
      "start": 840
    }
  ],
  "id": "xxxxxxxx",
  "releaseCutOff": 5,
  "chargeSwitch": 1
}
```
The Charge Time Quantum is what I want to modify. You can see there's one with Sort:1 that starts at 330 and ends at 359. This number is the number of minutes past midnight - i.e. 05:30 - 05:59. So by modifying this start value we can adjust when the battery will start charging in the off-peak power rates. Also by removing this section of json, we can prevent it from charging in the morning (e.g. if the battery has more than enough charge). Also to stop it from discharging into my EV, I also care about the releaseTimeQuantum where the Sort:0 starts at 0 (and can adjust this to start later). Otherwise releaseSwitch can also be changed to 0 to prevent it from discharging. Many ways to skin the cat. Discharge Time Quantum is for selling energy to the grid, which is something I don't do, but you can consider it. UpdateTime and createTime are just unix time formats. 

Save a copy of this file in the battery_api folder with the name **schedule_custom_charge.txt**. Also copy the file to **schedule_no_charge.txt** in the battery folder and remove the chargeTimeQuantum related to the early morning charge, like this:
```json
{
  "updateTime": xxxxxxx,
  "chargeTimeQuantum": [
    {
      "end": 839,
      "sort": 0,
      "start": 660
    }
  ],
  "deviceId": "INVERTER_ID",
  "chargeCutOff": 100,
  "dischargeCutOff": 0,
  "dischargeTimeQuantum": [],
  "releaseSwitch": 1,
  "createTime": xxxxxxx,
  "dischargeSwitch": 0,
  "releaseTimeQuantum": [
    {
      "end": 330,
      "sort": 0,
      "start": 0
    },
    {
      "end": 659,
      "sort": 1,
      "start": 360
    },
    {
      "end": 1439,
      "sort": 2,
      "start": 840
    }
  ],
  "id": "xxxxxxxx",
  "releaseCutOff": 5,
  "chargeSwitch": 1
}
```
n.b. Please use files generated from your own system. 

### Step 2 - Creating a custom scheduler script

My intention is to take 2 arguments from HA - if / when the battery should start charging in the morning, and what time the battery should start to discharge after midnight. If the first argument is 0, then there should be no pre-charging in the morning. 

Create a script called **set_custom_schedule_variable_charge.sh** in the battery_api directory
```bash
#!/bin/bash

#echo $1 > args.txt

# Check arguments
if [ "$#" -ne 2 ] ; then
  echo "error: Needs 2 arg"
  exit 1
fi

# Check it's a number
re='[0-9]+$'
if ! [[ $1 =~ $re ]] ; then
  echo "Error: not a number"
  exit 1
fi
re='[0-9]+$'
if ! [[ $2 =~ $re ]] ; then
  echo "Error: not a number"
  exit 1
fi

# Check it's within 0 - 358
if [ "$1" -ge 358 ] ; then
  echo "Error: Outside 0-358"
  exit 1 
fi
if [ "$1" -lt 0 ] ; then
  echo "Error: outside 0-358"
  exit 1
fi
if [ "$2" -ge 358 ] ; then
  echo "Error: Outside 0-358"
  exit 1 
fi
if [ "$2" -lt 0 ] ; then
  echo "Error: outside 0-358"
  exit 1
fi

ACCESS_TOKEN=`curl --silent --header "Content-Type: application/json" --request POST --data "@battery_api/login.txt" "http://esybackend.esysunhome.com:7073/login?grant_type=app" | jq -r '.data.access_token'`

if [ "$1" -eq 0 ] ; then
  REQUEST_JSON=`cat battery_api/schedule_no_charge.txt | tr -d '\n ' | sed -e s/START_TIME/$1/g -e s/END_TIME/$2/g -e s/\"/\\\\\"/g -e s/\:/\:\ /g`
else
  REQUEST_JSON=`cat battery_api/schedule_custom_charge.txt | tr -d '\n ' | sed -e s/START_TIME/$1/g -e s/END_TIME/$2/g -e s/\"/\\\\\"/g -e s/\:/\:\ /g`
fi

curl --silent --header "Content-Type: application/json" --header "Authorization: bearer $ACCESS_TOKEN" --request POST --data "$REQUEST_JSON" "http://esybackend.esysunhome.com:7073/api/lsydevicechargedischarge/save"
```
Make this script executable (chmod 755).


The first part of the script is sense checking the input arguments. \
The second part of the script is to get the auth token for API access \
The third part is to work out what json should be delivered to the API request. This referrences the 2 files you created earlier **schedule_no_charge.txt** and **schedule_custom_charge.txt** and performs a few substitutions to get a pretty json template file ready for a web api request. \
The last part is to submit the request.

I am using 2 arguments: **START_TIME** (the time to start pre-charging) and **END_TIME** (the time the EV charge ends).

### Step 3 - Creating the json templates

All we need to do is to set up the relevant place-holders in the 2 schedule files we created earlier. 

**schedule_custom_charge.txt**
```json
{
  "updateTime": xxxxx,
  "chargeTimeQuantum": [
    {
      "end": 839,
      "sort": 0,
      "start": 660
    },
    {
      "end": 359,
      "sort": 1,
      "start": START_TIME
    }
  ],
  "deviceId": "xxxxxx",
  "chargeCutOff": 100,
  "dischargeCutOff": 0,
  "dischargeTimeQuantum": [],
  "releaseSwitch": 1,
  "createTime": xxxxx,
  "dischargeSwitch": 0,
  "releaseTimeQuantum": [
    {
      "end": START_TIME,
      "sort": 0,
      "start": END_TIME
    },
    {
      "end": 659,
      "sort": 1,
      "start": 360
    },
    {
      "end": 1439,
      "sort": 2,
      "start": 840
    }
  ],
  "id": "xxxxxx",
  "releaseCutOff": 5,
  "chargeSwitch": 1
}
```
In the early morning chargeTimeQuantum, change the start from a numeric value to the placeholder START_TIME \
In the early morning releaseTimeQuantum, change the start from a numeric (0) to END_TIME and end from a numeric value to START_TIME  (yeah I know...).... The release will start when the EV charge ends, and it will end releasing when it's time to start charging. 

If you don't have an EV that you need to worry about, then ignore the END_TIME bit.

**schedule_no_charge.txt**
```json
{
  "updateTime": xxxxx,
  "chargeTimeQuantum": [
    {
      "end": 839,
      "sort": 0,
      "start": 660
    }
  ],
  "deviceId": "xxxxx",
  "chargeCutOff": 100,
  "dischargeCutOff": 0,
  "dischargeTimeQuantum": [],
  "releaseSwitch": 1,
  "createTime": xxxxx,
  "dischargeSwitch": 0,
  "releaseTimeQuantum": [
    {
      "end": 659,
      "sort": 0,
      "start": END_TIME
    },
    {
      "end": 1439,
      "sort": 1,
      "start": 840
    }
  ],
  "id": "xxxxx",
  "releaseCutOff": 5,
  "chargeSwitch": 1
}
```
This file has no early morning chargeTimeQuantum, and because I want the EV integration, I update the morning releaseTimeQuantum with start: END_TIME so that it only starts to release when the EV ends it's charging session.

**Please note** Use files generated from your own system 

### Step 4 - Testing the script.
From the homeassistant root folder, run the command battery_api/set_custom_schedule_variable_charge.sh 0 0 to test that the early morning charge is removed from your battery's app schedule
```bash
[core-ssh homeassistant]$ battery_api/set_custom_schedule_variable_charge.sh 0 0
{"code":0,"msg":"Successful","data":true}
[core-ssh homeassistant]$
```
It will give you a message that the API call was successful if it worked, or it will give some messages in Chinese if it didn't. You can use Google translate to try and figure out what the message means (it will also say "Server Upgrading" in Chinese if you stuffed up the JSON payload).
Then on your app, confirm that the early morning charge has been removed from the Battery Energy Management schedule.

If successful, run the command battery_api/set_custom_schedule_variable_charge.sh 120 0 to test that the early morning charge is starts at 2am
```bash
[core-ssh homeassistant]$ battery_api/set_custom_schedule_variable_charge.sh 120 0
{"code":0,"msg":"Successful","data":true}
[core-ssh homeassistant]$
```
It will give you a message that the API call was successful if it worked, or it will give some messages in Chinese if it didn't. You can use Google translate to try and figure out what the message means (it will also say "Server Upgrading" in Chinese if you stuffed up the JSON payload).
Then on your app, confirm that the early morning charge will commence from 2am in the Battery Energy Management schedule.

If this also passes the test, you can proceed.

### Step 5 - configuration.yaml
Add this shell command to your shell_command: part of your configuration.yaml file
```json
battery_set_custom_schedule_variable_charge: bash battery_api/set_custom_schedule_variable_charge.sh {{ states('sensor.calculated_charge_minutes') }} {{ states('sensor.calculated_start_discharge_minutes') }}
```
Restart HA (reloading your config will not work).

### Step 6 - HA helper entities
We need to build several helper entities now. 

**calculated_charge_minutes**\
Type: Template sensor
Template text:
```django
{% set target_pct = 30 %}

{% if( states('sensor.weather_forecast_here_temp_max_0')|int >= 38 ) %}
    {% set target_pct = 75 %}
{% elif states('sensor.solcast_pv_forecast_forecast_today')|int < 7 %}
    {% set target_pct = 50 %}
{% elif states('sensor.solcast_pv_forecast_forecast_today')|int < 15 %}
    {% set target_pct = 40 %}
{% endif %}

{% if( states('input_boolean.aircon_timer') == 'on' ) %}
{% set target_pct = target_pct + 10 %}
{% endif %}


{% set current_pct = states('sensor.state_of_charge')|int %}
{% set mins_per_pct = 3.0 %}
{% set end_time = 359 %}
{% if( current_pct < target_pct ) %}
    {% set charge_mins = ((target_pct - current_pct) * mins_per_pct + 0.5)|int %}
    {{ end_time - charge_mins }}
{% else %}
    0
{% endif %}
```
- My default target is 30%
- If my local weather forecast is going to be over 38 degrees C (i.e. hot), I don't really want to charge during the day, so I will charge to 75%. It'll stop the wires from melting.
- If the PV Solar forecast will be less than 7kwh, then it's probably going to be heavy rain, so charge to 50%
- If the PV Solar forecast will be less than 15kwh, then it's probably going to be patchy medium rain, so charge to 40%
- I have another button for my aircon to run in the early morning to warm the place up. It will need extra power in the morning, so add 10% if this is set up. 

Once you have your desired SoC worked out, we need to work out what the current delta is between out current SoC and our desired SoC. e.g. if current SoC is 20% and desired is 30%, then we have a 10% shortfall. Based on experience, the ESY battery will charge at 3% per minute. So it's easy to work out the number of minutes required to charge. 

Since my pre-charge time will end at 5:59am, this works out to 359 minutes past midnight. Therefore do some basic maths to figure out when to start charging. Return this if the current SoC is lower than the target SoC, otherwise return 0 to indicate that no pre-charging is necessary.

**calculated_start_discharge_minutes**\
Type: template sensor
Template text:
```django
{% set is_car_charging = (states('sensor.tesla_wall_connector_status') == 'charging') or (states('sensor.tesla_wall_connector_status') == 'charging_reduced') %}
{% if is_car_charging %}
  {% if ( now().hour >= 0 and ((now().hour == 5 and now().minute <= 30) or now().hour < 5 ) ) %}
    {% set start_battery_discharge = (now().hour * 60) + now().minute + 15 %}
    {% set calculated_charge_minutes = states('sensor.calculated_charge_minutes') | int %}
    {% if ((start_battery_discharge + 5) >= calculated_charge_minutes) and (calculated_charge_minutes > 0) %}
      {% set start_battery_discharge = calculated_charge_minutes - 5 %}
    {% endif %}
    {% if start_battery_discharge < 0 %}
      {% set start_battery_discharge = 0 %}
    {% endif %}
  {% else %}
    {% set start_battery_discharge = 0 %}
  {% endif %}
{% else %}
  {% set start_battery_discharge = 0 %}
{% endif %}
{{ start_battery_discharge }}
```
This works with a Tesla Wall Connector G3, but can be adapted to anything really.

This logic will look to see if it's charging and prompt the battery to only start discharging in now + 15 minutes, provided that is before 5:30am. However if a pre-charge is going to occur within the next 5 minutes, then allow it to turn back on as normal.

If you don't have an EV, you can just return 0.

### Step 7 - Automation schedule
Add an automation with the following properties:

When: Triggers every 5 minutes of every hour.

And if: If the time is after 12:00am and before 5:45am

Then do: Perform action 'shell_command.battery_set_custom_schedule_variable_charge'

Turn on the automation. Running it manually during the day probably isn't useful, but you can see how it goes for you.

## Time to Empty
It's useful to know how long it's likely going to be until you run out of juice. Add the following helper (you need the helpers above as dependancies)

**Time to Empty**\
Type: template sensor
Template text:
```django
{% set current_pct = states('sensor.state_of_charge')|int %}
{% set total_wh = 20500 * 0.95 %}
{% set current_wh = (total_wh * current_pct / 100)|int %}
{% set current_power = states('sensor.home_battery_export_rolling_average')|int %}
{% set hours_to_empty = current_wh / current_power %}
{% set mins_to_empty = (hours_to_empty * 60)|int %}
{% if ( (current_power == 0) or (states('sensor.battery_status') in ["0","1"]) ) %}
  {% if ( now().hour >= 0 and now().hour < 6 ) %}
    {% set charge_mins = states('sensor.calculated_charge_minutes') %}
    {% set discharge_mins = states('sensor.calculated_start_discharge_minutes') %}
    {% if discharge_mins == 0 %}
      {% set start_hour = (charge_mins|int / 60)|int %}
      {% set start_min = (charge_mins|int) - ( start_hour * 60 ) %}
      Charge starting at {{ '{:02}:{:02}'.format(start_hour, start_min) }}
    {% else %}
      {% set start_hour = (discharge_mins|int / 60)|int %}
      {% set start_min = (discharge_mins|int) - ( start_hour * 60 ) %}
      Suspended Until {{ '{:02}:{:02}'.format(start_hour, start_min) }}
    {% endif %}      
  {% else %}
    Charging or Idle
  {% endif %}
{% elif mins_to_empty > 60 %}
  {{ hours_to_empty|int }} hrs {{ mins_to_empty - (hours_to_empty|int * 60) }} mins
{% else %}
  {{ mins_to_empty }} mins
{% endif %}
```
This will return a nice text message.

## Example screen shots

### Main console
![Screen shot 1](/esy_ha_integration_1.png)

### Stats display
![Screen shot 2](/esy_ha_integration_2.png)

### Energy console
![Screen shot 3](/esy_ha_integration_3.png)
