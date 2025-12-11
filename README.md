# Documentation about using EnOcean devices in Home Assiastant

Motiviation collecting this information was that Mount Kelvin went out of market. It was a startup creating a proprietary light control system using mostly EnOcean devices.

This is not yet a completely drop-in replacement for Mount Kelvin. With this documentation and a lot of enthusiasm, you can get a complete system controlling your EnOcean lights.

This documentation uses links to the local Home Assistant server running in the **port 8123**. These instructions are written using **Home Assistant Core 2025.12.2**.

## Hardware

The following hardware is tested and documented here.

### Control

- [Home Assistant Green](https://www.home-assistant.io/green/)
- [EnOcean USB 300](https://www.enocean.com/en/product/usb-300/)

### Switches

- **Eltako FT55** wall switch with 2-way or 4-way panel
- **Eltako FMH4** 4-way rocker switch

### Dimmers

- **Eltako FUD61NPN** dimmer
- **Eltako FRGBW71L** 4-channel dimmer
- **Eltako FSUD-230V** wall socket dimmer

## Home Assistant

1. You need to install the following add-ons ([Settings > Add-ons](http://homeassistant.local:8123/hassio/dashboard) > Add-on store) to your Home Assistant:

    - [Advanced SSH & Web Terminal](http://homeassistant.local:8123/hassio/addon/a0d7b954_ssh/info) to access the command line
    - [File Editor](http://homeassistant.local:8123/hassio/addon/core_configurator/info) to edit `configuration.yaml`

2. Plug the dongle in

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
        name: EnOcean switch
    ```

6. Restart Home Assistant ([Developer tools > YAML](http://homeassistant.local:8123/developer-tools/yaml) > Restart > Restart Home Assistant). If you are a little shy, you can also press the check configuration button.

7. You can check it went all ok by checking the events and pressing the switch. The events can be listened in the Events tool ([Developer tools > Events](http://homeassistant.local:8123/developer-tools/event))
    - Listen to events
    - Events to subscribe to: `button_pressed`
    - Start listening

8. Repeat with all of your buttons
