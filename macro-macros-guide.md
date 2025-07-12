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
        { action_respond_info("No templates found to load for import.") }
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
