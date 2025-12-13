# Using EnOcean devices in Home Assiastant

Motiviation collecting this information was that Mount Kelvin went out of market. It was a startup creating a proprietary light control system using mostly EnOcean devices.

This is not yet a completely drop-in replacement for Mount Kelvin. With this documentation and a lot of enthusiasm, you can get a complete system controlling your EnOcean lights. These instructions are not a collection of everything you can do with your devices, but just for pairing these two.

Please let the authors of this document to know what you found with your devices in the Issues-section of this repository, or with having a pull-rewquest. Let these examples to inspire you to help to make this documentation better:

- With FSUD dimmer, the instructions were to check the id sticker on the device. If you didn't have the sticker, you needed a way to obtain it. This is documented here.
- With Eltako USB 300, the instructions were also to check the id sticker. You might have one, but it might be completely misleading. The sticker id is not the id to use in Home Assistant. This document has a way to get the id you need in Home Assistant, whether you don't have the sticker, you have the wrong id in it, or even if you have a correct sticker with the correct id.

All these findings will help to make these instructions better.

This documentation has links into different parts of Home Assistant. If you want to use these links, make sure you are in the same network with your Home Assistant, and it is running in the address http://homeassistant.local:8123/. The Home Assistant version in the time of writing was **2025.12.2**.

## Hardware

The following hardware is tested and documented here.

### Control

