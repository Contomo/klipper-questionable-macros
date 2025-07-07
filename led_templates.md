### Simple Display Templates

```ini
[display_template simple_color]
param_r: 1.0
param_g: 0.5
param_b: 0.0
param_w: 0.0
text:
  {param_r},{param_g},{param_b},{param_w}
```

Apply template:

```gcode
SET_LED_TEMPLATE LED=led1 TEMPLATE=simple_color
```
