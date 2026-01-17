# Integrating Legacy 345MHz Honeywell Security Hardware with Home Assistant

## Overview

This guide documents how to repurpose old Honeywell security sensors (344.975MHz) with Home Assistant after a control panel failure, using a software-defined radio (SDR) USB dongle to receive sensor signals directly.  While the software and Software Defined Radio typically uses 433Mhz, you can set the radio to only listen on 345Mhz, or you can frequency hop for a defined number of seconds on different frequencies.

## Background

After my Honeywell Lynx 5200 control panel died following a storm and power surge, I discovered I could continue using my existing wireless sensors without replacing them. By using an RTL-SDR USB dongle and the rtl_433 add-on, I'm able to receive status updates from all my door/window contacts and motion sensors directly in Home Assistant. As a bonus, I can also monitor temperature and weather data from other frequencies by having rtl_433 hop between frequencies.

<div align=center>
<img width="335" height="419" alt="image" src="https://github.com/user-attachments/assets/0fd82527-8228-4dd0-8e86-af5ec7271d1e" />
</div>

## Hardware Requirements

- **SDR Dongle**: [Nooelec RTL-SDR v5](https://www.amazon.com/dp/B01GDN1T4S) (or compatible RTL2832U-based dongle)
- **Existing Sensors**:
  - Honeywell/2Gig door/window contacts (DW10/DW11) - uses 345Mhz
  - Honeywell motion sensors (5800PIR) - uses 345Mhz
- **Optional Thermo-Hygrometers for Weather** Any other 433.92MHz compatible sensors, if you choose to frequency hop between 433 & 345
  - Example: LaCrosse TX141TH-BV2
  - Example: Ambient Weather F007TH

## Software Requirements

### Home Assistant Add-ons

1. **Mosquitto broker** - MQTT broker for message handling
2. **rtl_433** - Software to decode radio signals
3. **rtl_433 MQTT Auto Discovery** - Automatic sensor discovery

### Home Assistant Integrations

- [MQTT Integration](https://www.home-assistant.io/integrations/mqtt)

### Additional Tools (Optional)

- **MQTT Explorer** - Useful for debugging and discovering sensor topics

## Installation Steps

### 1. Install Required Add-ons

1. Navigate to **Settings → Add-ons → Add-on Store** in Home Assistant
2. Install the following add-ons in order:
   - Mosquitto broker
   - rtl_433
   - rtl_433 MQTT Auto Discovery

### 2. Configure Mosquitto Broker

1. Start the Mosquitto broker add-on
2. Create MQTT user credentials (default: `mqtt`/`mqtt`)
3. Enable "Start on boot"

### 3. Configure MQTT Integration

1. Go to **Settings → Devices & Services → Add Integration**
2. Search for and add "MQTT"
3. Configure with your Mosquitto broker details (usually `localhost:1883`)

### 4. Configure rtl_433

Create or edit the rtl_433 configuration file with the following settings:

```conf
output mqtt://[IP_of_HAOS]:1883,user=mqtt,pass=mqtt,retain=true
report_meta time:iso:usec:tz

# Frequency hopping configuration
frequency 433.92M   # For temperature and weather sensors
frequency 344.975M  # For Honeywell security sensors
hop_interval 120    # Switch frequencies every 120 seconds

# Output preferences
convert si

# Protocol filters (optional - speeds up processing)
# protocol 8   # LaCrosse TX Temperature/Humidity Sensor
# protocol 20  # Ambient Weather F007TH, TFA 30.3208.02, SwitchDocLabs F016TH temperature sensor
protocol 70  # Honeywell Door/Window Sensor, 2Gig DW10/DW11, RE208 repeater
# protocol 73  # LaCrosse TX141-Bv2, TX141TH-Bv2, TX141-Bv3, TX141W, TX145wsdth, (TFA, ORIA) sensor
```

**Note**: Replace `[IP_of_HAOS]` with your Home Assistant IP address, or use `localhost` if running on the same machine.  Uncomment whatever protocol you what to try to listen for.  The full list of 200+ is on the documentation for rtl_433.  For example I had some weather sensors that report back.  Be careful not to turn on too many protocols as the data collection may not cycle through all the devices before it hops to the next frequency.  You can also get too much data if you enable some of the TPMS protocols, for example.  While in concept it might be nice to find and narrow down TPMS sensors reporting tire pressure on your car, it will also collect data on every car that pulls into your driveway with the same brand TPMS which creates a lot noise in your logs.

### 5. Start rtl_433 Add-on

1. Start the rtl_433 add-on
2. Check the logs to ensure it's receiving signals
3. Enable "Start on boot"

## Sensor Discovery and Identification

### Using MQTT Explorer

1. Install and open MQTT Explorer
2. Connect to your Mosquitto broker
3. Navigate to `rtl_433/[device-id]/devices/Honeywell-Security/`
4. You should see a channel number (e.g., `8`) and then device IDs

### Identifying Individual Sensors

To determine which sensor ID corresponds to which physical sensor:

1. **Trigger each sensor** - Open/close doors or trigger motion sensors
2. **Watch MQTT Explorer** for state changes under the Honeywell-Security topic
3. **Physical identification**:
   - Remove the sensor cover (triggers tamper alert)
   - Watch which device ID shows the tamper event
   - Document the ID for that sensor location
4. Use process of elimination for remaining sensors

### Example MQTT Topic Structure

```
rtl_433/
  └── 17069798-rtl433/
      ├── events
      └── devices/
          └── Honeywell-Security/
              └── 8/                    # Channel number
                  ├── 125809/           # Device ID
                  │   ├── time
                  │   ├── id
                  │   ├── channel
                  │   ├── event
                  │   ├── state
                  │   ├── contact_open
                  │   ├── reed_open
                  │   ├── alarm
                  │   ├── tamper
                  │   ├── battery_ok
                  │   ├── heartbeat
                  │   └── mic
                  ├── 527088/
                  └── ...
```

## Home Assistant Configuration

### Configuration.yaml

Add the following to include your MQTT sensors:

```yaml
mqtt: !include mqtt.yaml
```

### mqtt.yaml

Create this file with your sensor definitions:

```yaml
sensor:
  # Door/Window Sensors
  - state_topic: "rtl_433/17069798-rtl433/devices/Honeywell-Security/8/527088/state"
    name: sensor.honeywell_security_8_527088_frontdoor
    value_template: "{{ value }}"
    qos: 1

  - state_topic: "rtl_433/17069798-rtl433/devices/Honeywell-Security/8/125809/state"
    name: sensor.honeywell_security_8_125809_backdoor
    value_template: "{{ value }}"
    qos: 1

  - state_topic: "rtl_433/17069798-rtl433/devices/Honeywell-Security/8/27976/state"
    name: sensor.honeywell_security_8_27976_patiodoor
    value_template: "{{ value }}"
    qos: 1

  - state_topic: "rtl_433/17069798-rtl433/devices/Honeywell-Security/8/840271/state"
    name: sensor.honeywell_security_8_840271_garagedoor
    value_template: "{{ value }}"
    qos: 1

  # Door sensor using reed_open attribute
  - state_topic: "rtl_433/17069798-rtl433/devices/Honeywell-Security/8/688378/reed_open"
    name: sensor.honeywell_security_8_688378_bedroomdoor
    value_template: >
      {% if value == '1' %}
        open
      {% else %}
        closed
      {% endif %}
    qos: 1

  # Motion Sensor (ID: 396729)
  - state_topic: "rtl_433/17069798-rtl433/devices/Honeywell-Security/8/396729/event"
    name: sensor.honeywell_security_8_396729_kitchenmotion
    value_template: >
      {% if value == '128' %}
        Motion Detected
      {% else %}
        No Motion
      {% endif %}
    qos: 1
```

**Important**: 
- Replace the device ID in the topic path with your actual device IDs
- Adjust the state topic paths based on which attribute provides reliable state changes
- Some sensors may use `state`, others `reed_open` or `contact_open`

## Dashboard Example

### Lovelace Card Configuration

```yaml
type: vertical-stack
cards:
  - type: entities
    entities:
      - entity: sensor.sensor_honeywell_security_8_125809_backdoor
        name: Back Deck Door
        secondary_info: last-changed
        icon: mdi:door
      - entity: sensor.sensor_honeywell_security_8_27976_patiodoor
        name: Screened Porch Door
        secondary_info: last-changed
        icon: mdi:door
      - entity: sensor.sensor_honeywell_security_8_527088_frontdoor
        name: Front Door
        secondary_info: last-changed
        icon: mdi:door
      - entity: sensor.sensor_honeywell_security_8_840271_garagedoor
        name: House Garage Door
        secondary_info: last-changed
        icon: mdi:door
      - entity: sensor.sensor_honeywell_security_8_396729_kitchenmotion
        name: Kitchen Motion Sensor
        secondary_info: last-changed
        icon: mdi:motion-sensor
      - entity: sensor.sensor_honeywell_security_8_688378_bedroomdoor
        name: Master Bedroom Deck Door
        secondary_info: last-changed
        icon: mdi:door
    title: Lock / Door Status Last Updated
```

## Optional: Wall-Mounted Control Panel

For a dedicated Home Assistant control interface, you can set up an inexpensive Android tablet as a wall-mounted control panel.

### Hardware

- **Tablet**: Inexpensive Android tablet or Amazon Fire Tablet
  - *Note*: Fire Tablets work well but require a one-time fee to remove lock screen ads
- **Wall Mount**: [Tablet Wall Mount](https://www.amazon.com/dp/B0BYCNTQV1)
- **Power Management**: 
  - USB-C cable fed through the wall
  - TP-Link Kasa Smart Plug EP25 (or similar smart plug)
- **Software**: 
  - [Fully Kiosk Browser](https://www.fully-kiosk.com/) with PLUS License ($14 one-time)

### Setup Steps

1. **Tablet Preparation**
   - Factory reset the tablet for a clean start
   - Create a basic user account specifically for the panel
   - Install Fully Kiosk Browser app
   - Purchase and activate Fully PLUS license

2. **Fully Kiosk Configuration**
   - Set your Home Assistant URL as the start page
   - Enable kiosk mode to hide navigation bars
   - Configure motion detection to wake screen (optional)
   - Set screen brightness and timeout preferences

3. **Power Management Setup**
   - Mount the tablet on the wall
   - Run USB-C cable through the wall to the smart plug
   - Configure smart plug in Home Assistant

4. **Battery Preservation Automation**

Create an automation to manage charging and preserve battery life:

```yaml
automation:
  - alias: "Control Panel - Start Charging"
    trigger:
      - platform: numeric_state
        entity_id: sensor.control_panel_battery_level
        below: 20
    action:
      - service: switch.turn_on
        target:
          entity_id: switch.control_panel_charger

  - alias: "Control Panel - Stop Charging"
    trigger:
      - platform: numeric_state
        entity_id: sensor.control_panel_battery_level
        above: 80
    action:
      - service: switch.turn_off
        target:
          entity_id: switch.control_panel_charger
```

### Home Assistant Integration

You can integrate the tablet with Home Assistant using either:

1. **Fully Kiosk Browser Integration**
   - Go to Settings → Devices & Services → Add Integration
   - Search for "Fully Kiosk Browser"
   - Enter tablet IP address and Fully Kiosk password
   - Provides sensors for battery level, motion detection, screen state, etc.

2. **Home Assistant Companion App**
   - Install the Home Assistant mobile app on the tablet
   - Provides device sensors and notification capabilities
   - Can be used alongside Fully Kiosk Browser

### Benefits

- Always-on dashboard display
- Battery management extends tablet lifespan
- Motion detection can wake the display
- Can display camera feeds, sensor status, and controls
- Relatively inexpensive compared to commercial panels

### Credit

Setup inspiration from [@EverythingSmartHome](https://www.youtube.com/@EverythingSmartHome) on YouTube.

## Troubleshooting

### No Sensors Appearing

1. **Check rtl_433 logs** - Ensure the add-on is receiving signals
2. **Verify antenna connection** - Make sure the SDR dongle antenna is properly connected
3. **Check frequency** - Confirm your sensors operate on 433.92MHz
4. **Test sensor battery** - Low battery can cause weak signals

### Intermittent Updates

1. **Antenna placement** - Move the SDR dongle to a more central location
2. **USB interference** - Try a different USB port or use a USB extension cable
3. **Frequency hopping** - If using multiple frequencies, you may miss some updates during frequency switches. Adjust `hop_interval` as needed.

### MQTT Connection Issues

1. **Verify Mosquitto is running** - Check the add-on status
2. **Check credentials** - Ensure MQTT username/password are correct in rtl_433 config
3. **Test with MQTT Explorer** - Confirm you can connect and see messages

### Wrong State Values

- Different sensors may publish state information in different attributes (`state`, `reed_open`, `contact_open`)
- Monitor MQTT Explorer to see which attribute changes when you trigger the sensor
- Adjust your `state_topic` accordingly

## Advanced Tips

### Binary Sensors

For better integration with Home Assistant, consider converting these to binary sensors:

```yaml
binary_sensor:
  - platform: mqtt
    name: "Front Door"
    state_topic: "rtl_433/17069798-rtl433/devices/Honeywell-Security/8/527088/state"
    payload_on: "open"
    payload_off: "closed"
    device_class: door
```

### Battery Monitoring

Monitor sensor battery levels:

```yaml
sensor:
  - state_topic: "rtl_433/17069798-rtl433/devices/Honeywell-Security/8/527088/battery_ok"
    name: "Front Door Battery"
    value_template: "{% if value == '1' %}OK{% else %}Low{% endif %}"
```

### Automations

Example automation for door notifications:

```yaml
automation:
  - alias: "Front Door Opened"
    trigger:
      - platform: state
        entity_id: sensor.sensor_honeywell_security_8_527088_frontdoor
        to: "open"
    action:
      - service: notify.mobile_app
        data:
          message: "Front door opened"
```

## Benefits of This Approach

- ✅ **Cost-effective** - Reuse existing sensors instead of buying new ones
- ✅ **No monthly fees** - Unlike traditional security systems
- ✅ **Full control** - All data stays local
- ✅ **Expandable** - Can monitor other 433MHz devices (weather stations, temperature sensors, etc.)
- ✅ **Integration** - Full Home Assistant automation capabilities

## Limitations

- ⚠️ **One-way communication** - Cannot send commands back to sensors
- ⚠️ **No encryption** - 433MHz signals are unencrypted
- ⚠️ **Range dependent** - SDR placement affects reception quality
- ⚠️ **Frequency hopping** - Multi-frequency monitoring may miss some updates

## Resources

- [rtl_433 Documentation](https://github.com/merbanan/rtl_433)
- [Home Assistant MQTT Integration](https://www.home-assistant.io/integrations/mqtt)
- [RTL-SDR Blog](https://www.rtl-sdr.com/)

## License

This documentation is provided as-is for educational purposes. Use at your own risk.

---

**Last Updated**: January 2026
