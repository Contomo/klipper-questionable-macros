# Klipper Macros & Jinja2 ‑ A Practical Guide

> **Strings in ➜ Strings out.**  Everything you feed Klipper is just text. Master that, and you master macros.


---

## Table of Contents

### Beginner

1. What *is* a template?
2. Core objects: `printer`, `params`, `rawparams`
3. Basic macro anatomy
4. Defensive techniques

### Medium

1. Delayed G‑code & variable storage
2. For‑loops, namespaces, mutables
3. Writing reusable Jinja *macros*
4. Utility helpers (`cycler`, `joiner`, …)

### Advanced

1. Complex variable types
2. Direct object access
3. Performance / caveats

---

# Beginner

## 1   What is a template?

A Jinja2 template is search–replace on steroids.  Feed it *context* → it renders *text*.

```nunjucks
[gcode_macro HOME]
description: Home all axes
gcode:
  G28            # renders exactly this line
```

Calling `HOME` just outputs `G28` to Klipper.

## 2   Core objects
### Special objects `printer`, `params`, `rawparams` *(`self`)*

| Object      | What it is                                                                 |
| ----------- | -------------------------------------------------------------------------- |
| `printer`   | Snapshot of *everything*: config, sensors, toolhead, temps… (actual values)|
| `params`    | Dict of named arguments passed to your macro — **all values are strings**! |
| `rawparams` | The raw text after the macro name (spaces included).                       |

## 3   Macro anatomy

```nunjucks
[gcode_macro TEMP_REPORT]
description: Show current hotend temp
gcode:
  {% set temp = printer.extruder.temperature %}
  RESPOND MSG="Hotend is {temp} °C"
```

*Logic* lives in `{% … %}`.  Output placeholders use single braces `{ … }`.

## 4   Defensive techniques

Common filters you’ll use daily:
`int`, `float`, `default`, `round`, `upper`, `lower`, `replace`, `join`, `length`, `sum`

* Try to make `params` optional using the `|default()` filter.
* Try to first check and only then access.
```nunjucks
[gcode_macro TEMP_REPORT]
description: Show current hotend temp
gcode:
    {% set extruder = params.EXTRUDER|default('extruder') %}
    {% if extruder in printer %}
        {% set temp = printer[extruder].temperature %}
        RESPOND MSG="Hotend is {temp} °C"
    {% else %}
        RESPOND MSG="extruder not found"
    {% endif %}
```
---
# Syntax/style interlude
Just like with many things in life, there isnt only a single path to the goal, the goal being, gcode commands echod to console.
a small note on that however.

