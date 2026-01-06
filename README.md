# Titon MVHR ESPHome Integration

[![ESPHome](https://img.shields.io/badge/ESPHome-2025.12.1-blue)](https://esphome.io/)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-2025.12.1-blue)](https://www.home-assistant.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Control and monitor your Titon MVHR (Mechanical Ventilation with Heat Recovery) unit via Home Assistant using ESPHome and a D1 Mini with RS-485 adapter.

## Disclaimer
This project is something very much in my spare time. I'm using AI initially to generate the logic in the YAML file and when possible, i'm reviewing the code to understand the logic being used and testing that the functionality is working as it should. While I welcome issues and will do my best to answer in my spare time, I am encouraging others to contribute to the YAML code and make this a better project for all that may wish to use it!

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

Our story begins in August 2022 when I posted on the Home Assistant forum ( https://community.home-assistant.io/t/heat-recovery-mvhr-integration-titon-beam-in-ireland-mechanical-ventilation-with-heat-recovery/454942/22 ) on whether anyone has an integration for Titon/Beam Mechanical Ventilation with Heat Recovery (MVHR).
I didn't have much response, other than one or two responses looking for similar. I managed to get a command line to the unit via RS485 and was able to understand some of the concepts using a manual from the vendor.

After much playing around with it via direct serial, I found the serial bus always crashing if there was a clash with data coming from the MVHR, so I decided in early 2025 to go with a D1 Mini and a  TTL (UART) to RS485 Module.
Its been a busy year (2025) and I have done very little with it... its a new year (Jan 2026 that I write this), so here's take two!

Here we go! ...

## Hardware (and other) Requirements

| Component | Description |
|-----------|-------------|
| D1 Mini (ESP8266) | Main microcontroller - I used a generic model from AliExpress which worked fine |
| UART to RS-485 Converter | For serial communication with MVHR - again another AliExpress purchase|
| Titon MVHR Unit | Compatible with BMS protocol - the Model I have is Titon HRV10M Q+ (TP481B) the B being the  |
| 4-wire cable | Connection between RS-485 converter and MVHR J9 port |
| Home Assistant | To get real value out of this, I assume you are using Home Assistant... if not, I highly recommend you do so |
| ESPHome Device Builder Add-on for Home Assistant | For this project, I'm using the SPHome Device Builder Add-on for Home Assistant |
| 3d Printed Case | There are a number of 3D printed cases available which house both the D1 Mini and the RS485 board. I will try to make recommendations here in future |

### Wiring Diagram
Below is an image for illustration purposes and following that, more specific details on the specific pins to use. At a later stage, i'll try and upload a specific image of how it all looks when connected to the Titon MHVR unit. On the D1 mini, you can probably use different pins but I suggest sticking with the below so you can copy the code verbatim (I think there was a reason using these GPIO pins but need to confirm).

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
I use the Titon Auramode Controller (others are available including the auralite HRV but I would find that the RS485 bus would be busy with the polling (chattering) between it and the BMS controller. I would find that monitoring the bus using a USB to RS485 adapter, shows a lot of chatter and CLASHES WITH THE POLLING ON THIS PROJECT - SO I DO RECOMMEND DISCONNECTING THE ESITING DISPLAY CONTROLLER to ensure 

## Communication Protocol

The Titon MVHR uses a proprietary RS-485 protocol (not Modbus):

| Parameter | Value |
|-----------|-------|
| Baud Rate | 1200 |
| Data Bits | 8 |
| Parity | None |
| Stop Bits | 1 |
| Handshaking | None |

### Command Format

Below is some technical details (thanks to Titon BMS Manual) on how the system communicates with the Titon BMS Controller. You don't need to worry too much about how this works to get started, it just might be useful for those who want to understand 'how' and 'why', and who may wish to contribute to project at a later stage.
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

### 1. upload and compile the YAML code (refer to YAML file in this repository) to your D1 mini microcontroller using the ESPhome Device Builder Add-on for Home Assistant
There are plenty of guides in setting up ESPhome devices such as the D1 Mini. Suggest creating a new device via the ESPHome Device Builder menu item in Home Assistant. Then copy/Paste the code into the newly created device. Change the basic details in the code to your own settings etc.

### 2. Create Secrets File

Create a `secrets.yaml` file in your ESPHome directory:

```yaml
wifi_ssid: "your_wifi_ssid"
wifi_password: "your_wifi_password"
```

### 3. Compile and Upload
Once your ready, upload your new code to your D1 mini as per the regular upload process (if you haven't before, lots of good tutorials online for this part)

### 4. Add to Home Assistant

The device will be automatically discovered by Home Assistant. Navigate to **Settings → Devices & Services** and configure the new ESPHome device.

## Configuration

### Basic Configuration

The YAML configuration included in this Github respository includes some of the following which you can tweak accordingly:

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

### Available Entities

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
- Fan Speed (dropdown selector) - needs work. Note that Fan Speed 2 typicall isn't an option as it's usually the default state. i.e. if you were in "Fan speed 3" and you turn this switch off, it will go to what is Fan Speed 2.

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
I find that when I press the boost button around our house, the project isn't updating the sensor of current speed. I need to investigate this further.
There are other items which probably are not required but will review accordingly. There are lots of comments left in the YAML to explain what each section relates to.

## Troubleshooting

### No Response from MVHR

1. **Check wiring**: Ensure A/B lines are correctly connected (try swapping if no response)
2. **Verify baud rate**: Must be exactly 1200
3. **Check for bus contention**: If using aurastat or other controller, it has continuous communication - polling may conflict

### Error Response (999/-99999)

This indicates an invalid address or communication error. Verify:
- Address exists in the protocol table
- Command format is exactly 12 bytes
- CR+LF terminators are present
- In my MVHR unit, it transpires that 2 of my temperature sensors were responding with -99999 (via the ESPHome logs - reply messages). The guys on the Titon tech team suggested putting a 10K resistor in the Temperature positions on the titon PCB (which should return a response of roughly “031+00250" or 25 Degrees). And they were right. Which indicated to me that my unit has 2 faulty temperature sensors :-(

### Intermittent Communication

- The aurastat® controller has almost continuous communication with the MVHR. Disconnect to ensure more reliable performance.
- Implement delays between commands to avoid bus contention (I need to check code again to see if I left some of this logic in from previous versions)
- Consider longer polling intervals (5+ seconds)

### Debug Logging

Enable debug logging to see raw communication - i.e. messages being sent to Titon BMS controller and responses being sent (which are formatted in an easy to read format):

```yaml
logger:
  level: DEBUG
  logs:
    uart: DEBUG
    mvhr: DEBUG
    mvhr_rx: DEBUG
```

## Home Assistant Automations

### Example: Night Mode Schedule - This will ensure that the system does not turn up the speed during the night and not disturb your sleep.

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

### Example: Humidity-Based Boost - in this example, if your humidity goes above 70, fan speed 3 (boost) will kick in.

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

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- Titon Ventilation Systems for the BMS communication protocol documentation
- ESPHome community for the essential building blocks to make this possible
- Home Assistant community for the amasing home automation platform. To see how much it has grown since my first use in 2018 in mind blowing. Keep up the great work Paulus, Frenck and team.
- Thanks to the comments on the Home Assistant Community Forums for keeping me motivated to get to this point.
- The Titon Technical support team who gave me a few pointers along the way when I ran into some dead ends.
- To https://github.com/respawner for sharing his progress on his project that connects directly via USB -> RS485 adapter and has a HACS integration.

## References

- Home assistant Community Forum where the topic existed. Link at https://community.home-assistant.io/t/heat-recovery-mvhr-integration-titon-beam-in-ireland-mechanical-ventilation-with-heat-recovery/454942/22
- Titon Home Assistant Integration - direct USB to HA using a USB to RS485 adaptor. Link at: https://github.com/respawner/hass_titon_controller_plugin
- Titon PCB Communication Protocol for BMS Manual - Link to manual at http://products.titon.com/wp-content/uploads/2025/03/BM893_Iss.01_PCB_Communication_Protocal_for_BS_V1_1-ENG.pdf
- [ESPHome Documentation](https://esphome.io/)
- [Home Assistant ESPHome Integration](https://www.home-assistant.io/integrations/esphome/)

---

**Disclaimer**: This is an unofficial integration. Use at your own risk. Modifying ventilation system settings incorrectly could affect your warrently, not to mention your indoor air quality!
