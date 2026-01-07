# Titon MVHR ESPHome Integration

[![ESPHome](https://img.shields.io/badge/ESPHome-2025.12.1-blue)](https://esphome.io/)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-2025.12.1-blue)](https://www.home-assistant.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Control and monitor your Titon MVHR (Mechanical Ventilation with Heat Recovery) unit via Home Assistant using ESPHome and a D1 Mini with RS-485 adapter.

## Disclaimer
This project/repository is something I have put together in my limited spare time. I am using AI initially to generate the logic in the YAML config file and when possible, I am reviewing the code to interrogate the logic being applied and testing the functionality is working as it should. While I welcome issues and will do my best to answer in my spare time, I am encouraging others to contribute to the YAML file and make this a better project for all that may wish to use it!

## Features

- **Temperature Monitoring**: Read all three thermistors (Stale In, Stale Out, Fresh In)
- **Humidity Monitoring**: Internal humidity sensor readings
- **Fan Speed Control**: Switch between Speed 1 (Trickle), Speed 2 (Normal), Speed 3 (Boost), and Speed 4 (Purge)
- **Summer Bypass & SUMMERboost**: Control summer ventilation modes
- **Boost Inhibit (Night Mode)**: Prevent high-speed operation during quiet hours
- **Status Monitoring**: Engine running status, filter remaining time, runtime hours
- **Error Detection**: Monitor for fan errors, thermistor errors, EEPROM errors, and more
- **Heat Recovery Efficiency**: Calculated efficiency based on temperature readings (a work in progress!)

## Background & introduction

Our story begins in August 2022 when I [posted on the Home Assistant forum](https://community.home-assistant.io/t/heat-recovery-mvhr-integration-titon-beam-in-ireland-mechanical-ventilation-with-heat-recovery/454942/22) on whether anyone has an integration for Titon (or sold as Beam in Ireland) Mechanical Ventilation with Heat Recovery (MVHR).
I didn't have much responses, other than one or two others looking for similar. I managed to get a command line to the unit via a USB to RS485 adaptor and was able to understand some of the concepts using a manual from the vendor.

After much playing around with it via direct serial, I found the serial bus always crashing (within Home Assistant) if there was a clash with data coming from the MVHR's Auramode Controller, so I decided in early 2025 to go with a D1 Mini and a TTL (UART) to RS485 Module.
Its been a busy year (2025) and I had done very little with it... but its a new year (Jan 2026 that I write this), so here's take two!

Here we go! ...

## Hardware (and other) Requirements

| Component | Description |
|-----------|-------------|
| D1 Mini (ESP8266) | Main microcontroller - I used a generic model from AliExpress which worked fine |
| UART to RS-485 Converter | For serial communication with MVHR - again another AliExpress purchase|
| Titon MVHR Unit | Compatible with BMS protocol - the Model I have is Titon HRV10M Q+ (TP481B) the B being the key distinction or TPxxxB/BC/BE Units |
| 2-wire cable | Connection between RS-485 converter and MVHR J9 port, mounted on the top of the Titon MVHR unit (few screws need to be remoted |
| Home Assistant | To get real value out of this, I assume you are using Home Assistant... if not, I highly recommend you do so |
| ESPHome Device Builder Add-on for Home Assistant | For this project, I am using the ESPHome Device Builder Add-on for Home Assistant |
| 3d Printed Case | There are a number of 3D printed cases available which house both the D1 Mini and the RS485 board. I have listed 2 in the next section below |

### 3D Case (to house D1 mini and UART to RS-485 Converter)
I have not printed either of these yet but upon initial review, I suggest the following:
 - [DIN rail mounting Case (via Printables)](https://www.printables.com/model/405333-esp-wemos-d1-mini-and-rs485-ttl-adapter-mounting-c)
 - [Non-DIN mounted Case (via Maker World)](https://makerworld.com/en/models/510708-wemos-d1-esp32-case-for-esphome-for-deye#profileId-426791)

### Wiring Diagram
Below is an image for reference (its from a different product repurposed). At a later stage, I will try and upload a specific image of how it all looks when connected to the Titon MHVR unit. On the D1 mini, you can probably use different GPIO pins but I suggest sticking with the below so you can copy the code verbatim (I think there was a reason using these GPIO pins previously but need to confirm).

[![Diagram](https://github.com/rosscullen/titon-mvhr-esphome/blob/main/images/rs485-ttl-d1-mini-titon-mvhr-controller-schematics.jpg)]

```
D1 Mini          RS-485 Converter          Titon MVHR (J9)
---------        ----------------          ---------------
GPIO1 (TX) ----> TX
GPIO3 (RX) <---- RX
3.3V       ----> VCC
GND        ----> GND
                 A  ------------------>    A
                 B  ------------------>    B
```

> **Note**: The 12V terminal on the Titon MHVR unit board typically powers the units display controllers (such as the aurastat controller that I use). You only need the A & B terminals to your RS485 board (A to A, and B to B) - do not directly connect this 12V to try and power the D1 Mini. There is probably a way to power the D1 mini from the MVHR unit but that is out of scope for this project. For now, i'm just poweing it using a USB cable and an old usb phone adaptor.

## Note regarding existing displays or controllers
I use the Titon Auramode Controller (others are available including the auralite HRV but I would find that the RS485 bus would be busy with the polling (chattering) between it and the BMS controller, causing clahes with data sending/receiving, as I believe its only half-duplex. I would find that observing the bus using a USB to RS485 adapter, shows a lot of chatter **SO I HIGHLY RECOMMEND DISCONNECTING THE EXISTING DISPLAY CONTROLLER** to ensure you have reliability. That's not to say that you may wish to plug it your existing controller in every now and than. 

## Communication Protocol

The Titon MVHR uses a proprietary RS-485 protocol (but note it does not use Modbus):

| Parameter | Value |
|-----------|-------|
| Baud Rate | 1200 |
| Data Bits | 8 |
| Parity | None |
| Stop Bits | 1 |
| Handshaking | None |

### Command Format

Below are some technical details (thanks to Titon BMS Manual) on how the system communicates with the Titon BMS Controller. You don't need to worry too much about this in order to get going, it just might be useful for those who want to understand the 'how' and 'why', and who may wish to contribute/fork this repository at a later stage.
**Read Command (12 bytes):**
```
AAA1+00000\r\n
│││││└────── Data (ignored for reads)
││││└─────── Polarity (+/-)
│││└──────── Read/Write (1=read, 0=write)
└└└───────── Address (000-999)
```

**Response (11 bytes):**
```
AAA+DDDDD\r\n
│││││└────── Data value (5 digits)
││││└─────── Polarity (+/-)
└└└───────── Address echo
```

### Available Addresses

| Address | Function | Notes |
|---------|----------|-------|
| 030 | Thermistor 1 (Stale In) | 0.1°C resolution (e.g., +00250 = 25.0°C) |
| 031 | Thermistor 2 (Stale Out) | 0.1°C resolution |
| 032 | Thermistor 3 (Fresh In) | 0.1°C resolution |
| 036 | Internal Humidity | 1% RH resolution |
| 060 | Runtime Hours | Total hours |
| 061 | Status Word | 16-bit status flags |
| 068 | Factory Reset | Write 21930 to reset |
| 151 | Speed 1 Switch | Write 1=ON, 0=OFF |
| 152 | Speed 3 Switch | Write 1=ON, 0=OFF |
| 154 | Speed 4 Switch | Write 1=ON, 0=OFF |
| 230 | Summer Bypass | Write 1=Enable, 0=Disable |
| 290 | SUMMERboost | Write 0=Enable, 1=Disable |
| 326 | Boost Inhibit | Write 1=Enable, 0=Disable |
| 341 | Filter Remaining | Hours until filter change |

### Status Word (Address 061)

The status word is a 16-bit value where each bit indicates a specific status:

| Bit | Decimal | Status |
|-----|---------|--------|
| 0 | 1 | Supply Fan Error |
| 1 | 2 | Thermistor Error |
| 2 | 4 | Extract Fan Error |
| 3 | 8 | EEPROM Error |
| 4 | 16 | Switch 1 Active |
| 5 | 32 | Switch 2 Active |
| 6 | 64 | Switch 3 Active |
| 7 | 128 | LS1 Active |
| 8 | 256 | LS2 Active |
| 9 | 512 | Engine Error |
| 10 | 1024 | Switch Error |
| 11 | 2048 | Engine Running |
| 12 | 4096 | Thermistor 1 Error |
| 13 | 8192 | Thermistor 2 Error |
| 14 | 16384 | Thermistor 3 Error |
| 15 | 32768 | Humidity Sensor Error |

> **Example**: A value of 2048 means engine running, no errors. A value of 2049 means engine running + supply fan error.

## Installation

### 1. upload and compile the YAML code (refer to YAML file in this repository) 
Upload YAML to your D1 mini microcontroller using the ESPhome Device Builder Add-on for Home Assistant.
There are plenty of guides in setting up ESPhome devices such as the D1 Mini online or via the ESPHome website. I would suggest creating a new device via the ESPHome Device Builder menu item in Home Assistant. Once you have it established with basic code and connectivity, then edit the device and copy/Paste in the code into the newly created device. Be sure to tweak it to your specific details (hostname, wifi etc).

### 2. Create Secrets File

With any ESPHome project, I recommend creating a `secrets.yaml` file in your ESPHome directory, so your not always exposing your WiFi SSID and Password directly in your project YAML file:

```yaml
wifi_ssid: "your_wifi_ssid"
wifi_password: "your_wifi_password"
```

### 3. Compile and Upload
Once everything is reviewed and auto validated in ESPHome, upload your new code (wirelessly or directly via USB) to your D1 mini as per the regular upload process (again, if you haven't done this before, there are numerous online tutorials which walk you through the process).

### 4. Add to Home Assistant

The device will be automatically discovered by Home Assistant. Navigate to **Settings → Devices & Services → ESPHome ** in order to view and configure your new Titon MVHR controller using ESPHome and Home Assistant! 

## Configuration

### Basic Configuration

The YAML configuration included in this Github respository includes some of the following which you can tweak accordingly, but I recommend the below settings for simplicity:

```yaml
esphome:
  name: mvhr
  friendly_name: Titon MVHR

esp8266:
  board: d1_mini

uart:
  id: uart_mvhr
  tx_pin: GPIO1
  rx_pin: GPIO3
  baud_rate: 1200
  data_bits: 8
  parity: NONE
  stop_bits: 1
```

### Available default Entities

#### Sensors
- Stale Air In Temperature
- Stale Air Out Temperature
- Fresh Air In Temperature
- Internal Humidity
- Runtime Hours
- Filter Remaining Time
- Heat Recovery Efficiency
- Status Word (raw)
- WiFi Signal Strength
- Uptime

#### Binary Sensors
- Filter Change Required
- Engine Running
- Supply Fan Error
- Extract Fan Error
- Thermistor Errors (1, 2, 3)
- Humidity Sensor Error
- EEPROM Error
- Engine Error

#### Switches
- Fan Speed 1 (Trickle)
- Fan Speed 3 (Boost)
- Fan Speed 4 (Purge)
- Boost Inhibit (Night Mode)

#### Selects
- Fan Speed (dropdown selector) - this needs work. Note that Fan Speed 2 typicall isn't an option as it's usually the default state. i.e. if you were in "Fan speed 3" and you turn this switch off, it will go to what is Fan Speed 2.

#### Buttons
- Poll Sensors Now
- Factory Reset MVHR

## Summer Bypass & SUMMERboost

### Summer Bypass
Diverts incoming fresh air around the heat exchanger to bring in cooler outside air (bypasses heat recovery). Useful for cooling your home with cooler air during the summer. I cannot say how effective this function is in general on a MVHR unit but better than nothing I guess!

### SUMMERboost
Runs both supply and extract fans at high speed (Speed 4) for maximum fresh air exchange. Only activates when Summer Bypass is active.

## Fan Speed Priority

The MVHR uses a "fastest speed wins" priority system:
- If an external boost switch holds the unit at Speed 3, setting Speed 1 via serial will have no effect
- The highest requested speed from any source will be active


## Whats not working (so far)
- I find that when I press the boost button around our house, the project isn't updating the sensor of current speed. I need to investigate this further.
- The filter remaining schedule - i'm not convinced mine is accurate. I need to investigate further whether the "remaining days" value is something that is managed from the MVHR BMS controller/mainboard, or it is something managed on the Auramode (and similar) control panel.

There are other items which probably are not required but will review accordingly. There are lots of comments left within the YAML file to explain what each section relates to.

## Troubleshooting

### No Response from MVHR

1. **Check wiring**: Ensure A/B lines are correctly connected (try swapping if no response). Suggest using the logging (details below further to enable logging) so you can see what commands are being sent and received. The YAML file is configured to ensure these messages are presented in a human readable format for ease of diagnosing issues.
2. **Verify baud rate**: Must be exactly 1200
3. **Check for bus contention**: If using aurastat or other controller, it has continuous communication - polling may conflict

### Error Response (999/-99999)

This indicates an invalid address or communication error. Verify:
- Address exists in the protocol table
- Command format is exactly 12 bytes
- CR+LF terminators are present
- In my MVHR unit, it transpires that 2 of my temperature sensors were responding with -99999 (via the ESPHome logs - reply messages). The guys on the Titon tech team suggested that I temporarily fitted a 10K resistor in the Temperature positions on the titon PCB (which should return a response of roughly “031+00250" or around 25 Degrees)... and they were right. Which indicated to me that my unit currently has 2 faulty temperature sensors :-(

### Intermittent Communication

- The aurastat controller (and others i'm sure) have almost continuous communication with the MVHR. Again I recommend disconnecting to ensure more reliable performance.
- If you insist on having it connected, you may need to Implement delays between commands to avoid bus contention (I need to check code again to see if I left some of this logic in from previous versions, where I tried to run both in parallel)
- Consider longer polling intervals (5+ seconds)

### Debug Logging

Enabling debug logging in your ESPHome YAML file allows you to see raw communication - i.e. messages being sent to Titon BMS controller and responses being sent (which are formatted in an easy to read format):

```yaml
logger:
  level: DEBUG
  logs:
    uart: DEBUG
    mvhr: DEBUG
    mvhr_rx: DEBUG
```

## Home Assistant Automations
Below are an initial set of Home Assistant Automations to leverage the power of having Home Assistant integrated with your Titon/Beam MVHR system.

### Example: Night Mode Schedule
This will ensure that the system does not turn up the speed during the night and not disturb your sleep.

```yaml
automation:
  - alias: "MVHR Night Mode"
    trigger:
      - platform: time
        at: "22:00:00"
    action:
      - service: switch.turn_on
        entity_id: switch.boost_inhibit_night_mode

  - alias: "MVHR Day Mode"
    trigger:
      - platform: time
        at: "07:00:00"
    action:
      - service: switch.turn_off
        entity_id: switch.boost_inhibit_night_mode
```

### Example: Humidity-Based Boost
In this example, if your humidity goes above 70, fan speed 3 (boost) will kick in.

```yaml
automation:
  - alias: "MVHR High Humidity Boost"
    trigger:
      - platform: numeric_state
        entity_id: sensor.internal_humidity
        above: 70
    action:
      - service: switch.turn_on
        entity_id: switch.fan_speed_3_boost
```

### Example: Filter Replacement Notification

```yaml
automation:
  - alias: "MVHR Filter Alert"
    trigger:
      - platform: state
        entity_id: binary_sensor.filter_change_required
        to: "on"
    action:
      - service: notify.mobile_app
        data:
          title: "MVHR Filter Change Required"
          message: "Your MVHR filter needs replacing"
```

## Contributing

Contributions are very much welcomed! I hope there are others out there looking to do similar with their Titon/Beam MVHR system, therefore please feel free to submit a Pull Request or fork this repository.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details (**to do**).

## Acknowledgments

- The Titon Technical support team who gave me a few pointers along the way when I ran into some dead ends. Also for Titon to publish details online of how their controller communicates so it can be leverages by such projects as this!
- ESPHome community for the essential building blocks to make this possible
- Home Assistant community for the amazing home automation platform that many know and love. To see how much it has grown since my first use in 2018 in mind blowing. Keep up the great work Paulus, Frenck and team.
- Thanks to the comments on the Home Assistant Community Forums for keeping me motivated to get to this point. I know it has a bit to go yet so it's a start I guess
- To https://github.com/respawner for sharing his similar project written in python, that connects directly via USB -> RS485 adapter and has a HACS integration. Check it out, it might suit your needs more.

## References

- [Home Assistant Community Forum(where this topic started)](https://community.home-assistant.io/t/heat-recovery-mvhr-integration-titon-beam-in-ireland-mechanical-ventilation-with-heat-recovery/454942/22)
- [Titon Home Assistant Integration - direct USB to HA using a USB to RS485 adaptor](https://github.com/respawner/hass_titon_controller_plugin)
- [Titon PCB Communication Protocol for BMS Manual](http://products.titon.com/wp-content/uploads/2025/03/BM893_Iss.01_PCB_Communication_Protocal_for_BS_V1_1-ENG.pdf)
- [ESPHome Documentation](https://esphome.io/)
- [Home Assistant ESPHome Integration](https://www.home-assistant.io/integrations/esphome/)

---

**Disclaimer**: This is an unofficial integration. Use at your own risk! Modifying ventilation system settings incorrectly could affect your warrenty, not to mention your indoor air quality!
