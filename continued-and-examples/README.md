
# Further reading
in here youll be finding things linked from the readme for further reading/specifics.

---



[Cheat Sheet](#Cheat-Sheet)
  - [Filters](#Filters)
  - [Tests](#Tests)
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
| `Identity`                   | `sameas`                              | Object identity (is the same object) |
| `Containment`                | `in`                                  | Membership (item in container)       |
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

comparison (pick your poison)


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




