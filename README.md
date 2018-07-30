# Dissect 
Dissect splits a string into its parts. A dissect implementation compares a string against a pattern and then splits the string based on the pattern rules. This specification defines the expected behavior for dissect implementations. 

# Terms:
* pattern (or tokenizer) - the pattern used to define how to split a string. For example `"%{timestamp} %{+timestamp} %{+timestamp} %{logsource} %{program}[%{pid}]: %{message}`.   
* key - the part of the string to match, identified by `%{key}`.  
* delimiter - the part of the string to NOT match. 
* modifier - An special instruction found inside the dissect key to change the behavior. 
* parser - the software that implements this specification to split the string.

# Basic example:
* pattern: ```%{a} %{b},%{c}```
* string: ```foo bar,baz``` 
* result: ```a=foo, b=bar, c=baz```

This pattern has 3 keys: `a`,  `b`, and `c`  , and two delimiters ` ` (space)  and `,` (comma).  When the parser is run against the string with the given pattern, the result is a set of key/value pairs.  The parser searches for the when the delimiter in the pattern matches the delimiter in the string. 

In this example, search the string for ` ` (space), the first delimiter. Found a space in the string, so assign the a key equal to everything up until the space (but not including). So a=foo. The next delimiter is `,` (comma) search for comma in the string, found it, so assign b=bar. No more delimiters, so assign c to the remainder of the string (baz).   

# Specification
1. Pattern Specification
    1. A dissect pattern must contain at least one key  
    2. A dissect pattern may have leading and trailing and delimiters
    3. A dissect pattern may have multiple delimiters of different characters of different lengths. 
    4. A dissect patten must contain unique key names unless the modifier allows or requires duplicated key names.
    5. A dissect pattern may not use `%`  or `{` or `}`  as delimiters
2. Key specification
    1. A dissect key must start with `%{` and end with `}`
    2. A dissect key may have a name, e.g. `%{key_name}` and it must be able to be encoded as UTF-8.
    3. A dissect key may have an empty name e.g. `%{}`, this is called a skip key and must not be included in the final results.
    4. A dissect key name may not have any of the modifiers characters as part of the name. 
    5. A dissect key may have modifiers to the left or the right, or left and right of the key name. 
3. Modifier specification
    1. A dissect modifier must be defined inside the dissect key, to the left or right the key name.
    2. Multiple dissect modifiers per key may be allowed.
    3. `->`: Right padding ignore - instructs the parser to ignore repeating consecutive repeating delimiters to the right of the key.  This key must be placed to the right of the key name and is allowed to co-exist with any other modifiers and must always be the furthest right modifier. _see example below_
    4. `+` Append -  instructs the parser to append this key's value to the value to the prior key (left to right) with the same name. This key must be placed to the left of the key name. _see example below_
    5. `+` and `/n` Append with order - instructs the parser to append this key's to the value of the prior key with the same name based on order. The `+` must be placed on the left of the key name and `/n` placed to the right of the key name, where n = order.  The order must start at `1`. _see example below_
    6. `?` - Named skip key  instructs the parser to not include this result in the final result set. Behaves identical to an empty skip key `%{}` but may be used to help with human readability. This key must be placed to the left of the key name. _see example below_ 
    7. `*` and `&` reference modifiers. This modifier requires two keys with the same name present in the dissect pattern. One key with the `*` and another with the `&`. This instructs the parser that the value discovered by the `*` is to be used as the key name for the value discovered by the corresponding `&` key. These modifiers must be placed on the left of the key name. _see example below_
4. Parser specification 
    1. A dissect parser must not allow partial matches. All delimiters must be present in string, and all keys must have a corresponding value.
    2. A dissect parser must support an empty key `%{}` (skip key) as valid match, but not include the result in the result set. 
    3. A dissect parser must be able to parse any string that can be encoded as UTF-8
    4. A dissect parser must match the leading and trailing delimiters if present in the dissect pattern. 
    5. A dissect parser must allow the last key of a pattern to match the remainder of the string without additional modifiers.  _see example below_
    6. A dissect parser must treat consecutive repeating delimiters as valid empty matches unless instructed otherwise by modifiers. _see example below_
    7. A dissect parser must allow a user specified string to use as the value between append operations. _see example below_
    8. A dissect parser must support multiple charachter delimiters. 
    9. A dissect parser result set must be string/string key value pairs. 
    10. A dissect parser must support all modifiers defined by they specification. 

