# Automatic Garage Door
[//]: # (TODO add gif opening and closing door)
[//]: # (TODO add foto of build)
[//]: # (TODO add screenshot of schematic)

The house we bought has an automatic garage door. Super dope. The bad news is that the remote broke, and a replacement is hard to come by and too expense. So why not see this as a chance to create the controller myself?

## Garage door motor remote
The garage door motor that the previous owner installed is a Chamberlain ML500. This is a bit of an older model so it should be easy to hack into. After some digging I found out that the current remote uses a 433mhz signal to communicate. A simple [433mhz sender receiver](https://www.amazon.nl/AZDelivery-433-MHz-Radio-Ontvanger/dp/B01N5GV39I/ref=asc_df_B01N5GV39I/?tag=nlshogostdde-21&linkCode=df0&hvadid=430533494283&hvpos=&hvnetw=g&hvrand=1912627896593820558&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=1010304&hvtargid=pla-655433778930&th=1) should be suited to replicate the remote signal. Easy peasy, **right??**

Wrong. Older garage door motors used to send plain telegrams from the remotes, but this model is not that old. A more secure way of communicating is via [_rolling codes_](https://crypto.stackexchange.com/questions/18311/how-does-a-rolling-code-work). In short, this means that the sender and receiver have an internal sequence generator. When the button is pressed, the remote sends the next key in the sequence. When this key matches (or is close to) the next key that the receiver generates, the door opens. 

This did mean that I would not be able to send a message via 433mhz. After a lot more digging, I found out that these garage door motors usually support a simple button that can be installed inside the garage. This enables the owner to simple open the door from the inside. Now this I could replicate! The manual showed this infographic to install such a button: 

[//]: # (TODO add image)

From what a could gather this button would simply connect the two pins on the motor. A simple relay could do this, so the plan is: Connect two wires from the motor pins to a relay, close the relay to connect the wires, ???, profit. 

## Hardware
So the planned setup was this:
 - **NodeMCU** - Cheap ESP8266 development board I had laying around. This would receive the signal via WiFi.
 - **5V relay** - To connect the two pins
 - **5V power supply** - For the NodeMCU. 

Connecting this all up is really simple. The relay needs a power supply, a digital input and a ground. Connect the power supply pin of the relay to 3v3 in the NodeMCU. Although this is a 5v relay, I've found that 3v3 works just fine ¯\_(ツ)_/¯. Connect ground to ground and input to any digital pin on the NodeMCU.

## Controlling the ESP
I use [Home Assistant](https://www.home-assistant.io/) for my other domotics projects. Home Assistant is fantastic. Every time I think of something new to automate in the house, Home Assistant already has some nifty integration ready! This project was no exception. 

I searched for a simple library to program the ESP and found that in [ESPHome](https://esphome.io/). This project enables you to configure the ESP with just a simple YAML file. Automatic Home Assistant integration is all taken care of with [this](https://www.home-assistant.io/integrations/esphome/) integration. No need to think at all, great!

The ESPHome yaml file for this project looks like this:
```yaml
esphome:
  name: garagedoor
  platform: ESP8266
  board: nodemcuv2

wifi:
  ssid: "WIFI_SSID"
  password: "WIFI_PASSWORD"
  # I use a static ip as '.local' addresses don't really work in my network (a story for another blog post...)
  manual_ip:
    static_ip: 192.168.1.106
    gateway: 192.168.1.100
    subnet: 255.255.255.0

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Garage door Fallback Hotspot"
    password: "NotTheActualPassword"

# Enable logging
logger:

# Enable Home Assistant API
api:

# Enables Over the air updates
ota:

switch:
  - platform: gpio
    pin: D1
    id: gara_bt
  - platform: template
    name: Garage Door
    optimistic: True
    turn_on_action:
      - switch.turn_on: gara_bt
      - delay: 200ms
      - switch.turn_off: gara_bt
    turn_off_action:
      - switch.turn_on: gara_bt
      - delay: 200ms
      - switch.turn_off: gara_bt
```

### Switch
As you can seen in the example I use a template switch. This switch enables pin D1 for 200ms and then disables it. This template is used for both enable and disable. The garage door expects a button to be pressed so only setting the pin to high on enable and low on disable would not work.

## Results
She works great! After uploading the config to the NodeMCU and starting it, Home Assistant immediately notified me that it had found a new device. Add device, click switch, door opens. Amazing!

Though there is room for improvement. The WiFi connection of the NodeMCU was dodgy to say the least. It could be offline for a day, than be back up for a couple hours, etc. Not great.

Besides, I was really missing a way to see the status of the door. Since the template switch switches to relay back and forwards, the door opens when you enable and disable it. This works just fine, but when you're inside, you have no idea if the button worked.

## Improvements
After these results, I changed 3 things:
 1. Swapped the **NodeMCU** for a **Wemos D1 Mini**.  
This has the same chip but also features an antenna. Should improve the connection.
 2. Added a **Reed switch** and a magnet to the door.  
This way I can tell when the door is closed.
 3. Changed from the ESPHome API to **MQTT**.  
I have some other devices connected to Home Assistant via MQTT and this makes the system more decoupled. Home Assistant is no longer directly responsible for the connection to other devices which is nice. ESPHome also has [fantastic](https://esphome.io/components/mqtt.html) MQTT support although they don't advice it. 

The new ESPHome config looks like this:
```yaml
esphome:
  name: garagedoor
  platform: ESP8266
  board: d1_mini_pro

wifi:
  ssid: "SSID"
  password: "PASSWORD"

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Garage door mqtt Fallback Hotspot"
    password: "NotTheActualPassword"

# Enable logging
logger:

ota:

mqtt:
  broker: BROKER_IP

switch:
  - platform: gpio
    pin: D1
    id: gara_bt
  - platform: template
    name: Garage Door
    turn_on_action:
      - switch.turn_on: gara_bt
      - delay: 200ms
      - switch.turn_off: gara_bt
    turn_off_action:
      - switch.turn_on: gara_bt
      - delay: 200ms
      - switch.turn_off: gara_bt

# A sensor to monitor the wifi signal strength
sensor:
  - platform: wifi_signal
    name: "NodeMCU WiFi Signal Sensor"
    update_interval: 15s

# The binary sensor for the reed switch
binary_sensor:
  - platform: gpio
    name: "Garagedeur open"
    device_class: garage_door
    pin:
      number: D2
      mode: INPUT_PULLUP
```

Results for the WiFi signal are great! Every once in a while the device if offline, but not more than a couple minutes at a time.

This chip does have a 5v pin, and I did need to use this one for the relay as the 3.3v pin didn't work.

# Conclusion
Things that are great:
 1. **Wemos D1 Mini**
Great range improvement, great price
 2. **ESPHome**
A joy to work with
 3. **Home Assistant**
All powerful catch all solution for everything
 4. **An automatic garage door**
More lazy, more better.
