
# Macros Macros

- Setup:
  - [make your own](#creation)
  - [using them](#usage)
  - [loading them](#loading)
- [Examples](#examples):
  - [toolchanger](#toolchanger-helpers)


---

## Creation
A note is to be taken with jinjas macros. if you arent too familiar with it. they output the entire content, that would also include **newlines and spaces**.
to get around that we can control whitespace using `{%-` and/or `-%}`

macros also always return strings. 
To avoid running into issues, we can build our macros in such a way, that we always hand in a mutabl to "edit in place". 
(something like a namespace, list or dict)
```nunjucks
[gcode_macro _math_helper_macros]
gcode:
  {%- macro sqrt(x) -%}
    ...
    {root}
  {%- endmacro -%}
  ...
```


## Usage
- just all
  ```nunjucks
  [gcode_macro MATH_TEST]
  gcode:
      {% import math as math %}
      RESPOND MSG={math.sin(math.pi)}
  ```
- only one
  ```nunjucks
  [gcode_macro MATH_TEST]
  gcode:
      {% from math import sin, pi %}
      RESPOND MSG={sin(pi)}
  ```
- import and also add your `printer`, `action_<respond/raise>_<info/error...>` to that template
  ```nunjucks
  [gcode_macro MATH_TEST]
  gcode:
      {% import math as math with context %}
      RESPOND MSG={math.something_that_uses_action_respond("hello")}
  ```
*(it do be as simple as that, yes)*

## Loading
Klipper doesnt nativly offer a file loading, that means, when you try to use `{% import ... as ... %}` you usually get an error.
The reason for the error is simply, it couldnt find it in its globals, and thus, wants to "load from file". Since klipper didnt pack that, we error out.

The simple solution: pack our already evaluated/loaded templates into that global at startup. simple as that.


For that we can use this macro, feel free to change `prefix` and `suffix` to suit your naming scheme.
```nunjucks
[delayed_gcode _REGISTER_IMPORTS_AT_START]
initial_duration: 0.5
gcode:
    {% set prefix = "_"              %}
    {% set suffix = "_helper_macros" %}

    {% set registered = [] %}
    {% set env = printer.printer.lookup_object('gcode_macro').env %}
    #---< register every matching runtime macro >------------------------------
    {% for name, obj in printer.printer.lookup_objects() if 'template' in obj.__dir__() %}
        {% if name.startswith("gcode_macro " ~ prefix) and name.endswith(suffix) %}
            {% set start = ("gcode_macro " ~ prefix)|length %}
            {% set end   = name|length - suffix|length %}
            {% set core  = name[start:end] %}
            {% set _     = env.globals.update({core: obj.template.template}) %}
            {% set _     = registered.append(" ─ '" ~ name ~ "' as '" ~ core ~ "'") %}
        {% endif %}
    {% endfor %}
    #---< summary block >---------------------------------------------------------
    {% if registered %}
        {% set html = "<details>" ~ "<summary>" ~ registered|length ~ " helper template(s) registered</summary>" ~ registered|map('e')|join("<br>") ~ "</details>" %}
        {action_respond_info(html)}
    {% else %}
        {action_respond_info("No templates found to load for import.")}
    {% endif %}

```


---

**Optionally if you dont want to do that**
```nunjucks
[gcode_macro DO_SOMETHING_WITH_CFG_HELPERS]
gcode:
    {% set template = printer.configfile.settings['gcode_macro _save_config_helper'].gcode %}
    {% set gcode_macro = printer.printer.lookup_object('gcode_macro') %}
    {% set cfg_helper_no_context   = gcode_macro.env.from_string(template).module %}
    {% set cfg_helper_full_context = gcode_macro.env.from_string(_lib_cfg, globals=self._TemplateReference__context).module %}

    {cfg_helper_full_context.save_config_unstage('stepper_y', 'rotation_distance')}
    {cfg_helper_full_context.save_config_stage  ('stepper_x', 'rotation_distance', 40.999)}
```



---

# Examples

---

## toolchanger helpers
> most macros (that you dont expect to return something) may be best piped straight to the output.

### toolchanger shorts

| Macro                        | Description                                     | Example                           |
| ---------------------------- | ----------------------------------------------- | --------------------------------- |
| `get_probe_name_from_tn(tn)` | Lookup probe section name for tool `tn`         | `{get_probe_name_from_tn(1)}` |
| `get_tool_target(tn)`        | Get target temperature of tool `tn`             | `{get_tool_target(0)}`        |
| `get_tool_temp(tn)`          | Get current temperature reading for tool `tn`   | `{get_tool_temp(0)}`          |
| `tool_can_extrude(tn)`       | Returns `'True'` if tool’s extruder can extrude | `{tool_can_extrude(1)}`       |
| `get_mounted_tn()`           | force query the current mounted tn (~250ms)     | `{get_mounted_tn()}`          |

### Variable helpers

| Macro                                            | Description                                             | Example                                                          |
| ------------------------------------------------ | ------------------------------------------------------- | -----------------------------------------------------------------|
| `svf_update(path, value)`                        | Write or overwrite an SVF entry (supports dot-paths)    | `{svf_update("tool_offsets.1.x", 0.1)}`                          |
| `svf_edit_value(path, delta)`                    | Increment/decrement an existing SVF value               | `{svf_edit_value("probe_position.z_offset", -0.05)}`             |
| `gcode_var_update(macro, path, value)`           | same but for gcode variables                            | `{gcode_var_update("CLEAN_NOZZLE", "last_cleaned.t1", 1500)}`    |
| `save_tool_targets_to_variable(macro, variable)` | Save all tool target temps into a dict in another macro | `{save_tool_targets_to_variable("my_macro","saved_temps")}`      |
| `restore_tool_targets_from_variable(macro, var)` | Restore tool temps from a saved dict in another macro   | `{restore_tool_targets_from_variable("my_macro","saved_temps")}` |
> [!NOTE]
> > These all also update the "local" variable inside the `printer.save_variables.variables` or `printer[gcode_macro MACRO].variables`<br>
> > What that means for you: You can just access the results as if you updated them live, even if the actual update happends later.
> > The output must be piped into the template output (dont do `{% set _ = svf_update() %}` as all those return the actual string to save<br>
> > (or nothing on failure)

### Frequently executed checks.

| Macro                              | Description                                           | Example                                 |
| ---------------------------------- | ----------------------------------------------------- | --------------------------------------- |
| `check_homed()`                    | Ensure axes are homed (or auto-home/error)            | `{check_homed()}`                   |
| `check_tc_status()`                | Initialize or error if tool-changer is in fault state | `{check_tc_status()}`               |
| `check_tn_actn()`                  | Verify mounted tool vs. expected probe                | `{check_tn_actn()}`                 |
| `check_ok()`                       | Run all checks (`homed`, `tc_status`, `tn_actn`)      | `{check_ok()}`                      |

### small editors for "internal"

| Macro                              | Description                                           | Example                                 |
| ---------------------------------- | ----------------------------------------------------- | --------------------------------------- |
| `update_tool_probe_from_svf(tn)`   | Apply saved `z_offset` from SVF to the probe object   | `{update_tool_probe_from_svf(1)}`   |
| `update_tool_offsets_from_svf(tn)` | Apply saved `x/y/z` offsets from SVF to tool object   | `{update_tool_offsets_from_svf(1)}` |
| `update_ttbz_from_svf()`           | Update calibrator’s `trigger_to_bottom_z` from SVF    | `{update_ttbz_from_svf()}`          |
> [!NOTE]
> > these all directly access the actual klipper tool/tool_probe objects.


---