# Examples:

### Right padding modifier `->`
* pattern: ``` %{a->} %{b} %{c}```
* string: ```foo         bar baz```
* result: ```a=foo, b=bar, c=baz```

In the above example, the delimiter is ` ` (space), the `->` instructs the parser to skip all of the consecutive repeating` ` to the right of  `a`

* pattern: ``` %{a->},%{b},%{c}```
* string: ```foo,,,,bar,baz```
* result: ```a=foo, b=bar, c=baz```

In the above example, the delimiter is `,` (comma) and the `->` instructs the parser to skip all of the consecutive repeating `,` to the right of `a`

Multi-character delimiters must be supported.

* pattern: ``` %{a->},:%{b},%{c}```
* string: ```foo,:,:,:,:bar,baz```
* result: ```a=foo, b=bar, c=baz```

Empty skip key with right padding must be supported. 

* pattern: ```%{->},%{b},%{c}```
* string: ```foo,,,,bar,baz```
* result: ```b=bar, c=baz```

### Append modifier `+`
* pattern: ```%{a} %{+a} %{+a}```
* string: ```foo bar baz```
* result: ```a=foobarbaz```

In the above example the, the values are append in left to right order to the result. 

A user specified append string must be supported. Assume the user define the separator to be `, ` (comma space)

* pattern: ```%{a} %{+a} %{+a}```
* string: ```foo bar baz```
* result: ```a=foo, bar, baz```

### Append modifier with order `+` with `/n`
* pattern: ```%{a} %{+a/2} %{+a/1}```
* string: ```foo bar baz```
* result: ```a=foobazbar```

In the above example the values are appended together based on the order specified.  

### Named skip key `?`
* pattern:```%{a} %{?skipme} %{c}```
* string: ```foo bar baz```
* result: ```a=foo, c=baz```

In the above example, the parser finds the matches correctly, but excludes the middle key from the results. This is the same behavior as `%{}`, and the name is only used for human readability. 

### Reference keys `*` and `&`
* pattern: ```%{*a} %{b} %{&a}```
* string: ```foo bar baz```
* result: ```foo=baz, b=bar```

In the above example, there is a pair of `a` keys. One has the `*` and the other `&`.  This instructs the parser to use the value of the `*` as the key name for the value of `&` in the result set.  `*` and `&` must come in pairs in the dissect pattern.  

The left / right order of `*` and `&`does not matter.  

* pattern: ```%{&a} %{b} %{*a}```
* string: ```foo bar baz```
* result: ```baz=foo, b=bar```

### Remaining match
* pattern: ```%{a} %{b},%{c}```
* string: ```foo bar,baz  something more here``` 
* result: ```a=foo, b=bar, c=baz something more here```

In the above example the last key matched the remainder of the input string. 

### Consecutive repeating delimiters
* pattern: ```%{a},%{b},%{c},%{d},%{e},%{f},%{g}```
* string: ```foo,,,,,,bar``` 
* result: ```a=foo, b="", c ="", d="", e="", f="", g=bar```

In the above example the `,` repeats many times, leaving 5 empty key/value pairs.

* pattern: ```%{a},%{b},%{c},%{d},%{e},%{f},%{g}```
* string: ```foo,,bar,,,,baz``` 
* result: ```a=foo, b="", c ="bar", d="", e="", f="", g=baz```

In the above example the `,` repeats many times, finds a value, then repeats more. 

* pattern: ```%{a->},%{g}```
* string: ```foo,,,,,,bar``` 
* result: ```a=foo, g=bar```

In the above example the `,` repeats many times, but the right padding modifier `->`instructs the parser to skip over the repeating delimiters. 

### Postfix pattern
* pattern: ```%{timestamp} %{+timestamp} %{+timestamp} %{logsource} %{program}[%{pid}]: %{message}```
* string:  ```Mar 16 00:01:25 example postfix/smtpd[1713]: connect from example.com[192.100.1.3]```
* result: ```timestamp="Mar 16 00:01:25" , logsource="example", program="postix/smtpd" pid="1713" message="connect from example.com[192.100.1.3]"```

