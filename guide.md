# Comprehensive Jinja2 & Klipper Macros Guide

This guide provides a clear and concise breakdown of klipper macros and its involvement of Jinja2

## Table of Contents

1. [Basic](#basic)
2. [getting started](#getting_started)
3. [Advanced](#advanced)
4. [Advanced Advanced](#advanced-advanced)

---

## **Basic**

---
### **getting started**

I think the most basic thing that we need to understand is that jinja2 is a templating engine. 
That means, explained very simply, it is not much more then Notepad++ search and replace.
gcode macros, are very similar in that way. the output of a gcode macro will **always** be text.
your console, the gcode file youre printing, the macros youre running are all doing one thing. inputting text as commands into klipper.
example:
```jinja
[gcode_macro HOME]
gcode:
  G28
```
calling `HOME` in the console, a gcode file, or a macro will always cause the string response `G28` to be printed and "evaluated"

---
### **ninjago**

our templating engine uses `{ }` to evaluate variables, or `{% %}` to evaluate (for us almost exclusively) conditionals.
klipper provides:
* `printer` -> is a snapshot of our current state.
* `params` -> is a key value dict of the parameters passed in (stringified)
* `rawparams` -> the raw string of parameters, including spaces etc..

using jinja we can evaluate those. the printer object contains our more complex objects which hold all information we need.

**all information?** *yes*

this printer object contains not only our configuration (printer.settings.config) but also a snapshot of our current state.

strings in strings out. thats the gist of it.
basic jinja evaluation: `{'G28'}` == `G28` == `"{'G28'}"` jinja evaluates it to the string 'G28' and thats all that klipper cares about.


---
### **Usage of the printer object**

```jinja
[gcode_macro TEMPERATURE_EXAMPLE]
gcode:
  {% set temp = printer.extruder.temperature %}
  RESPOND MSG={"extruder temperature is: ' ~ temp ~ "°C'"}
```
do note in this example that there isnt only one way to do things. `~` is a special operator, think of it like `+` but "forcing" the output to be a string.
`{"RESPOND MSG=extruder temperature is: '" ~ temp ~ "°C'"}` would be equally valid.

`params.TEMP` is always going to be a string. to prevent klipper from erroring with `.` key access in our dict, one may use `.get()` for safer access. (`params.get('TEMP', 220)`) where here, 220 is the fallback.
equal and more "valid" would be to use the `|default()` filter. (`params.TEMP|default(220)`) (mainly because mainsail recognizes this as the "default" value for a parameter and thus shows it in the gui)
oh, dictionary is what it says. its a book, containing our words with their assigned value. 


---
### **Available Filters**

So what do you do with those useless strings?
* Use explicit filters to convert between types (`|int`, `|float`, etc)
some common filters would be:
- `default, int, float, string, round, lower, upper, replace, join, length, count, list, last, abs, max, min, sum, format, trim`

and some more uncommon ones:
- `map, select, reject, rejectattr, sort, dictsort, items, safe, e, escape, title, capitalize, truncate, indent, batch, center, slice, unique, reverse, random, pprint, filesizeformat, wordcount, wordwrap, urlencode, urlize, xmlattr, d`

for building even the simplest macros, you can refer to my search macros to let you browse the printer object a bit which makes it easier to find the variables youre looking for.


---
### **For Loops & Namespacing**

Once youre stringing around with `{% if something is equal not_something %} {something} {% endif %}`
you may want to make things more efficent or deduplicate giant if else endif gunk.
For loops:
```jinja
{% set min_y = printer.configfile.config["stepper_y"]["position_min"]|float %}
{% set max_y = printer.configfile.config["stepper_y"]["position_max"]|float %}
{% set min_x = printer.configfile.config["stepper_x"]["position_min"]|float %}
{% set max_x = printer.configfile.config["stepper_x"]["position_max"]|float %}
{% if params.X < max_x or params.X > min_x %}....
  G0 X{params.X}
```
(you can see this getting repetetive)
Instead you can use a for loop.
```jinja
{% set allowed_move = True %}
{% for axis in ['x', 'y'] %}
  {% if params[axis] > printer.configfile.config["stepper_" ~ axis]["position_max]|float %}
    {% set allowed_move = False %}
  {% endif %}
{% endfor %}
```
If you didnt read from the start you may have missed it, but there are critical errors:
- `params[axis]` is not protected, if X or Y isnt supplied by the user, the macro crashes.
- `params[axis]` returns a string! we cannot compare a `string` to a `float`!
- `{% set allowed_move = False %}` is never being retained... why?
  
---
**Namespacing:**
- a namespace is a window we are in. a for loop is a different "namespace" thus values arent retained outside
- `{% set allowed_move = False %}` would set the local to the for loop "`allowed_move`" to false, and immediately discard it.
```jinja
{% set ns = namespace(counter=0) %}
{% for i in range(5) %}
  {% set ns.counter = ns.counter + i %}
{% endfor %}
RESPOND MSG="{ns.counter}"
=>
// 10
```
a namespace of course isnt the only way around it. we can trick jinja by setting the return type of functions like `append` or `update`
(hint, they return `None` :P)
```jinja
{% set allowed_axis = [] %}
{% for axis in ['x', 'y'] %}
  {% if params[axis]|default(-999)|float < printer.configfile.config["stepper_" ~ axis]["position_max]|float %}
    {% if params.get[axis]|default(-999)|float > printer.configfile.config["stepper_" ~ axis]["position_min]|float %}
      {% set _ = allowed_axis.append(axis) %}
    {% endif %}
  {% endif %}
{% endfor %}
{% if 'x' in allowed_axis %}
G0 X{params.X} 
.....
```


---

## Advanced

### gcode variables
```
[gcode_macro CLEAN_NOZZLE]
variable_cleaned: False
gcode:
  {% if not cleaned %}
    _CLEAN_NOZZLE
  {% endif %}
  SET_GCODE_VARIABLE MACRO=CLEAN_NOZZLE VARIABLE=cleaned VALUE={True}

[gcode_macro PRINT_END]
gcode:
  .....
  SET_GCODE_VARIABLE MACRO=CLEAN_NOZZLE VARIABLE=cleaned VALUE={False}
```
*(to be or not to be.... You may be wondering... why is he evaluating `{False}`? because just `VALUE=False` would be the **string** "False" -> `variable_cleaned: 'False'` would be saved. not the actual boolean `False`. the string 'False' is **truish** in python/jinja that means, we are testing the variable `cleaned` to not be. but a string (non empty) is...)*

### Advanced Variable Types
ok so you can not only save strings in there then right? *yes* they are parsed as python literals.
```
{% dict_last_cleaned = {} %}
{% set _ = dict_last_cleaned.update({'last_clean': printer.print_stats.info.current_layer|int}) %}
{% set _ = dict_last_cleaned.update({'last_time': printer.toolhead.estimated_print_time|round(3)}) } %}
SET_GCODE_VARIABLE MACRO=CLEAN_NOZZLE VARIABLE=cleaned VALUE={dict_last_cleaned}
```
=>
```
[gcode_macro CLEAN_NOZZLE]
variable_cleaned: {'last_clean': 12, 'last_time': 1426.420}
```
to retrieve those, one may access them like `{% set cleaned = printer['gcode_macro CLEAN_NOZZLE'].cleaned %}`

**VERY IMPORTANT ENDING NOTE**
variables **MUST** always be all lowercase!

### more jinja horrors
- **macros** (reusable blocks of Jinja2 code)
```jinja
[gcode_macro HELLO_WORLD]
  {% macro greet(name) %}
    RESPOND MSG="Hello {name}!"
  {% endmacro %}

{greet('World')}
```
// Hello World

a couple gotchas with macros.
- macros always return strings.
- macros always return string (even the newlines!)
- macros are only avalible in our current gcode macro. you cannot `import greet from HELLO_WORLD`
- newlines only become an issue when you shove the result into variables, if you parse the result into commands as the example above does, klipper already cuts those.

what can we do about it?... whitespace control
- `{%- set something = something -%}` will strip the newlines returned from a macro.
- simply never return. you can use any mutable (`namespace`, `list`, `dict`) to hand into the macro, simply modify that instead and discard the mess of newlines returned.



- **cycler**
```jinja
[gcode_macro SHAKE]
  {% set direction = cycler(-1, 1) %}
  {% for _ in range(20) %}
    G0 X{direction}
  {% endfor %}
```
G0 X-1... -> G0 X1..... ->G0 X-1.......
(yeah... it cycles through the "list")

- **joiner/lipsum** (not really needed but its there)
```jinja
{% set comma = joiner(', ') %}
{% for param in params %}{comma()}{param}{% endfor %}
```
-> X=1, Y=1, Z=1 
(doesnt end in a comma, obmitted if last)
`lipsum(n=3)` to generate lorem ipsum text





### Save Variables & Delayed Gcode

Saving state:

```jinja
SAVE_VARIABLE VARIABLE=my_setting VALUE="42" # the string 42
SAVE_VARIABLE VARIABLE=my_setting VALUE={{}} # an empty dict.
```

## Advanced Advanced

### But what about *second printer*?

Directly access Klipper objects and methods:
*(this will blow your brains out)*
```jinja
{% set probe = printer.printer.lookup_object('probe') %}
{% set toolhead = printer.printer.lookup_object('toolhead') %}
{% set triggered = probe.mcu_probe.query_endstop(toolhead.get_last_move_time()) %}
RESPOND MSG="Probe Triggered: {triggered}"
```
Use with caution—direct object access is powerful and essentially equivalent to writing a klipper extra.
for example (not that you should) one may use the above in an LED template to show the probe trigger status.
issues comes in when youre trying to home, and realise that its messing with that too.





---

### Wrapping Up

This guide provides you with the necessary knowledge to write powerful and dynamic Klipper configurations leveraging the full potential of Jinja2 templating. Happy macro crafting!
