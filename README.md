# klipper-questionable-macro-helpers
Some macro helpers for klipper/klipper-toolchanger

## How to use: 
```ini
[gcode_macro DO_SOMETHING_WITH_CFG_HELPERS]
gcode:
    {% set _lib_cfg = printer.configfile.settings['gcode_macro _save_config_helper'].gcode %}
    {% set save_cfg = printer.printer.lookup_object('gcode_macro').env.from_string(_lib_cfg, globals=self._TemplateReference__context).module %}

    {save_cfg.save_config_unstage('stepper_y', 'rotation_distance')}
    {save_cfg.save_config_stage('stepper_x', 'rotation_distance', 40.999)}
```

 why is it called "questionable"? because were passing a  `_TemplateReference__context` from our `self` which has clearly been name mangled.
technically, if you helpers dont require context, you can omit the globals. (or pass in your own even!) for example, just passing printer, action respond stuff, would likely suffice.


pro tip, you can add 
```ini
    {%- if rawparams -%}
        RESPOND MSG="returned: {self._TemplateReference__context[params.MACRO](params.DATA)}"
    {%- endif -%}
```
at the bottom of your gcode macro macro templates to be able to test the macros in them.