- [Home Assistant Green](https://www.home-assistant.io/green/)
- [EnOcean USB 300](https://www.enocean.com/en/product/usb-300/)
  - This is located in the device path `/dev/ttyUSB0`. If you have a different RF module, you might need to change this serial port path in the commands in this document.

### Switches

- **Eltako FMH4** 4-way rocker switch
- **Eltako FT55** wall switch with 2-way or 4-way panel

### Dimmers

- **Eltako FRGBW71L** 4-channel dimmer
- **Eltako FSUD** wall socket dimmer (11/14 and later)
- **Eltako FUD61NPN** dimmer without wired connection to the wall switch

## Home Assistant

Prepare your Home Assistant for EnOcean

1. You need to install the following add-ons ([Settings > Add-ons](http://homeassistant.local:8123/hassio/dashboard) > Add-on store) to your Home Assistant:

    - [Advanced SSH & Web Terminal](http://homeassistant.local:8123/hassio/addon/a0d7b954_ssh/info) to access the command line
    - [File Editor](http://homeassistant.local:8123/hassio/addon/core_configurator/info) to edit `configuration.yaml`

2. Your EnOcean USB 300 dongle should be plugged in

3. Add an EnOcean device ([Settings > Devices & Services](http://homeassistant.local:8123/config/integrations/dashboard) > Add Integration > EnOcean)

## Add switches

1. In the EnOcean configuration, enable debug logging. ([Settings > Devices & Services > EnOcean](http://homeassistant.local:8123/config/integrations/integration/enocean) > upper right ⠇ > Enable debug logging)

2. Press your switch

3. Open the logs ([Settings > System > Logs](http://homeassistant.local:8123/config/logs)). Select Home Assistant Core and select Show Full Logs. You should see a line: `[homeassistant.components.enocean.dongle] Received radio packet: FE:FD:6E:1F->FF:FF:FF:FF (-70 dBm): 0x01`...

4. Please note the four bytes in the beginning. In my case FE:FD:6E:1F

5. Open `configuration.yaml` ([File Editor](http://homeassistant.local:8123/core_configurator/ingress) > Browse Filesystem > configuration.yaml), and add the following lines where you replace the id with yours:

    ```yaml
    binary_sensor:
      - platform: enocean
        id: [0xFE,0xFD,0x6E,0x1F]
        name: EnOcean switch 1
    ```

6. Restart Home Assistant ([Developer tools > YAML](http://homeassistant.local:8123/developer-tools/yaml) > Restart > Restart Home Assistant). If you are a little shy, you can also press the check configuration button.

7. You can check it is working, by listening the events and pressing the switch. The events can be listened in the Events tool ([Developer tools > Events](http://homeassistant.local:8123/developer-tools/event))
    - Listen to events
    - Events to subscribe to: `button_pressed`
    - Start listening

8. Repeat steps 2-5 with all of your buttons, and restart your Home Assistant. You don't need to repeat the line `binary_sensor:` when adding switches. Your file will look like this with your names and ids:

    ```yaml
    binary_sensor:
      - platform: enocean
        name: EnOcean switch 1
        id: [0xFE,0xFD,0x6E,0x1F]
      - platform: enocean
        name: EnOcean switch 2
        id: [0xFE,0xFD,0x6E,0x2F]
    ```

## Prepare Home Assistant for teaching

1. Go to [Terminal](http://homeassistant.local:8123/a0d7b954_ssh).
2. Run the following command to install necessary Python plugins

    ```zsh
    pip3 install enocean
    ```

3. Obtain the EnOcean **sender id**. This is the base id for all the devices you will teach later. Each device will receive individual sender id.

    ```zsh
    python3 -c "from enocean.communicators.serialcommunicator import SerialCommunicator;
    c = SerialCommunicator(port='/dev/ttyUSB0'); c.start();
    hex_list_str = '[' + ', '.join(f'{b:#04x}' for b in c.base_id) + ']'; print('Base ID:', hex_list_str);
    c.stop()"
    ```

4. Notice the output `Base ID: [0xff, 0xd3, 0x6e, 0x80]`. You have **128 sender ids** you can assign starting from this id. In this document, base `[0xff, 0xd3, 0x6e, 0x80]` will be used in the examples.

## Add dimmers

The dimmers need to be paired with Home Assistant. Every dimmer has two ids:

- `sender_id` is teached to the device to receive commands from Home Assistant
- `id` is used for receiving status from the device to Home Assistant, not all devices need this

1. **Clear** the dimmer memory contents completely to prevent it registering unwanted control signals.
   - [FRGBW71L](#clearing-frgbw71l)
   - [FSUD](#clearing-fsud)
   - [FUD61NPN](#clearing-fud61npn)
2. Obtain the **device id** to be configured as `id` later.
   - [FSUD](#get-fsud-id)
   - [FUD61NPN](#get-fud61npn-id)
3. Take a **new sender id** by changing the last hexadecimal number in your **base sender id**. Write this id down. In this example it is `[0xff, 0xd3, 0x6f, 0x81]`.
4. Put the device in **teaching mode**.
   - [FRGBW71L](#teaching-frgbw71l)
   - [FSUD](#teaching-fsud)
   - [FUD61NPN](#teaching-fud61npn)
5. In Home Assistant [Terminal](http://homeassistant.local:8123/a0d7b954_ssh), run the following command replacing first `my_id = [0xff, 0xd3, 0x6f, 0x81]` with the id you want to be teached:

    ```bash
    python3 -c "import time;
    from enocean.communicators.serialcommunicator import SerialCommunicator;
    from enocean.protocol.packet import RadioPacket;
    from enocean.protocol.constants import PACKET, RORG;
    c = SerialCommunicator(port='/dev/ttyUSB0');
    c.start();
    my_id = [0xff, 0xd3, 0x6f, 0x81];
    print(f'TEACHING WITH ID: {my_id}');
    p = RadioPacket(PACKET.RADIO_ERP1, data=[0]*10, optional=[0]*7);
    p.optional = [];
    print('1. Sending Eltako GFVS Signature...');
    p.data = [RORG.BS4, 0xE0, 0x40, 0x0D, 0x80] + my_id + [0x00];
    c.send(p); time.sleep(1); c.stop(); print('Done.')"
    ```

6. The LED goes out.
7. Configure the device id (`id`) and teached sender id (`sender_id`) into your `configuration.yaml` ([File Editor](http://homeassistant.local:8123/core_configurator/ingress) > Browse Filesystem > configuration.yaml). If you couldn't obtain the device id, you can make it up. Just be sure each device has different id:

    ```yaml
    light:
      - platform: enocean
        name: FUD61 hallway
        id: [0x01, 0x8D, 0x6F, 0x16]
        sender_id: [0xFF,0xD3,0x6E,0x81]
    ```

8. **Restart** Home Assistant ([Developer tools > YAML](http://homeassistant.local:8123/developer-tools/yaml) > Restart > Restart Home Assistant). Check the config if you don't feel confident.

### Instructions per device

#### FRGBW71L

##### Clearing FRGBW71L

1. Set the **middle** rotary switch to **CLR**. The LED flashes at a high rate.
2. Within the next 10 seconds, turn the **upper** rotary switch **three times** to the **right stop** (turn clockwise) and then turn back away
from the stop. The LED stops flashing and goes out after 2 seconds.
3. All taught-in sensors are cleared.

##### Teaching FRGBW71L

1. Set the **top** rotary switch to **10**
2. Turn the **lower** rotary switch to the required channel **1** to **4**. Each RGBW channel needs to be teached separately to separate sender id
3. Set the **middle** rotary switch to **LRN**. The LED flashes at a low rate.
4. Teach
5. After teaching, turn the **middle** rotary switch away from **LRN**

#### FSUD

##### Clearing FSUD

1. Press the left button **LRN/CLR** for approximately **3 seconds**, the LED flashes exitedly.
2. Press the right button **ON/OFF** approximately **5 seconds**, the LED goes out.
3. All taught-in sensors are cleared, the repeater and the confirmation telegrams are switched off.

##### Get FSUD id

1. In the EnOcean configuration, enable debug logging. ([Settings > Devices & Services > EnOcean](http://homeassistant.local:8123/config/integrations/integration/enocean) > upper right ⠇ > Enable debug logging)

2. Press a button in your dimmer

3. Open the logs ([Settings > System > Logs](http://homeassistant.local:8123/config/logs)). Select Home Assistant Core and select Show Full Logs. You should see a line: `[homeassistant.components.enocean.dongle] Received radio packet: FE:FD:6E:2F->FF:FF:FF:FF (-70 dBm): 0x01`...

4. Please note the four bytes in the beginning. In my case FE:FD:6E:2F

##### Teaching FSUD

1. Press and hold the left button **LRN/CLR** for approx. **0.5 seconds** and then release. The LED lights up.
2. Press the right button **ON/OFF** briefly once. The LED flashes once as confirmation.

#### FUD61NPN

##### Clearing FUD61NPN

1. Set the **upper** rotary switch to **CLR**. The LED flashes at a high rate.

2. Within the next 10 seconds, turn the **lower** rotary switch **three times** to the **right stop** (turn clockwise) and then turn back away from the stop. The LED stops flashing and goes out after 2 seconds.

3. All taught-in sensors are cleared.

##### Get FUD61NPN id

1. Look the sticker at the bottom of your FUD61NPN
2. The id for Home Assistant is 8 digit hexadecimal number. If you have 7 number sticker, you need to add 0 in the front of it. The id should be converted to 4 hexadecimal numbers. For example sticker 18D6F16 becomes `[0x01, 0x8D, 0x6F, 0x16]`.
3. If you don't have it, you can make this up, but remember to make it unique. It also appeared that the last number needs to be even if you make this up.

##### Teaching FUD61NPN

1. It doesn't matter where the **lower** rotary switch is.
2. Set the **upper** rotary switch to **LRN**. The LED flashes at a low rate.
3. Teach
4. After teaching, set the **lower** rotary switch to **Auto**, and **upper** rotary switch in the **middle** of the travel

## Pairing your switches to the dimmers in Home Assistant

1. Go to Settings > [Automations & scenes](http://homeassistant.local:8123/config/automation/dashboard)
2. Take one of your **button id**s you have stored, and replace it later to the place of `[0xFE,0xFD,0x6E,0x1F]`
3. Create automation for **turning on** the lights. Select Create automation > Create new automation
    - When > Add trigger > Manual event > Manual event
      - Event type: `button_pressed`
      - Event data:

        ```yaml
          id: [0xFE,0xFD,0x6E,0x1F]
          pushed: 1
          which: 1
          onoff: 0
        ```

        where
          - `id` is your button id
          - `pushed` is always `1`
          - `which` is `0` for left in 4-way rocker, and `1` for right in 4-way rocker. If you have only one rocker in your wall switch, it's the right one (`1`)
          - `onoff` is `0` for up, and `1` for down

        If these instructions were not clear, Event viewer will show you these values.
    - Then > Add action > Light > Turn on > Add target, and select your lights you want to turn on.
4. Create another automation for **turning off** the lights by repeating the step above. Change `onoff` to `1` for pressing the rocker down.
5. Repeat all the steps for each switch-light combination, and you are done.

## Finalization

Back up your settings, and you can disable your SSH and file Add-ons to make security attacks a little less likely.

## Tips

### Controlling a light from the command line

You can get faster feedback that your teaching was successful by controlling your light from the command line. The following command sets the brightness to 40% for the light paired with the sender id `[0xff, 0xd3, 0x6f, 0x81]`. A LED in the dimmer should flash once when running this command.

```zsh
python3 -c "import time;
from enocean.communicators.serialcommunicator import SerialCommunicator;
from enocean.protocol.packet import RadioPacket;
from enocean.protocol.constants import PACKET, RORG;
c = SerialCommunicator(port='/dev/ttyUSB0'); c.start();
my_id = [0xff, 0xd3, 0x6f, 0x81];
p = RadioPacket(PACKET.RADIO_ERP1, data=[0]*10, optional=[0]*7);
p.data = [RORG.BS4, 0x02, 40, 0x00, 0x09] + my_id + [0x00];
p.optional = []; c.send(p);
print('Dimmed to 40%');
time.sleep(1); c.stop()"
```

### Conflict using the serial port

When operating the USB dongle both from the command line and from the Home Assistant EnOcean module, one of them might become inoperable. You may see the following line on [Settings > System > Logs](http://homeassistant.local:8123/config/logs):

> Serial port exception! (device disconnected or multiple access on port?)

You can fix this by disabling and enabling EnOcean module, or just rebooting your Home Assistant.
