
# Further reading
in here youll be finding things linked from the readme for further reading/specifics.

---


* `printer`, `eventtime`, and `cache` are the only "real" (non-dunder) attributes—expected in Klipper’s root object.




[Cheat Sheet](#Cheat-Sheet)
  - [Filters](#Filters)
  - [Tests](#Tests)
  - [methods](#python/built-in-methods)
  - [Loop specifics](#For-loop)

## **Cheat Sheet**

---

### Filters

| **Category**            | **Filters**                                                                                                                                |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **number**              | `abs`, `float`, `int`, `max`, `min`, `round`, `sum`                                                                                        |
| **string**              | `trim`, `format`, `replace`, `upper`, `lower`, `center`, `string`, `striptags`, `title`, `truncate`, `capitalize`, `wordcount`, `wordwrap` |
| **escaping/safety**     | `e`, `tojson`, `xmlattr`, `escape`, `forceescape`, `safe`, `urlencode`, `urlize`                                                           |
| **definedness/default** | `d`, `default`                                                                                                                             |
| **sequence/collection** | `batch`, `count`, `first`, `groupby`, `items`, `last`, `length`, `list`, `map`, `random`, `reverse`, `slice`, `sort`, `unique`             |
| **filtering/selecting** | `join`, `reject`, `rejectattr`, `select`, `selectattr`                                                                                     |
| **misc/object/meta**    | `attr`, `indent`, `pprint`                                                                                                                 |
| **other**               | `filesizeformat`                                                                                                                           |

<br>

If you read the readme you likely saw an example on how to [join those parameters](#2-complex-variable-types--advanced-filters) back into something usable.
Here is a more practical example that uses filters.

```nunjucks
[gcode_macro ORIGINAL]
rename_existing: OLD_ORIGINAL
gcode:
  {% set bar = params.pop('FOO', none) %}
  {% if bar is not none %}
    # your code before the other one runs
  {% endif %}
  THE_OLD_HIGHJACKED_MACRO {params|xmlattr}
```

> *(ORIGINAL)*     `FOO=2 BAZ="WEIRD STRING" FLOOP=11`<br>
> *(params)* -> ` {'FOO': '2', 'BAZ': 'WEIRD STRING', 'FLOOP': '11'}`<br>
> *(pop)* ->     `{'BAZ': 'WEIRD STRING', 'FLOOP': '11'}`<br>
> *(xmlattr)* ->   `BAZ="WEIRD STRING" FLOOP="11"`<br>

[KAMP](https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging/blob/b0dad8ec9ee31cb644b94e39d4b8a8fb9d6c9ba0/Configuration/Smart_Park.cfg#L13-L15)
has a good example of how one can compress many iterative lines down into some single filters.<br>
(`{% set all_points = printer.exclude_object.objects | map(attribute='polygon') | sum(start=[]) %}`)

---

### generators
generators are whats being returned when you do a filter call like "select" or "cycle",<br>
they are like **a stable platonic coworker relationship**
- Interaction is one-pass and shallow by design.
- appear to be a list or dict at a distance, but are neither.
- Not meant for introspection or serialization.
- are purely theoretical
- For anything deeper or repeatable, convert to a list.
  
<br>

ie:
 - `some_list|select|string` -> `<generator object selectattr at 0x7f0b...>` <br>
<br>

> Some internals would be:
> - `send`, `throw`, `close`<br>
>   *(Used to interact with Python generators “from outside” (advanced control flow))*
> - `gi_code` `gi_yieldfrom` `gi_running` `gi_frame` `gi_suspended`<br>
>     *(Introspection for debugging tools, not end-user code.)*
> None of them however serve any purpose for our context here.
    
---

### Tests

| **Test**                     | **Associates / Operators / Aliases**  | **Description**                      |
| ---------------------------- | ------------------------------------- | ------------------------------------ |
| `string`                     | lower, upper, escaped                 | String-related tests                 |
| `mapping`                    | iterable                              | Mapping/dict type test               |
| `iterable`                   | sequence, string                      | Iterable related to sequence         |
| `sequence`                   | iterable, string                      | Sequence related to iterable         |
| `boolean`                    | true, false, defined, undefined, none | Boolean and definedness tests        |
| `defined`                    | undefined, none                       | Definedness presence tests           |
| `undefined`                  | defined, none                         | Definedness absence tests            |
| `none`                       | defined, undefined                    | Null-like value test                 |
| `number`                     | integer, float, divisibleby           | Numeric type tests                   |
| `integer`                    | odd, even                             | Integer-specific tests               |
| `divisibleby`                | number, float, int                    | Divisibility test                    |
| `odd`, `even`                | integer                               | Oddness test                         |
| `sameas`                     |                                       | the same (refrence in memory!)       |
| `in`                         | sequence, string                      | Membership (item in container)       |
| `callable`, `filter`, `test` |                                       | filters, macros, functions           |
| **Usual tests**              |                                       |                                      |
| `Equality`                   | `==`, `eq`, `equalto`                 | Value equality                       |
| `Inequality`                 | `!=`, `ne`                            | Value inequality                     |
| `Greater Than`               | `>`, `gt`, `greaterthan`              | Greater than comparison              |
| `Greater or Equal`           | `>=`, `ge`                            | Greater than or equal to comparison  |
| `Less Than`                  | `<`, `lt`, `lessthan`                 | Less than comparison                 |
| `Less or Equal`              | `<=`, `le`                            | Less than or equal to comparison     |

They are all just "tests", the table/list just shows a broad use case organization.
for example, mapping, iterable, string are all sequences.

**Specifics/common mistakes to avoid**
- `number`<br>
  number here refers to the type. `float` and `int` are numbers, `'0'` (the string) does not pass that test.
- `sequence`, `iterable`<br>
  they can both come across as confusingly similar, iterable just means "can you for loop over this"<br>
  a small note that, strings are sequences. you can access characters in a string with list index,<br>
  which is also why the `in` test works. `if 'ber' in 'num`ber`'`

  
---

### [python/built in methods](https://docs.python.org/3/library/stdtypes.html#str.format)
Since Jinja is purely string, and many things from klipper come in as strings (rawparams, params dictionary) some of the string methods can be very useful for us to convert between strings and types that are easier to work with.
- `isdecimal`, `isdigit`, `isnumeric`: pretty similar, true or false, more like a test then a method.
- 

- `format`, `format_map`: .format(**data) == .format_map(data) *(see [value unpacking](url))*<br>
  `"extruder: {extruder} is at {temp}°C".format(extruder='t1': temp=25.0)` -> `extruder: T1 is at 25°C`

| **Type**       | **High Frequency / Essential**                                                                            | **Moderate Use**                                                                 | **Rare / Specialized**                                                                                                                                                                                                                                                                          |
| -------------- | --------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **string**     | `strip`, `split`, `join`, `lower`, `upper`, `startswith`, `endswith`, `isdecimal`, `isdigit`, `isnumeric` | `capitalize`, `title`, `count`, `replace`, `format`, `zfill`, `rstrip`, `lstrip` | `encode`, `casefold`, `expandtabs`, `swapcase`, `translate`, `removeprefix`, `removesuffix`, `isascii`, `islower`, `isupper`, `istitle`, `isspace`, `isalpha`, `isalnum`, `isidentifier`, `isprintable`, `format_map`, `maketrans`, `rfind`, `rindex`, `ljust`, `rjust`, `splitlines`, `rsplit` |
| **list**       | `append`, `pop`, `remove`, `index`, `count`, `clear`, `copy`                                              | `extend`, `sort`, `reverse`, `insert`                                            | —                                                                                                                                                                                                                                                                                               |
| **dictionary** | `get`, `setdefault`, `pop`, `keys`, `items`, `values`, `update`, `clear`, `copy`                          | —                                                                                | `popitem`, `fromkeys`                                                                                                                                                                                                                                                                           |
| **tuple**      | `index`, `count`                                                                                          | —                                                                                | —                                                                                                                                                                                                                                                                                               |
| **int**        | `bit_length`, `bit_count`                                                                                 | `to_bytes`, `from_bytes`                                                         | `conjugate`, `as_integer_ratio`, `denominator`                                                                                                                                                                                                                                                  |
| **float**      | `is_integer`                                                                                              | `hex`, `fromhex`                                                                 | `conjugate`, `as_integer_ratio`                                                                                                                                                                                                                                                                 |






---

### For loop ([best read that stuff here](https://jinja.palletsprojects.com/en/stable/templates/#for))

things in a for loop:
| **Variable**            | **Description**                           |
| ----------------------- | ----------------------------------------- |
| `loop.index`, `loop.revindex`       | 1-based (reverse)index        |
| `loop.index0`, `loop.revindex0`     | 0-based (reverse)index        |
| `loop.first`, `loop.last`           | `true`/`false`                |
| `loop.previtem`, `loop.nextitem`    | `ìtem`/`undefined`            |
| `loop.length`           | total number of items in the sequence     |
| `loop.cycle(...)`       | cycling helper (e.g., alternating values) |




