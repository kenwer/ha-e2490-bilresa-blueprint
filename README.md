# HA Blueprint for the IKEA E2490 BILRESA Scroll Wheel
A Home Assistant automation blueprint for the **[IKEA E2490 BILRESA](https://www.ikea.com/us/en/p/bilresa-remote-control-white-smart-scroll-wheel-70617457)** scroll wheel controller,
supporting both **[Zigbee2MQTT](https://www.zigbee2mqtt.io/)** and **[ZHA](https://www.home-assistant.io/integrations/zha/)** integrations.

---
## Features
- **4 scroll modes:** brightness, volume, color temperature, hue
- **Mode cycling:** double-tap ON to cycle between configured modes at runtime
- **Auto-reset:** returns to the default mode after a configurable idle period
- **Custom button actions:** override any button with your own action sequences
- **Light state guard:** prevents scroll events from turning lights back on after an OFF press
- **Min brightness clamp:** scroll-to-bottom never physically cuts power to the bulb

### Integration support
| Feature | Zigbee2MQTT | ZHA |
|---|---|---|
| ON / OFF buttons | Yes | Yes |
| Double-press buttons | Yes | No |
| Brightness scroll | Yes | Yes |
| Volume scroll | Yes | No |
| Color temp scroll | Yes | No |
| Hue scroll | Yes | No |

ZHA does not expose double-press events for this device, so there is no easy way to cycle scroll
modes at runtime. Because of this, the additional scroll modes (volume, color temp, hue) are
not implemented for ZHA.

---
## Installation

[![Import blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fkenwer%2Fha-e2490-bilresa-blueprint%2Frefs%2Fheads%2Fmain%2Fikea-e2490-bilresa-scroll-wheel.yaml)

Or manually: **Settings -> Automations & Scenes -> Blueprints -> Import Blueprint**, then paste the raw URL of `ikea-e2490-bilresa-scroll-wheel.yaml` from this repository.

---
## Setup
Go to **Settings -> Automations & Scenes -> Blueprints**, find
**IKEA E2490 BILRESA Scroll Wheel**, and click **Create Automation**.

### Required inputs
| Input | Description |
|---|---|
| Controller Device | Select your E2490 from the device list (Z2M or ZHA) |
| Scroll wheel target (light) | The primary light — used for brightness mode and as the default ON/OFF target |

### Zigbee2MQTT: set your device topic
The blueprint needs to read the raw MQTT payload to get the scroll position. Set the
**Zigbee2MQTT device topic** field to the exact topic for your device, e.g.
`zigbee2mqtt/Kitchen-RemoteDial-Beige`. You can find this in the Z2M device list.

> **Why is this needed?** Home Assistant blueprint triggers must be static strings, so the
> topic cannot be derived automatically from the device selector. The default `zigbee2mqtt/#`
> wildcard works but subscribes to all Z2M traffic; setting a specific topic is more efficient.

ZHA users: leave the topic field at its default.

### Optional: enable additional scroll modes
Set any of these to unlock the corresponding scroll mode:
| Input | Enables |
|---|---|
| Volume target (media_player) | Volume mode — ON/OFF buttons also control the media player |
| Color temperature target (light) | Color temp mode — scrolls between the bulb's warmest and coolest white |
| Hue target (light) | Hue mode — scrolls through 0-360 degrees of the color wheel |

Modes without a configured target are silently skipped during cycling.

---
## Scroll mode cycling

If you configure more than one scroll mode, you can switch between them at runtime by
double-pressing ON. This requires a **Scroll Mode Helper** (`input_select`).

### Create the helpers
Create these via **Settings -> Helpers -> + Create Helper**, or add them to your
`configuration.yaml`:

```yaml
input_select:
  kitchen_bilresa_scroll_mode:
    name: "Kitchen-Bilresa-ScrollModeHelper"
    options: [brightness, volume, color_temp, hue]
    initial: brightness
    icon: mdi:gesture-swipe

input_datetime:
  kitchen_bilresa_last_activity:
    name: "Kitchen-Bilresa-LastActivityHelper"
    has_date: true
    has_time: true
```

The options list can be in any order and can include only the modes you use.
The `input_datetime` helper is only needed if you want auto-reset.

> If you have multiple BILRESA devices, create a separate pair of helpers per device
> so each device tracks its scroll mode independently.

### Auto-reset
Set **Last Activity Helper** and a non-zero **Auto-reset timeout** (default: 30 seconds).
After the configured idle period the scroll mode automatically reverts to the default
mode you selected in the automation.

---
## Button behavior
### Default behavior (no custom actions configured)

| Button | Default action |
|---|---|
| ON (short) | Turn on the light (or media player in volume mode) |
| OFF (short) | Turn off the light (or media player in volume mode) |
| ON (double) | Cycle to next scroll mode (if helper configured), or set brightness to maximum |
| OFF (double) | Turn off the brightness target light |

### Custom actions
All four buttons accept custom action sequences. When a custom action is configured it
takes full priority — the default behavior is not executed.

---
## Tuning
| Input | Default | Notes |
|---|---|---|
| Min brightness | 2 | Scroll-to-bottom clamps here instead of 0 (which would cut power on most bulbs) |
| Max volume | 1.0 | Useful if you never want the scroll wheel to go to full volume |

---
## Known device quirks (BILRESA / E2490)
- `action_level` range is **1-241**, not 0-255. The blueprint scales this to the 1-255
  brightness range and 0-360 hue range automatically.
- The device sends `action_level: null` at the start of a scroll session and alongside
  every button press. These null events are discarded at the trigger level to prevent a
  snap-to-255 brightness side effect.
- On every button press the device fires `brightness_move_to_level` (with an `action_level`)
  **before** the `on`/`off` action. A light-state guard prevents this from turning the
  light back on immediately after an OFF press.
- ZHA double-press events are not exposed, so ON double / OFF double only work with Z2M.

---
## Acknowledgements
- Thanks to [Andy McInnes](https://github.com/mcinnes01) for the [original blueprint](https://community.home-assistant.io/t/ikea-e2490-bilresa-scroll-wheel/976506) this work is based on.
