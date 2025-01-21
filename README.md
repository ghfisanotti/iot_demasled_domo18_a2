# Flashing ESPhome to a Beken/Tuya CB2S powered smart switch

This is a WiFi Switch powered by a Tuya/Beken CB2S module with a BK7231N MCU sold in my country under the brand Demasled, model DOMO-18-A2

It's used with the Tuya app installed in Android.

Before any hardware hacking, it is convenient to configure this device using the Tuya app and verify that wverything is in order.

## Hacking it to be used locally with HomeAssistant

Although it is possible to somewhat integrate Tuya devices in the cloud with HomeAssistant, 
I prefer a local integration, in order to do that, I changed the firmware of the CB2S for
ESPhome, which also required to reverse engineer the connections of the relay, the button
and the LED.

After some fiddling with the multi-tester and some trying, I determined these are the
connections of the CB2S module:

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

## Generating ESPhome firmware

This can be done with the ESPhome Dashboard, one way is running this in Docker:

~~~
docker run --rm --detach --name esphome \
  -v /home/gfisanot/esphome:/config \
  --net=host esphome/esphome
~~~

Then you can access the dashboard using Google Chrome at http://localhost:6052

Create a new BK72XX device and select CB2S as the board. Then replace the default
template with the following YAML code:

~~~
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
    key: "5MxV8NIFaR4Z+lf5gbfhtGIsTn25kru/F2ZwJR8bLaM="

ota:
  - platform: esphome
    password: "b842264a646cf08499788a050bca8a87"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  manual_ip:
    static_ip: 192.168.0.28
    gateway: 192.168.0.1
    subnet: 255.255.255.0
    dns1: 192.168.0.1

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Demasled-Domo18A2"
    password: "ez14E6tkef65"

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
~~~

Then choose Install, Manual download, finally select UF2 Package format and save to local disk.

## Flashing the generated firmware

This step requires preparing the CB2S to be flashed and installing the flashing tool.

### Hardware preparation

A few basic hardware tools are required:

- USB-TTL adapter
- Soldering iron
- some cables
- a multi-tester

Under REFERENCES below there is a link to the CB2S datasheet.

Unfortunately, in this case it is necessary to desolder the CB2S module from the device because some of the components connected to the RX1 and/or TX1 pins make it impossible to read or write the firmware. With patience, some flux and desoldering wick it is possible to desolder it.

Another problem that I encountered was that the silkscreen on my CB2S was wrong, the pins where labeled "3V3","GND","RX2","TX1","P24","P26" from right to left, they should be labelled in that order but from left to right. This cost me a few hours of head banging till I noticed in the datasheet that the order was inverted, then I followed the output of the AMS117 regulator with the multitester and to my surprise I corroborated the error in the silkscreen. Later I found a couple of references to this problem on the Internet.
 
The first step requires soldering temporary wires from the WB2S to a USB-TTL adapter. Only five wires are
needed: pins RX1, TX1, CEN, GND and 3V3. RX1 from the CB2S whould go to Tx on the TTL adapter, 
likewise, TX1 should go to Rx on the other end. 

3V3 should go to 3.3V (Datasheet suggests to use a separate power supply as the TTL 
adapter may not suffice, but I had no problem whith mine). 

Obviously, GND from WB2S should go to GND on the TTL adapter.

CEN should to GND temporarily while the flashing tool is trying to access the module in order to put the CB2S in 
flashing mode.

### Software flashing tool

~~~
pip install ltchiptool
~~~

To read (download) firmware from the chip:

~~~
ltchiptool flash read BK7231N demasled_original_firmware.bin
~~~

Once we have the original firmware, we can use an ltchiptool plugin to analize it and generate a ESPhome-compatible YAML with all connections and pins from the original firmware, a real life saver!

To install the plugin:

~~~
sudo pip3 install upk2esphome
~~~

To generate the ESPhome YAML:

~~~
ltchiptool plugin upk2esphome firmware demasled_original_firmware.bin
~~~

To write (upload, flash) new firmware to the chip:

~~~
ltchiptool flash write demasled.uf2
~~~

Once sucessfully flashed, confirm that the device connects to the WiFi network, turn off, desolder the five temporary cables, install the CB2S again in the device and everything should be working.

### REFERENCES

https://developer.tuya.com/en/docs/iot/cb2s-module-datasheet?id=Kafgfsa2aaypq

https://docs.libretiny.eu/

https://youtu.be/t0o8nMbqOSA
