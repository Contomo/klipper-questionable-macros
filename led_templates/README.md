
---

# Klipper Advanced Toolchanger LED Control

An LED setup aimed for being as plug and play as possible. Automatically detecting states of the toolchanger, calibration, and regular Klipper things like homing.

### Overview

**Features**
-   **Automatic Status Detection:** LEDs react to printer actions without needing changes to most of your macros.
-   **Per-Tool Configuration:** Easily define `logo`, `nozzle`, and custom LED sections for each tool.
-   **Thermal Nozzle LEDs:** Nozzle lights automatically reflect the current hotend temperature.
-   **Highly Customizable:** Create your own templates and simply define them in the settings file.
-   **UI Integration:** Syncs tool colors with Mainsail's UI buttons.
-   **Error-Proofed:** Designed to be robust *(i hope)*. If a required configuration is missing, you will receive an informative error message at startup instead of a hard crash.

**Functionality/Logic**
-   Status detection works by estimation: homing is detected via out-of-bounds checks or "at homing speed" heuristics.
-   QGL detection works reliably by checking if the toolhead is moving along the defined probe path.
-   Parked tools are considered "busy" if heating, and 'ready' while idle otherwise (idle timeout -> 'idle').
-   parked tools nozzles are 'ready', (idle timeout -> 'idle'), or representing temperature if heated.

**Limitations**
-   The LED update frequency is internally *(klipper)* limited to ~2Hz.
-   There is no reliable way to auto-detect a bed mesh routine, so this remains a manual `STATUS_MESHING` call if desired.
-   This setup **requires** you to have a `[tool_probe_endstop]` section. It does not work with toolchangers ""tool detection"".
-   The "cleaning" status is only detected if you have a `CLEAN_NOZZLE` macro containing coordinates in its variables (e.g., `variable_middle_<x/y/z>`).

---

## Installation

1.  Add the files seperately or as a folder to your config.
2.  Add the include to your `printer.cfg` (`[include led_templates/*.cfg]`):

---

## Setup & Configuration

For each tool, you need to define the physical LEDs and indicies for this system to find them.
#### 1. Define your Physical LEDs

