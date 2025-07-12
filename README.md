
***

# Klipper Macros & Jinja2 ‑ A Practical Guide

> **Strings in ➜ Strings out.** Everything you feed Klipper is just text. Master that, and you master macros.

---
## Table of Contents

### **Beginner**

1. [What *is* a template?](#1-what-is-a-template)
2. [Core objects: `printer`, `params`, `rawparams`](#2-core-objects)
3. [Macro anatomy](#3-macro-anatomy)
4. [Defensive techniques](#4-defensive-techniques)
5. [Surprise Syntax/Style Interlude](#surprise-syntaxstyle-interlude)

### **Medium**

1. [Delayed G-code & persistent variables](#1-delayed-g%E2%80%91code--persistent-variables)
2. [For-loops, namespaces, mutables](#2-for%E2%80%91loops-namespaces-mutables)

### **Advanced**

1. [Writing reusable Jinja *macros*](#1-writing-reusable-jinja-macros)
2. [Complex variable types & advanced filters](#2-complex-variable-types--advanced-filters)
3. [Direct object access](#3-direct-object-access)

### **Creative Mode**

1. [The reactor](#1-the-reactor)
2. [Full filter list & Helpers](#2-full-filter-list--helpers)
3. [Further Reading](#3-further-reading)

---

# Beginner

### 1. What *is* a template?

A Jinja2 template is search–replace on steroids. Feed it *context* → it renders *text*.

```nunjucks
[gcode_macro HOME]
description: Home all axes
gcode:
  G28            # renders exactly this line
```
Calling `HOME` just outputs `G28` to Klipper.

### 2. Core objects
Klipper provides special objects you can access within your macros:

| Object | What it is |
| :--- | :--- |
| `printer` | Snapshot of *everything*: config, sensors, toolhead, temps… (actual values) |
| `params` | Dict of named arguments passed to your macro — **all values are strings**! |
| `rawparams` | The raw text after the macro name (spaces included). |

### 3. Macro anatomy
Logic lives in `{% … %}`. Output placeholders use single braces `{ … }`.

```nunjucks
[gcode_macro TEMP_REPORT]
description: Show current hotend temp
gcode:
  {% set temp = printer.extruder.temperature %}
  RESPOND MSG="Hotend is {temp} °C"
```

### 4. Defensive techniques
Make your macros robust by making parameters optional and checking for objects before you access them.

**Common filters:** `int`, `float`, `default`, `round`, `upper`, `lower`, `replace`, `join`, `length`, `sum`

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
> # Surprise Syntax/Style Interlude
>
> Just like with many things in life, there isn’t only a single path to the goal—the goal being: G-code commands echoed to console. A quick note on how Klipper parses these:
>
> * It splits on `whitespace` or `*` for parameters, and on `=` for values.
>
> Examples:
>
> * `RESPOND MSG=hello`                    ✅ no spaces, no nothing
> * `RESPOND MSG="hello world"`            ✅ space protected with quotes
> * `RESPOND MSG='{"foo":42,"bar":"baz"}'` ✅ valid because the `'` encase it. *(this wont even "compile" if you add it to a macro)*
>
> But why do some commands fail as "malformed"? Here are ways to avoid that:
>
> * Use Jinja’s `{% raw %}` tags to treat content as pure string:
>   * `{% raw %}RESPOND MSG='{"foo":42,"bar":"baz"}'{% endraw %}`
> * Or evaluate with Jinja: *(with `{% set foo_dict = {'foo': 42} %}` done earlier)*
>   * `RESPOND MSG="{foo_dict}"`  
>   * `"{'RESPOND MSG=' ~ foo_dict}"`
>   * `RESPOND MSG="{{'foo': 42}}"` *<- one of the few times double curlys are valid*
>
> **What doesn’t work:**
>   * `RESPOND MSG={foo_dict}` -> gets split as `{'foo':`, `42}` (gibberish to Klipper)
>   * `RESPOND MSG='{foo_dict}'` -> becomes `'{'`foo`': 42}'` (bad quotes)
>   * ...and many more variants.
>
> **Tip:** Always keep in mind Klipper’s autism with spaces and improper quoting.
>   * use `|tojson()` to `"` -> `'` so that when you add those `"` they dont get closed prematurely.
>   * use `|pprint()` to convert it into an escaped format.



---
# Medium

### 1. Delayed G‑code & persistent variables
*   **Delayed G‑code**: Really nothing to add which isnt already covered by the [Klipper documentation](https://www.klipper3d.org/Command_Templates.html#delayed-gcodes).
*   **Variable Storage**: Klipper offers two ways to store variables: `SET_GCODE_VARIABLE` for runtime state and `SAVE_VARIABLE` for persistent storage.

**`SAVE_VARIABLE`**
This stores Python literals in a separate file (e.g., `variables.cfg`), which persists after a restart. It's often referred to as `svf` (save variable file) or `svv` (save variables variables).
```nunjucks
[gcode_macro STORE_VALUE]
description: Remember the last Z offset
gcode:
  SAVE_VARIABLE VARIABLE=last_z VALUE={printer.toolhead.position.z}
```
> *A small comment on that:*
> *position is a coordinate type, that means `position[0]` == `position['x']` ≈≈ `position.x`*
> <sup>*(only ≈ because `.` cant be used to dynamically grab stuff)*</sup>

Retrieve the value later:
```nunjucks
[gcode_macro SHOW_LAST_Z]
description: Report stored Z offset
gcode:
  {% set z = printer.save_variables.variables.last_z|default('unset') %}
  RESPOND MSG="Last Z = {z}"
```

**G-code variables (`SET_GCODE_VARIABLE`)**
These are stored within the macro's definition and reset on restart.
```nunjucks
[gcode_macro CLEAN_NOZZLE]
variable_cleaned: False   # parsed as *boolean*
gcode:
  {% if not cleaned %}
    _CLEAN_NOZZLE
    SET_GCODE_VARIABLE MACRO=CLEAN_NOZZLE VARIABLE=cleaned VALUE=True 
  {% endif %}
```
> *If someone (you) fumbles one day and accidentally does `variable_cleaned: "False"` (a **string**) it will never clean because it is **truthy** – the `if not cleaned` guard will fail. Watch out.*

### 2. For‑loops, namespaces, mutables
This example uses a `for-else` loop to process axis parameters and raises an error if no valid axes are provided.
- **List (works best with this example)**
  ```nunjucks
  {% set axis_params = [] %}
  {% for axis in ['X','Y','Z'] if axis in params %}
    {% set _ = axis_params.append(axis ~ '=' ~ params[axis]) %}
  {% else %}
    {action_raise_error("Please provide at least one of X, Y or Z")}
  {% endfor %}
  G0 {axis_params|join(' ')}
  ```
- **Dict (slightly more cursed, but gets the point across)**
  ```nunjucks
  {% set axis_params = {} %}
  {% for axis in ['X','Y','Z'] if axis in params %}
    {% set _ = axis_params.update({ axis: params[axis] }) %}
  {% else %}
    {action_raise_error("Please provide at least one of X, Y or Z")}
  {% endfor %}
  G0{% for a, v in axis_params.items() %} {' ' ~ a}={v}{% endfor %}
  ```
> *wait..  is that an `if` and `{% else %}` tag in a for loop? can i always use if/else?*
>   - `else` kinda... yes. the else branch only runs if the for loop didnt run **at all**, if it ran once or more. never hit.
>   - `if` yes. putting an if inside the for loop definition is a very clear way to show `{% break %}` (since there is no break for us, we can use a namespace flag, or the filling/emptying as a list to define a "stop" definition.

---
# Advanced

### 1. Writing reusable Jinja *macros*
Not to be confused with G-code macros, these are for reusable template logic.
  ```nunjucks
  [gcode_macro GREET]
  description: Say hi
  gcode:
    {% macro greet(name) %}
      RESPOND MSG="Hello {name}!"
    {% endmacro %}
  
    {greet('World')}  
  ```
* **Gotchas:**
   *   Macros return strings only, newlines are kept. Use `{%-` or `-%}` to trim whitespace.
   *   Macros can call themselves recursively. Be careful to avoid infinite loops, which can stall or crash Klipper.
* **Cope:**
   * hand in mutables like dict, namespace or list modify them in place, and throw those newlines into the gcode output :P

### 2. Complex variable types & advanced filters
- **Filters (even worse example for this use case)**
```nunjucks
{% set axis_params = params|dictsort|selectattr(0,'in',['X','Y','Z']) %}
G0 {axis_params|map('join','=')|join(' ')}
```
**Breakdown:**
1.  **`dictsort`** -> `[ ('X', 10), ('Y', 5.5), … ]`
    <sup>*Like `items`, but always sorted by key.*</sup>
2.  **`selectattr(0, 'in', ['X','Y','Z'])`** -> selects all that have tuple index `0` == any of `X`, `Y`, or `Z`.
3.  **`map('join', '=')`** -> `("X", 10)` becomes `["X=10", "Y=5.5", ...]`.
4.  **`join(' ')`** -> joins the list with a space per entry, resulting in `"X=10 Y=5.5"`.

### 3. Direct object access
> *(but what about second printer?)*

**The following is closer to writing a Klipper extra than a simple macro.**

The standard `printer` object is a `GetStatusWrapper`, which pulls data from `get_status()` calls. It only lets you read what an extra wants you to read. For more power, you can access `printer.printer`, which is the main printer instance. From here, you can access Klipper's internal objects and methods.

**Example: Direct Endstop Query**
```nunjucks
{% set probe        = printer.printer.lookup_object('probe') %}
{% set toolhead     = printer.printer.lookup_object('toolhead') %}
{% set is_triggered = probe.mcu_probe.query_endstop(toolhead.get_last_move_time()) %}
RESPOND MSG="Probe triggered: {is_triggered}"
```
> **Note:** This is a powerful technique. Before you start playing, be aware that running this while another command (like `G28`) needs the same hardware will cause issues.

---
# Creative Mode

### 1. The `reactor`
The `reactor` is Klipper's main timing orchestrator. You can access it via `printer.printer.reactor`. This is the first object youll likely be interrested in for timing purposes.

**Example: stalling it with a long macro**
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

> PI N=100000<br>
> Took: 0.116444 seconds<br>
> Pi is just about 3.141583


**If you made it until here, you should have no problems understanding what this does, if you don't, good job. if you do, i hope you have trust issues.**
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

### 2. Full filter list & Helpers
This is the list of filters available in Klipper's sandboxed Jinja2 environment:
`abs`, `attr`, `batch`, `capitalize`, `center`, `count`, `d`, `default`, `dictsort`, `e`, `escape`, `filesizeformat`, `first`, `float`, `forceescape`, `format`, `groupby`, `indent`, `int`, `items`, `join`, `last`, `length`, `list`, `lower`, `map`, `max`, `min`, `pprint`, `random`, `reject`, `rejectattr`, `replace`, `reverse`, `round`, `safe`, `select`, `selectattr`, `slice`, `sort`, `string`, `striptags`, `sum`, `title`, `tojson`, `trim`, `truncate`, `unique`, `upper`, `urlencode`, `urlize`, `wordcount`, `wordwrap`, `xmlattr`

*   **cycler** – alternate values:
    ```nunjucks
    {% set dir = cycler(-1, 1) %}
    {% for _ in range(4) %} G0 X{dir.next()}{% endfor %}
    ```
*   **joiner** – print delimiter *except* after the last item:
    ```nunjucks
    {% set comma = joiner(', ') %}
    {% for k, v in params.items() %}{% if comma.first %}{% endif %}{comma()}{k}={v}{% endfor %}
    ```

### 3. Further Reading
*   Klipper docs: [`Config_Reference.html#gcode_macro`](https://www.klipper3d.org/Config_Reference.html#gcode_macro) ↗
*   Jinja2 docs: [`jinja.palletsprojects.com`](https://jinja.palletsprojects.com/en/3.1.x/templates/) ↗
*   [Kevin O’Connor’s toolchanger extras](https://github.com/Klipper3d/klipper/blob/master/klippy/extras/toolhead.py) ↗

Happy macro crafting
