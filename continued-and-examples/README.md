
# Further reading
in here youll be finding things linked from the readme for further reading/specifics.

---



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
| **escaping/safety**     | `e`, `tojson`, `escape`, `forceescape`, `safe`, `urlencode`, `urlize`, `xmlattr`                                                           |
| **definedness/default** | `d`, `default`                                                                                                                             |
| **sequence/collection** | `batch`, `count`, `first`, `groupby`, `items`, `last`, `length`, `list`, `map`, `random`, `reverse`, `slice`, `sort`, `unique`             |
| **filtering/selecting** | `join`, `reject`, `rejectattr`, `select`, `selectattr`                                                                                     |
| **misc/object/meta**    | `attr`, `indent`, `pprint`                                                                                                                 |
| **other**               | `filesizeformat`                                                                                                                           |

> todo: explain generators, specifcially show issues and usages of `selectattr` centered around "generator" 

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
  
---

### python/built in methods
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

### For loop

things in a for loop:
| **Variable**      | **Description**                           |
| ----------------- | ----------------------------------------- |
| `loop.index`      | 1-based index                             |
| `loop.index0`     | 0-based index                             |
| `loop.revindex`   | 1-based reverse index                     |
| `loop.revindex0`  | 0-based reverse index                     |
| `loop.first`      | `true` if first iteration                 |
| `loop.last`       | `true` if last iteration                  |
| `loop.length`     | total number of items in the sequence     |
| `loop.cycle(...)` | cycling helper (e.g., alternating values) |




