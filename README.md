# Flashing ESPHome to a Beken/Tuya CB2S-Powered Smart Switch

This WiFi switch is powered by a Tuya/Beken CB2S module with a BK7231N MCU. It is sold in my country under the brand **Demasled**, model **DOMO-18-A2**, and is used with the **Tuya app** on Android.

Before any hardware hacking, it is recommended to configure the device using the Tuya app to ensure everything is functioning correctly.

## Hacking It for Local Use with Home Assistant

While it is possible to integrate Tuya devices with Home Assistant via the cloud, I prefer a local integration. To achieve this, I replaced the CB2S firmware with **ESPHome**. This process required reverse-engineering the connections for the relay, button, and LED.

After some experimentation with a multimeter, I determined the following connections for the CB2S module:

~~~
RX1 (P11) -> Switch S1
TX1 (P10) -> Switch S2
P26 (P26) -> Switch on the lid
P24 (P24) -> LED
CEN       -> NC
ADC (P23) -> NC
P8 (P8)   -> NC
P7 (P7)   -> Relay 2
P6 (P6)   -> Relay 1
~~~

## Generating ESPHome Firmware

This can be done using the **ESPHome Dashboard**. One way to run it is via Docker:

~~~
docker run --rm --detach --name esphome \
  -v /home/gfisanot/esphome:/config \
  --net=host esphome/esphome
~~~

You can then access the dashboard using Google Chrome at [http://localhost:6052](http://localhost:6052).

1. Create a new **BK72XX** device and select **CB2S** as the board.
2. Replace the default template with the following YAML code:

```yaml
esphome:
  name: demasled-domo18a2
  friendly_name: demasled_domo18a2

bk72xx:
  board: cb2s

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "5MxV8N.............n25kru/F2ZwJR8bLaM="

ota:
  - platform: esphome
    password: "b8..................ca8a87"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  manual_ip:
    static_ip: 192.168.0.28
    gateway: 192.168.0.1
    subnet: 255.255.255.0
    dns1: 192.168.0.1

  # Enable fallback hotspot (captive portal) in case Wi-Fi connection fails
  ap:
    ssid: "Demasled-Domo18A2"
    password: "ez1XXXXXXef65"

captive_portal:

text_sensor:
  - platform: libretiny
    version:
      name: LibreTiny Version

binary_sensor:
  - platform: status
    name: "Node Status"
    id: system_status  
  - platform: gpio
    id: binary_onoff_1
    pin:
      number: P10
      inverted: true
      mode: INPUT_PULLUP
    on_state:
      then:
        - switch.toggle: switch_1
  - platform: gpio
    id: binary_onoff_2
    pin:
      number: P11
      inverted: true
      mode: INPUT_PULLUP
    on_state:
      then:
        - switch.toggle: switch_2
  - platform: gpio
    id: binary_switch_all
    pin:
      number: P26
      inverted: true
      mode: INPUT_PULLUP
    on_press:
      then:
        - switch.toggle: switch_1
        - switch.toggle: switch_2

switch:
  - platform: gpio
    id: switch_1
    name: Relay 1
    pin: P6
  - platform: gpio
    id: switch_2
    name: Relay 2
    pin: P7

status_led:
  pin:
    number: P24
    inverted: true
```

3. Choose **Install**, then **Manual Download**, and finally select **UF2 Package Format** to save the firmware to your local disk.

## Flashing the Generated Firmware

This step requires preparing the CB2S for flashing and installing the flashing tool.

### Hardware Preparation

You will need the following tools:

- USB-TTL adapter
- Soldering iron
- Some cables
- A multimeter

Refer to the **CB2S datasheet** (linked under **REFERENCES** below) for details.

Unfortunately, some components connected to the RX1 and/or TX1 pins prevent reading or writing the firmware. I was unable to flash the firmware without removing the CB2S module from the main board. With patience, flux, and desoldering wick, it is possible to desolder it.

Another issue I encountered was an error in the silkscreen on my CB2S. The pins were labeled **"3V3", "GND", "RX2", "TX1", "P24", "P26"** from right to left, but they should have been labeled in that order from left to right. This caused several hours of frustration until I noticed the discrepancy in the datasheet. Using a multimeter, I confirmed the error by tracing the output of the AMS117 regulator. I later found references to this issue online.

The first step involves soldering temporary wires from the CB2S to a USB-TTL adapter. Only five wires are needed: **RX1**, **TX1**, **CEN**, **GND**, and **3V3**. Connect **RX1** on the CB2S to **TX** on the TTL adapter, and **TX1** on the CB2S to **RX** on the TTL adapter. 

- **3V3** should connect to **3.3V** (the datasheet recommends using a separate power supply, as the TTL adapter may not provide sufficient power, but I had no issues with mine).
- **GND** on the CB2S should connect to **GND** on the TTL adapter.
- **CEN** should be temporarily connected to **GND** while the flashing tool attempts to access the module, putting the CB2S into flashing mode.

### Software Flashing Tool

Install the flashing tool:

~~~
pip install ltchiptool
~~~

To read (download) the firmware from the chip:

~~~
ltchiptool flash read BK7231N demasled_original_firmware.bin
~~~

Once you have the original firmware, you can use an **ltchiptool plugin** to analyze it and generate an ESPHome-compatible YAML file with all the connections and pins from the original firmware. This is a huge time-saver!

To install the plugin:

~~~
sudo pip3 install upk2esphome
~~~

To generate the ESPHome YAML:

~~~
ltchiptool plugin upk2esphome firmware demasled_original_firmware.bin
~~~

To write (upload/flash) the new firmware to the chip:

~~~
ltchiptool flash write demasled.uf2
~~~

Once the firmware is successfully flashed, confirm that the device connects to your Wi-Fi network. Power it off, desolder the five temporary cables, reinstall the CB2S in the device, and everything should work as expected.

### REFERENCES

- [CB2S Module Datasheet](https://developer.tuya.com/en/docs/iot/cb2s-module-datasheet?id=Kafgfsa2aaypq)
- [LibreTiny Documentation](https://docs.libretiny.eu/)
- [Helpful YouTube Tutorial](https://youtu.be/t0o8nMbqOSA)