This system works with chainable LEDs (e.g., `[neopixel]` or `[dotstar]`). First, make sure you setup those leds. (**[How?](https://www.klipper3d.org/Config_Reference.html?h=neopixel#neopixel)**)

**Small Example:**
```salt
[neopixel T0_leds]
pin: toolboard_0:PD3
chain_count: 11
# ... and other required parameters for your specific LED type.
``` 

#### 2. Define your tools settings macro
> the naming scheme of those is based entirely on your `[tool <short_name>]` names!

if you dont have it already, we need to create a settings macro for the template loader to know which indicies of your `[neopixel]` to call for nozzle or logo. *(or a custom one if you add it, more on that later)*
- **Example**:
  - `[tool T0]` -> `[gcode_macro _T0_vars]`
  - `[tool White_Anthead]` -> `[gcode_macro _White_Anthead_vars]` *(case must match too)<sup> i think?</sup>* 
```salt
[gcode_macro _White_Anthead_vars]
variable_logo_led_name: "T0_leds"
variable_logo_idx: "1,2,3,4,5,6,7,8,11"
variable_logo_brightness: "1,1,1,1,1,1,1,1,0.4"
variable_nozzle_led_name: "T0_leds"
variable_nozzle_idx: "9,10"
variable_nozzle_brightness: "1,1"
gcode:
  # This gcode section is required, but can be empty.
```
Where here:
 - `led_name` *<sup>(ie name of leds)</sup>*
   - *(`[neopixel TOOL_LED1]` for example)*
   - may be different for nozzle/logo/custom sections
 - `idx`:
   - the physical index assigned to the sections led name (absolute, number in chain)
 - `brightness`:
   - the brigness "scalar" assigned to the indicies in that `idx` list
   - similarly to the color order definition in your neopixel section, may also be omitted, or just one (to scale all assigned idx's)


---

## Customization

### Editing Settings
`tc_led_status_settings.cfg` is all you need for **98%** of the settings youll likely want to fiddle with. Similarly to other LED setups, all color handling is done inside the `[gcode_macro _toolchanger_led_vars]` section.

-   **To change colors:** Edit the `variable_colors` dictionary. You can use the regular RGBW color dictionary or a template name, some of those are provided to get you started, but feel free to add your own/open a PR.
    ```salt
    variable_colors: { 
        'logo': { 
            # Static color for homing
            'homing'       :{'r':0.30, 'g':1.00, 'b':0.00, 'w':0.00},
            # Template for paused state
            'paused'       :'rainbow',
        },
        'nozzle': { ... },
      }
    ```

-   **Global settings:**
    -   `global_brightness: 1.00`: A master brightness multiplier for all LEDs.
    -   `invert: False`: Invert colors during probing moves.
    -   `force_updates: True`: Periodically adds a tiny offset to force LED updates, which can help correct pixel errors on noisy LED chains.
    -   `debug: ''`: Manually enter a tool number or use `DEBUG_LED T<number>` to get debug info in the console.

-   **Manual Control:** Similarly to the usual status system, some manual control remains.
    -   `STATUS_ON`: Turn nozzle LEDs on.
    -   `STATUS_OFF`: Turn all LEDs off.
    -   `STATUS_MESHING`: Set logo to 'meshing' color.
    -   `STATUS_IDLE`: Return to automatic control.

### Advanced Usage

-   **Custom Templates:** Create new patterns by defining a `[display_template <template_name>]` in `tc_led_status_settings.cfg`. 
If you arent familiar with led templates, it is basically like a gcode macro, except for the fact that the output *MUST* be a string of color values (`R,G,B,W`) from 0-1 (can be larger or smaller, are clamped, but the range to use is 0-1):
    
**simple example '`police`' template:**
```salt
[display_template simple_police]
text:
    {% if printer.toolhead.estimated_print_time|int % 2 == 0 %}
        1,0,0,0
    {% else %}
        0,0,1,0
    {% endif %}
```

-   **Automatic Parameters:** The system provides `param_idx`, `param_max_idx`, and `param_tn` to your template automatically.
    - > *do note that idx and max_idx are not absolte indicies like in your led macro settings, they are number in the chain/length of the chain of that tools logo leds*  
-   **Custom Parameters:** You can pass your own parameters to a template directly from the `variable_colors` definition.
    - > *do note that these are parsed as strings, on how to convert them back to your numbers, see `gradient-rgb` as an example*  
    - Example: `'printing': 'gradient-rgb param_from=1,0,0.5 param_to=0,0.5,1 param_speed=0.1'`<br>These become available as `param_from`, `param_to`, and `param_speed` inside the `gradient-rgb` template.

**The following can be easily found at the top of `tc_led_template_loading.cfg`**
-    **Additional Status Commands:** in case you need extra `[gcode_macro STATUS_<STATUS>]` calls because some arent covered by this setup, simply add the wanted status action to the `variable_colors:` dict of our settings macro.
     - call the overwrite with `_OVERWRITE_STATUS LOCATION=NOZZLE STATUS=<status> T=<active or number>`
     - or add the `STATUS_<STATUS>` macro to the config to call that instead.
-    **Additional Locations:** *(nozzle/logo)* you may add extra locations, define them the same way as the rest, and load your own templates onto them `LOAD_TOOL_LED_TEMPLATE NAME='<location>' TEMPLATE='<your_template>' T=<your tn>`

---

## led_effect

> A small note on custom templates. this is merely a system to switch to preconfigured "templates" and or static colors.  
> nothing is should prevent you from making a template which calls the common `led_effects` extra.

So for example:
(this kinda sucks because of how led_effect works but...)

```
[display_template rainbow]
param_tn: -1
text:
  {% if param_tn == 0 %}
     SET_LED_EFFECT EFFECT=rainbow_T0 REPLACE=1
  ....
```

This however is untested. if you do test it, let me know.

---

If you have any ideas/requsts/issues feel free to text me on discord, or open a `pull request` or `issue` here on github.