as mentioned earlier, string in string out.
Klippers parser is actually pretty simple. it splits on whitespace to get a parameter, and on = to get that value for that parameter.
* `RESPOND MSG=hello` -> valid, no space to confuse it.
* `RESPOND MSG="hello world"` -> valid, cause quotes contained it.
* `RESPOND MSG='{"foo":42,"bar":"baz"}'` -> also valid (to the parameter parser! Not our template parser neither to jinja!)
So why isnt it? how can i ensure i dont get a "malformed command"?
theres many ways to get this work.
- Jinjas `{% raw %}` tags (they treat the following content purely as "string" *(or also, ignore)*
 `{% raw %}RESPOND MSG='{"foo": 42,"bar": "baz"}'{% endraw %}`
- simply evaluating it with jinja. (many ways)
  1. `"{'RESPOND MSG=' ~ {'foo': 42}}"`
  2. `RESPOND MSG="{{'foo': 42}}"` *<- (one of the very few times youll see double curlies)*
  3. `RESPOND MSG="{foo_dict}"` *(of course previously `{% set foo_dict = {'foo': 42} %}`)*
What doesnt work and why
  1. `RESPOND MSG={foo_dict}` -> {'foo':` `42}  *(klipper splits into the parameters `{'foo':`, `42` which is gibberish)*
  2. `RESPOND MSG='{foo_dict}'` -> '{'foo': 42}'  *(not immediately obvious but `'{'`foo`': 42}'` are indeed bad quotes)*
  3. *many many more....*
---

# Medium

## 1   Delayed G‑code & persistent variables
### Delayed G‑code 
* Really nothing to add which isnt already covered by the [klipper documentation](https://www.klipper3d.org/Command_Templates.html#delayed-gcodes)

### [SAVE\_VARIABLE](https://www.klipper3d.org/G-Codes.html#save_variables) / [SET\_GCODE\_VARIABLE](https://www.klipper3d.org/G-Codes.html#set_gcode_variable)
---
### SVF
- stores arbitrary Python literals in your gcode macros variable or save variables files variables
- often see referred to it as `svf`(save variable/s file) or `svv`(save variables variables)
- once youve mastered parsing dicts, you may find yourself failing to save simple strings. that is because `VALUE="{yourstring}"` is not techincally a string... `VALUE="'{yourstring}'"` would be a string. *(you can escape quotes using `\'` or `\"` btw)*
- *a small note on that save variables file. can be any type. (most common is variables.cfg) just make sure to not accidentally include that file in your printer.cfg or klipper will complain.*

```nunjucks
[gcode_macro STORE_VALUE]
description: Remember the last Z offset
gcode:
  SAVE_VARIABLE VARIABLE=last_z VALUE={printer.toolhead.position.z}
```
*once again a small comment on that:*
- *position is a coordinate type, that means `position[0]` == `position['x']` ≈≈ `position.x`*
  - <sup>*(only ≈ because . lookup doesnt work with `|default()` as a fallback)*</sup>
  
Retrieve later:
```nunjucks
[gcode_macro SHOW_LAST_Z]
description: Report stored Z offset
gcode:
  {% set z = printer.save_variables.last_z|default('unset') %}
  RESPOND MSG="Last Z = {z}"
```

### gcode variables
```nunjucks
[gcode_macro CLEAN_NOZZLE]
variable_cleaned: False   # parsed as *boolean*
gcode:
  {% if not cleaned %}
    _CLEAN_NOZZLE
    SET_GCODE_VARIABLE MACRO=CLEAN_NOZZLE VARIABLE=cleaned VALUE=True 
  {% endif %}
```
* *If someone (you) fumbles one day and accidentally does `variable_cleaned: "False"` (a **string**) it will never clean because it is **truthy** – the `if not cleaned` guard will fail.  Watch out.*

## 2   For‑loops, namespaces, mutables
Presenting the power of filters, mutables, and the for loops which can profit insanely from them.
- List *(works best with this example)*
    ```nunjucks
    {% set axis_params = [] %}
    {% for axis in ['X','Y','Z'] if axis in params %}
      {% set _ = axis_params.append(axis ~ '=' ~ params[axis]) %}
    {% else %}
      {action_raise_error("Please provide at least one of X, Y or Z")}
    {% endfor %}
    G0 {axis_params|join(' ')}
    ```
- Dict *(slighty more cursed, but gets the point across)*
    ```nunjucks
    {% set axis_params = {} %}
    {% for axis in ['X','Y','Z'] if axis in params %}
      {% set _ = axis_params.update({ axis: params[axis] }) %}
    {% else %}
      {action_raise_error("Please provide at least one of X, Y or Z")}
    {% endfor %}
    G0{% for a, v in axis_params.items() %} {' ' ~ a}={v}{% endfor %}
    ```

---

# Advanced

- Filters *(even worse example for this use case)*
    ```nunjucks
    {% set axis_params = params|dictsort|selectattr(0,'in',['X','Y','Z']) %}
    G0 {axis_params|map('join','=')|join(' ')}
    ```
    1. **`dictsort`**
       `[ ('X', 10), ('Y', 5.5), … ]`  
       <sup>*Like `items`, but always sorted by key.*</sup>
       
    2. **`selectattr(0, 'in', ['X','Y','Z'])`**
        selects all that have tuple index `0` == any of `X, Y, or Z.`
       
    3. **`map('join', '=')`**
        `("X", 10)` -> `["X=10", "Y=5.5", ...]`
       
    4. **`join(' ')`**
        joins the list with a space per entry. -> `"X=10 Y=5.5"`.
       
## 3   Reusable Jinja *macros*

```nunjucks
[gcode_macro GREET]
description: Say hi
gcode:
  {% macro greet(name) %}
    RESPOND MSG="Hello {name}!"
  {% endmacro %}

  {greet('World')}  
```
- gotchas
  - macros return strings only, newlines are kept
    - use `{%-` and / or `-%}` to trim unwanted newlines when capturing macro output into a variable.
    - cast results back to types (`|float`, `|int`, or if you like suffering, use filters like `|map` to convert back to lists/dicts)
    - hand in mutables (dict, list or namespaces)
  - unlike gcode macros... these can call themself. be careful if you do intend to use this tho, maybe add a global that keeps track of depth and abort if too much (can stall klipper or crash the "execution").
      
---
# Expert

## 2   Direct object access
(*but what about second printer?*)
- `printer.printer` -> your actual main printer instance. the regular `printer` is just a `GetStatusWrapper` which pulls all the stuff from the klippy extras `get_status()` calls. it only lets you read what the extra wants you to read... how boring right?
- since `printer.printer` is your highest level instance. you can access everything from it. all the internals of the klippy extras, all the objects defined which dont fall under the boring `int, float, string, list, dict....etc` now were actually playing with weird abstract python objects.
- `reactor` is comperable to what the `printer` object is to our `printer.printer` of importance. its our main timing orchestrator. the first cool magic trick would be to call `reactor.monotonic()` to get its current value.

*(note: i am not a mathmagician)*
```nunjucks 
[gcode_macro PI]
gcode:
    {% set start_time, n = printer.printer.reactor.monotonic(), params.N|default(1000)|int %}
    {% set ns = namespace(sum=0.0) %}
    {% for i in range(n) %}
        {% set term = 1.0 / (2 * i + 1) %}
        {% set ns.sum = ns.sum + term * (1 if (i % 2 == 0) else -1) %}
    {% endfor %}
    {% set pi_val = ns.sum * 4 %}
    {% set pi_str = '%.6f' % pi_val %}
    {% set duration = printer.printer.reactor.monotonic() - start_time %}
    {action_respond_info("Took: %.6f seconds" % duration)}
    RESPOND MSG="Pi is just about {pi_str}"
```

```
                PI N=100000
                Took: 0.116444 seconds
                Pi is just about 3.141583
```



Powerful but possibly **dangerous**, it is closer to a klipper extra then it is to a simple macro.

This section is the reason you may have trust issues from now on.
```nunjucks
[delayed_gcode ANGER_MANAGEMENT_TESTER]
initial_duration: 0.1
gcode:
    {% if printer.toolhead.estimated_print_time < 1 %}
        {% set next_run = range(2*3600, 16*3600)|random %}
        UPDATE_DELAYED_GCODE ID=ANGER_MANAGEMENT_TESTER DURATION={next_run}
    {% else %}
        {% set _ = printer.printer.reactor.end() %}
    {% endif %}
```


```nunjucks
{% set probe        = printer.printer.lookup_object('probe') %}
{% set toolhead     = printer.printer.lookup_object('toolhead') %}
{% set is_triggered = probe.mcu_probe.query_endstop(toolhead.get_last_move_time()) %}
RESPOND MSG="Probe triggered: {is_triggered}"
```
/\ 
This seems very good, before you start playing, a note on this example specifically. `G28` will also require the endstop. if you run this macro whole youre homing, it will cause issues.

---

### Full filter list 

`abs`, `attr`, `batch`, `capitalize`, `center`, `count`, `d`, `default`, `dictsort`, `e`, `escape`, `filesizeformat`, `first`, `float`, `forceescape`, `format`, `groupby`, `indent`, `int`, `items`, `join`, `last`, `length`, `list`, `lower`, `map`, `max`, `min`, `pprint`, `random`, `reject`, `rejectattr`, `replace`, `reverse`, `round`, `safe`, `select`, `selectattr`, `slice`, `sort`, `string`, `striptags`, `sum`, `title`, `tojson`, `trim`, `truncate`, `unique`, `upper`, `urlencode`, `urlize`, `wordcount`, `wordwrap`, `xmlattr`, `d`

* **cycler** – alternate values:

  ```nunjucks
  {% set dir = cycler(-1, 1) %}
  {% for _ in range(4) %} G0 X{dir} {% endfor %}
  ```
* **joiner** – print delimiter *except* after last item:

  ```nunjucks
  {% set comma = joiner(', ') %}
  {% for k, v in params.items() %}{comma()}{k}={v}{% endfor %}
  ```
* `lipsum(n=3)` Lorem ipsum filler (i dunno...)

### Further Reading
* Klipper docs: `docs.klipper3d.org/Config_Reference.html#gcode_macro` ↗
* Jinja 2 docs ↗
* [Kevin O’Connor’s toolchanger extras](https://github.com/Klipper3d/klipper) ↗
Happy macro crafting!



