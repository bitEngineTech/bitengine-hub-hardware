# Interfaces, indicators and specifications

This page describes what is on the outside of the bitEngine Hub and how the main blocks are
connected inside. For the overall structure see the [block diagram](block-diagram.svg).

The status-LED and button behaviour described here is implemented in the
[ESP32-C5 firmware](https://github.com/bitEngineTech/bitengine-hub-firmware) and reflects the
current firmware; if the two ever disagree, the firmware is authoritative.

Dimensions, mounting and the positions of connectors and light pipes on the enclosure will be
published with the enclosure files.

## External connectors

| Connector | Type | Function |
|---|---|---|
| Ethernet | RJ45, shielded | 10/100BASE-TX Ethernet (DM9051, SPI-attached to the ESP32-C5) and IEEE 802.3af PoE power input |
| USB-C | USB Type-C receptacle | 5 V DC power input (sink only), and USB 2.0 Full Speed to the ESP32-C5's built-in USB Serial/JTAG controller for flashing, serial console and debugging |
| ANT1 | RP-SMA female (reverse polarity) | Zigbee antenna — EFR32MG24, 2.4 GHz |
| ANT2 | RP-SMA female (reverse polarity) | Thread + Wi-Fi antenna — ESP32-C5 (802.15.4 at 2.4 GHz and Wi-Fi at 5 GHz), via the module's U.FL connector and a short internal coaxial pigtail |

Both antenna connectors are **RP-SMA**, not standard SMA: antennas and test cables need an
RP-SMA plug. A standard SMA antenna will not mate. The hub uses two detachable external
2.4 / 5 GHz dual-band antennas of the same type, one per radio, so Zigbee and Thread never share
an antenna or a radio.

The hub has one push button and four RGB status LEDs; they are described below.

## Power

| Item | Value |
|---|---|
| PoE input | IEEE 802.3af powered device; nominal 48 V DC (37–57 V at the powered device, per 802.3af). TMI7321 PD interface and flyback DC-DC converter to 5 V |
| USB-C input | 5 V DC, power sink only |
| Both inputs connected | Allowed. The two 5 V sources are combined by diode OR-ing; PoE takes priority |
| Internal rails | 5 V (also supplies the WS2812B LEDs); 3.3 V from a synchronous buck converter for all ICs |

## Radios

| Radio | Chip | Antenna | Band | Use |
|---|---|---|---|---|
| Zigbee 3.0 | EFR32MG24B020 (integrated PA, up to +19.5 dBm) | ANT1 | 2405–2480 MHz, IEEE 802.15.4 channels 11–26 | Zigbee coordinator (EmberZNet NCP), used by Zigbee2MQTT or ZHA through the hub |
| Thread | ESP32-C5 (IEEE 802.15.4 radio) | ANT2 | 2405–2480 MHz, IEEE 802.15.4 channels 11–26 | Thread Border Router running on the hub |
| Wi-Fi 6 | ESP32-C5 | ANT2 | 5 GHz only | Network uplink as an alternative to Ethernet (client mode). When the hub has no network connection it opens a temporary 5 GHz setup access point on channel 36 |
| Wi-Fi 2.4 GHz | ESP32-C5 | — | — | Present in hardware, **disabled in firmware** so the 2.4 GHz band stays clear for Zigbee and Thread |
| Bluetooth LE | ESP32-C5 and EFR32MG24 | — | — | Present in hardware, not enabled by the firmware |

- Zigbee (EFR32MG24) and Thread (ESP32-C5) are separate chips with separate antennas and can
  operate at the same time on different channels.
- 5 GHz Wi-Fi and Thread share the ESP32-C5's RF path and are time-shared by its coexistence
  mechanism.
- Zigbee and Thread transmit power limits follow the regulatory region selected in the hub's
  settings.

## Internal links

| Link | Details |
|---|---|
| ESP32-C5 ↔ EFR32MG24 | UART at 460 800 baud with RTS/CTS hardware flow control, carrying EZSP. The ESP32-C5 also controls the EFR32MG24's reset and boot-select lines, so it can update the Zigbee firmware over the same UART link |
| ESP32-C5 ↔ DM9051 | SPI |
| ESP32-C5 → status LEDs | One data line to a daisy chain of four WS2812B LEDs |
| Button → ESP32-C5 | One digital input |

There is no external flash: the ESP32-C5-WROOM-1U-N8R8 module provides 8 MB flash and 8 MB
PSRAM, and the EFR32MG24 uses its internal 1536 kB flash.

## Programming and debug access

| Chip | Access |
|---|---|
| ESP32-C5 | USB-C (built-in USB Serial/JTAG): firmware flashing, serial console, JTAG debugging. The board has no BOOT button |
| EFR32MG24 | Normally updated by the ESP32-C5 over the UART link. For direct programming, SWD test pads are provided inside the enclosure (not user-accessible) |

Pad locations and pinouts will be published with the schematics.

## Status LEDs

Four RGB LEDs, labelled **SYS**, **ZIGBEE**, **THREAD** and **NET**, in that order.

The three function LEDs (ZIGBEE, THREAD, NET) share one colour scheme:

| Indication | Meaning |
|---|---|
| Solid green | Working normally |
| Yellow, slow blink | Needs attention: in progress, degraded, or waiting for something |
| Red, slow blink | Fault |
| Off | Function not running |

The LEDs show the current **state**, not only activity, so a healthy idle hub shows green rather
than going dark.

"Slow blink" is about 1 Hz (0.5 s on, 0.5 s off); "fast flash" is about 3 Hz (0.15 s on,
0.15 s off).

### SYS — whole device

| Indication | Meaning |
|---|---|
| Solid green | Normal operation |
| Solid yellow | Starting up, or a firmware update has just finished and the hub is about to restart |
| Yellow, slow pulse (bright / dim) | Setup access point is active and waiting for configuration |
| Blue, slow pulse (bright / dim) | Connecting to the network, or reconnecting after losing it |
| Purple, fast flash | Firmware update in progress — do not remove power |
| Red, fast flash for a few seconds | An error event, after which the LED returns to its normal indication (see below) |

The red flash lasts different times depending on the cause:

| Duration | Cause |
|---|---|
| 3 s | Network connection lost (SYS then shows blue while reconnecting) |
| 3 s | Firmware update failed (the hub keeps running the previous firmware) |
| 2 s | The Zigbee chip was reset |
| 5 s | A setting could not be saved to flash |

### ZIGBEE

| Indication | Meaning |
|---|---|
| Solid green | Zigbee chip is working and a Zigbee host (Zigbee2MQTT or ZHA) is connected |
| Short blue blinks over green | Zigbee traffic through the hub |
| Yellow, slow blink | Zigbee chip is working, but no Zigbee host is connected |
| Red, slow blink | Zigbee chip is not responding |
| Off | Zigbee bridge disabled in the settings |

### THREAD

| Indication | Meaning |
|---|---|
| Solid green | The hub is attached to its Thread network and border routing is running |
| Yellow, slow blink | Not yet attached or dropped off the Thread network, or border routing is temporarily stopped (for example while the uplink switches between Ethernet and Wi-Fi) |
| Red, slow blink | The 802.15.4 radio has stopped sending and receiving; the firmware tries to recover automatically |
| Off | The Thread Border Router is not running |

This LED shows only the hub's own Thread state. A green THREAD LED does not mean every Thread
device is online; devices may be attached to other Thread routers.

### NET — network uplink

| Indication | Meaning |
|---|---|
| Solid green | Online (Ethernet or Wi-Fi) |
| Short white blinks over green | Network traffic (Zigbee bridge, web interface, MQTT, webhooks), at most one blink every 0.4 s |
| Yellow, slow blink | Connecting, reconnecting, or waiting for network setup |
| Red, slow blink | No network |

### Brightness and night mode

LED brightness and a night mode are set in the web interface. During the night-mode time
window all four LEDs are off; this is normal. The button feedback described below is still
shown during night mode and at full brightness.

## Button

One push button. The action depends on how long it is held:

| Hold time | Action |
|---|---|
| Less than 0.1 s | Ignored (accidental touch) |
| 0.1 – 2 s | Restart the hub |
| 2 – 10 s | No action (guard band against accidental factory reset). Also used as a physical confirmation when an action started in the web interface asks for one |
| 10 – 20 s | Factory reset |
| 20 s or more | Treated as a stuck button and ignored |

While the button is held, all four LEDs show what releasing it will do:

| LEDs while holding | Releasing now |
|---|---|
| Normal display (up to 2 s) | Restarts the hub (if held for at least 0.1 s) |
| All solid red (from 2 s) | Does nothing — the restart window has passed; keep holding for factory reset |
| All alternating red / white, fast (from 10 s) | Performs a factory reset |

To cancel after the LEDs have started alternating red and white, keep holding until 20 s have
passed; the press is then ignored.

A factory reset from the button clears the Wi-Fi settings, the password and the configuration,
and keeps the Thread network. The Zigbee network is stored in the EFR32MG24 and is not affected.

## Environmental

Design specification:

| Item | Value |
|---|---|
| Use | Indoor, fixed installation |
| Operating temperature | 0 to +45 °C |
| Storage temperature | −20 to +60 °C |
| Humidity | 10–90 % RH, non-condensing |

## Mechanical

- Production enclosure: aluminum extrusion. Prototype units use a 3D-printed enclosure.
- Dimensions, weight, mounting, and the positions of connectors and light pipes will be
  published with the enclosure files.
