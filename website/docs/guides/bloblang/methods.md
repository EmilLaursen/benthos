---
title: Bloblang Methods
sidebar_label: Methods
description: A list of Bloblang methods
---

Methods provide most of the power in Bloblang as they allow you to augment values and can be chained indefinitely:

```coffee
root.doc.id = this.thing.id.string().catch(uuid_v4())
root.doc.reduced_nums = this.thing.nums.for_each(
  match this {
    this < 10 => deleted()
    _ => this - 10
  }
)
```

## General Purpose

### `apply`

Apply a declared map on a value.

```coffee
map thing {
  root.inner = this.first
}

root.foo = this.doc.apply("thing")

# In:  {"doc":{"first":"hello world"}}
# Out: {"foo":{"inner":"hello world"}}
```

### `bool`

Attempt to parse a value into a boolean. An optional argument can be provided, in which case if the value cannot be parsed the argument will be returned instead.

```coffee
root.foo = this.thing.bool()
root.bar = this.thing.bool(true)
```

### `catch`

If the result of a target query fails (due to incorrect types, failed parsing, etc) the argument is returned instead.

```coffee
root.doc.id = this.thing.id.string().catch(uuid_v4())
```

### `from`

Execute a query from the context of another message in the batch. This allows you to mutate events based on the contents of other messages. For example, the map:

```coffee
root = this
root.foo = json("foo").from(1)
```

Would extract the contents of the JSON field `foo` specifically from message index `1` of a batch, effectively overriding the field `foo` for all messages of a batch to that of message 1.

### `from_all`

Execute a query for all messages of the batch, and return an array of all results.

```coffee
root = this
root.foo_summed = json("foo").from_all().sum()
```

### `or`

If the result of the target query fails or resolves to `null`, returns the argument instead. This is an explicit method alternative to the coalesce pipe operator `|`.

```coffee
root.doc.id = this.thing.id.or(uuid_v4())
```

### `type`

Returns the type of a value as a string, providing one of the following values: `string`, `bytes`, `number`, `bool`, `array`, `object` or `null`.

```coffee
root.bar_type = this.bar.type()
root.foo_type = this.foo.type()

# In:  {"bar":10,"foo":"is a string"}
# Out: {"bar_type":"number","foo_type":"string"}
```

### `number`

Attempt to parse a value into a number. An optional argument can be provided, in which case if the value cannot be parsed into a number the argument will be returned instead.

```coffee
root.foo = this.thing.number() + 10
root.bar = this.thing.number(5) * 10
```

### `string`

Marshal a value into a string. If the value is already a string it is unchanged.

```coffee
root.nested_json = this.string()

# In:  {"foo":"bar"}
# Out: {"nested_json":"{\"foo\":\"bar\"}"}
```

## Object and Array Stuff

### `append`

Returns an array with new elements appended to the end.

```coffee
root.foo = this.foo.append("and", "this")

# In:  {"foo":["bar","baz"]}
# Out: {"foo":["bar","baz","and","this"]}
```

### `collapse`

Collapse an array or object into an object of key/value pairs for each field, where the key is the full path of the structured field in dot path notation.

```coffee
root.result = this.collapse()

# In:  {"foo":[{"bar":"1"},{"bar":"2"}]}
# Out: {"result":{"foo.0.bar":"1","foo.1.bar":"2"}}
```

### `contains`

Checks whether a string contains a substring, an array contains an element matching the argument, or an object contains a value matching the argument, and returns a boolean result.

```coffee
root.has_foo = this.thing.contains("foo")

# In:  {"thing":["this","foo","that"]}
# Out: {"has_foo":true}

# In:  {"thing":["this","bar","that"]}
# Out: {"has_foo":false}

# In:  {"thing":"this foo that"}
# Out: {"has_foo":true}

# In:  {"thing":"this bar that"}
# Out: {"has_foo":false}
```

### `enumerated`

Converts an array into a new array of objects, where each object has a field `index` containing the index of the element and a field `value` containing the original value of the element.

```coffee
root.foo = this.foo.enumerated()

# In:  {"foo":["bar","baz"]}
# Out: {"foo":[{"index":0,"value":"bar"},{"index":1,"value":"baz"}]}
```

### `exists`

Checks that a field, identified via a [dot path][field_paths], exists in an object.

```coffee
root.result = this.foo.exists("bar.baz")

# In:  {"foo":{"bar":{"baz":"yep, I exist"}}}
# Out: {"result":true}

# In:  {"foo":{"bar":{}}}
# Out: {"result":false}

# In:  {"foo":{}}
# Out: {"result":false}
```

### `explode`

Explodes an array or object at a [field path][field_paths].

#### On arrays

Exploding arrays results in an array containing elements matching the original document, where the target field of each element is an element of the exploded array:

```coffee
root = this.explode("value")

# In:  {"id":1,"value":["foo","bar","baz"]}
# Out: [{"id":1,"value":"foo"},{"id":1,"value":"bar"},{"id":1,"value":"baz"}]
```

#### On objects

Exploding objects results in an object where the keys match the target object, and the values match the original document but with the target field replaced by the exploded value:

```coffee
root = this.explode("value")

# In:  {"id":1,"value":{"foo":2,"bar":[3,4],"baz":{"bev":5}}}
# Out: {"foo":{"id":1,"value":2},"bar":{"id":1,"value":[3,4]},"baz":{"id":1,"value":{"bev":5}}
```

### `flatten`

Iterates an array and any element that is itself an array is removed and has its elements inserted directly in the resulting array.

```coffee
root.result = this.flatten()

# In:  ["foo",["bar","baz"],"buz"]
# Out: {"result":["foo","bar","baz","buz"]}
```

### `fold`

Takes two arguments: an initial value, and a mapping query. For each element of an array the mapping context is an object with two fields `tally` and `value`, where `tally` contains the current accumulated value and `value` is the value of the current element. The mapping must return the result of adding the value to the tally.

The first argument is the value that `tally` will have on the first call.

```coffee
root.sum = this.foo.fold(0, this.tally + this.value)

# In:  {"foo":[3,8,11]}
# Out: {"sum":22}
```

```coffee
root.result = this.foo.fold("", "%v%v".format(this.tally, this.value))

# In:  {"foo":["hello ", "world"]}
# Out: {"result":"hello world"}
```

### `get`

Extract a field value, identified via a [dot path][field_paths], from an object.

```coffee
root.result = this.foo.get(this.target)

# In:  {"foo":{"bar":"from bar","baz":"from baz"},"target":"bar"}
# Out: {"result":"from bar"}

# In:  {"foo":{"bar":"from bar","baz":"from baz"},"target":"baz"}
# Out: {"result":"from baz"}
```

### `index`

Extract an element from an array by an index. The index can be negative, and if so the element will be selected from the end counting backwards starting from -1. E.g. an index of -1 returns the last element, an index of -2 returns the element before the last, and so on.

```coffee
root.last_name = this.names.index(-1)

# In:  {"names":["rachel","stevens"]}
# Out: {"last_name":"stevens"}
```

### `keys`

Returns the keys of an object as an array. The order of the resulting array will be random.

```coffee
root.foo_keys = this.foo.keys()

# In:  {"foo":{"bar":1,"baz":2}}
# Out: {"foo_keys":["foo","bar"]}
```

### `length`

Returns the length of a string, array or object.

```coffee
root.foo_len = this.foo.length()

# In:  {"foo":"hello world"}
# Out: {"foo_len":11}
```

### `map_each`

#### On arrays

Apply a mapping to each element of an array and replace the element with the result. Within the argument mapping the context is the value of the element being mapped.

```coffee
root.new_nums = this.nums.map_each(
  match this {
    this < 10 => deleted()
    _ => this - 10
  }
)

# in:  {"nums":[3,11,4,17]}
# out: {"new_nums":[1,7]}
```

#### On objects

Apply a mapping to each value of an object and replace the value with the result. Within the argument mapping the context is an object with a field `key` containing the value key, and a field `value`.

```coffee
root.new_dict = this.dict.map_each(this.value.uppercase())

# in:  {"dict":{"foo":"hello","bar":"world"}}
# out: {"new_dict":{"foo":"HELLO","bar":"WORLD"}}
```

### `merge`

Merge a source object into an existing destination object. When a collision is found within the merged structures (both a source and destination object contain the same non-object keys) the result will be an array containing both values, where values that are already arrays will be expanded into the resulting array.

```coffee
root = this.apply("foo").merge(this.apply("bar"))
```

### `slice`

Extract a slice from a string or array value by specifying two indices, a low and high bound, which selects a half-open range that includes the first element, but excludes the last one. If the second index is omitted then it defaults to the length of the input sequence.

```coffee
root.beginning = this.value.slice(0, 2)
root.end = this.value.slice(4)

# In:  {"value":"foo bar"}
# Out: {"beginning":"fo","end":"bar"}

# In:  {"value":["foo","bar","baz","buz","bev"]}
# Out: {"beginning":["foo","bar"],"end":["bev"]}
```

A negative low index can be used, indicating an offset from the end of the sequence. If the low index is greater than the length of the sequence then an empty result is returned.

A negative high index can also be used, indicating an offset from the end of the sequence. If the high index is greater than the length of the sequence, slice extracts through to the end of the sequence.

```coffee
root.last_chunk = this.value.slice(-4)
root.the_rest = this.value.slice(0, -4)

# In:  {"value":"foo bar"}
# Out: {"last_chunk":" bar","the_rest":"foo"}
```

### `sort`

Attempts to sort the values of an array in increasing order. The type of all values must match in order for the ordering to be accurate. Supports string and number values.

```coffee
root.sorted = this.foo.sort()

# In:  {"foo":["bbb","ccc",aaa"]}
# Out: {"sorted":["aaa","bbb","ccc"]}
```

It's also possible to specify a mapping argument, which is provided an object context with fields `left` and `right`, the mapping must return a boolean indicating whether the `left` value is less than `right`. This allows you to sort arrays containing non-string or non-number values.

```coffee
root.sorted = this.foo.sort(this.left.value < this.right.value)

# In:  {"foo":[{"v":"bbb","id":"foo"},{"v":"ccc","id":"bar"},{"v":"aaa","id":"baz"}]}
# Out: {"sorted":[{"id":"baz","v":"aaa"},{"id":"foo","v":"bbb"},{"id":"bar","v":"ccc"}]}
```

### `sum`

Sum the numerical values of an array.

```coffee
root.sum = this.foo.sum()

# In:  {"foo":[3,8,4]}
# Out: {"sum":15}
```

### `unique`

Attempts to remove duplicate values from an array. The array may contain a combination of different value types, but numbers and strings are checked separately (`"5"` is a different element to `5`).

```coffee
root.uniques = this.foo.unique()

# In:  {"foo":["a","b","a","c"]}
# Out: {"uniques":["a","b","c"]}
```

It's also possible to deduplicate values that aren't strings or numbers by providing a mapping argument that converts an element into a string or number for comparison:

```coffee
root.uniques = this.foo.unique(val)

# In:  {"foo":[{"val":"a"},{"val":"b"},{"val":"a"},{"val":"c"}]}
# Out: {"uniques":[{"val":"a"},{"val":"b"},{"val":"c"}]}
```

### `values`

Returns the values of an object as an array. The order of the resulting array will be random.

```coffee
root.foo_vals = this.foo.values()

# In:  {"foo":{"bar":1,"baz":2}}
# Out: {"foo_vals":[1,2]}
```

### `without`

Returns an object where one or more [field path][field_paths] arguments are removed. Each path specifies a specific field to be deleted from the input object, allowing for nested fields.

If a key within a nested path does not exist or is not an object then it is not removed.

```coffee
root = this.without("inner.a","inner.c","d")

# In:  {"inner":{"a":"first","b":"second","c":"third"},"d":"fourth","e":"fifth"}
# Out: {"inner":{"b":"second"},"e":"fifth"}
```

## String Stuff

### `capitalize`

Takes a string value and returns a copy with all Unicode letters that begin words mapped to their Unicode title case.

```coffee
root.title = this.title.capitalize()

# In:  {"title":"the foo bar"}
# Out: {"title":"The Foo Bar"}
```

### `encode`

Encodes a string or byte array target according to a chosen scheme and returns a string result. Available schemes are: `base64`, `base64url`, `hex`, `ascii85`, `z85`.

```coffee
root.encoded = this.value.encode("hex")

# In:  {"value":"hello world"}
# Out: {"encoded":"68656c6c6f20776f726c64"}
```

### `decode`

Decodes an encoded string target according to a chosen scheme and returns the result as a byte array. When mapping the result to a JSON field the value should be cast to a string using the method [`string`][methods.string], or encoded using the method [`encode`][methods.encode], otherwise it will be base64 encoded by default.

Available schemes are: `base64`, `base64url`, `hex`, `ascii85`, `z85`.

```coffee
root.decoded = this.value.decode("hex").string()

# In:  {"value":"68656c6c6f20776f726c64"}
# Out: {"decoded":"hello world"}
```

### `encrypt_aes`

Encrypts a string or byte array target according to a chosen AES encryption method and returns a string result. The algorithms require a key and an initialization vector / nonce. Available schemes are: `ctr`, `ofb`, `cbc`.

```coffee
root.encrypted = this.value.encrypt_aes("ctr", "key", "vector").encode("hex")

# In:  {"value":"hello world!"}
# Out: {"encrypted":"84e9b31ff7400bdf80be7254"}
```

### `decrypt_aes`

Decrypts an encrypted string or byte array target according to a chosen AES encryption method and returns the result as a byte array. The algorithms require a key and an initialization vector / nonce. Available schemes are: `ctr`, `ofb`, `cbc`.

```coffee
root.decrypted = this.value.decode("hex").decrypt_aes("ctr", "key", "vector").string()

# In:  {"value":"84e9b31ff7400bdf80be7254"}
# Out: {"decrypted":"hello world!"}
```

### `escape_html`

Escapes a string so that special characters like `<` to become `&lt;`. It escapes only five such characters: `<`, `>`, `&`, `'` and `"` so that it can be safely placed within an HTML entity.

```coffee
root.escaped = this.value.escape_html()

# In:  {"value":"foo & bar"}
# Out: {"escaped":"foo &amp; bar"}
```

### `unescape_html`

Unescapes a string so that entities like `&lt;` become `<`. It unescapes a larger range of entities than `escape_html` escapes. For example, `&aacute;` unescapes to `á`, as does `&#225;` and `&xE1;`.

```coffee
root.unescaped = this.value.unescape_html()

# In:  {"value":"foo &amp; bar"}
# Out: {"escaped":"foo & bar"}
```

### `escape_url_query`

Escapes a string so that it can be safely placed within a URL query.

```coffee
root.escaped = this.value.escape_url_query()

# In:  {"value":"foo & bar"}
# Out: {"escaped":"foo+%26+bar"}
```

### `unescape_url_query`

Expands escape sequences from a URL query string.

```coffee
root.unescaped = this.value.unescape_url_query()

# In:  {"value":"foo+%26+bar"}
# Out: {"unescaped":"foo & bar"}
```

### `format`

Use a value string as a format specifier in order to produce a new string, using any number of provided arguments.

```coffee
root.foo = "%s(%v): %v".format(this.name, this.age, this.fingers)

# In:  {"name":"lance","age":37,"fingers":13}
# Out: {"foo":"lance(37): 13"}
```

```coffee
root.foo = this.template.format(this.arg1, this.arg2)

# In:  {"template":"%s -> %s","arg1":"foo","arg2":"bar"}
# Out: {"foo":"foo -> bar"}
```

### `has_prefix`

Checks whether a string has a prefix argument and returns a bool.

```coffee
root.t1 = this.v1.has_prefix("foo")
root.t2 = this.v2.has_prefix("foo")

# In:  {"v1":"foobar","v2":"barfoo"}
# Out: {"t1":true,"t2":false}
```

### `has_suffix`

Checks whether a string has a suffix argument and returns a bool.

```coffee
root.t1 = this.v1.has_suffix("foo")
root.t2 = this.v2.has_suffix("foo")

# In:  {"v1":"foobar","v2":"barfoo"}
# Out: {"t1":false,"t2":true}
```

### `hash`

Hashes a string or byte array according to a chosen algorithm and returns the result as a byte array. When mapping the result to a JSON field the value should be cast to a string using the method [`string`][methods.string], or encoded using the method [`encode`][methods.encode], otherwise it will be base64 encoded by default.

Available algorithms are: `hmac_sha1`, `hmac_sha256`, `hmac_sha512`, `sha1`, `sha256`, `sha512`, `xxhash64`.

The following algorithms require a key, which is specified as a second argument: `hmac_sha1`, `hmac_sha256`, `hmac_sha512`.

```coffee
root.h1 = this.value.hash("sha1").encode("hex")
root.h2 = this.value.hash("hmac_sha1","static-key").encode("hex")

# In:  {"value":"hello world"}
# Out: {"h1":"2aae6c35c94fcfb415dbe95f408b9ce91ee846ed","h2":"d87e5f068fa08fe90bb95bc7c8344cb809179d76"}
```

### `join`

Join an array of strings with an optional delimiter into a single string.

```coffee
root.joined_words = this.words.join()
root.joined_numbers = this.numbers.map_each(this.string()).join(",")

# In:  {"words":["hello","world"],"numbers":[3,8,11]}
# Out: {"joined_words":"helloworld","joined_numbers":"3,8,11"}
```

### `lowercase`

Convert a string value into lowercase.

```coffee
root.foo = this.foo.lowercase()
```

### `parse_csv`

Attempts to parse a string into an array of objects by following the CSV format
described in RFC 4180. The first line is assumed to be a header row, which
determines the keys of values in each object:

```coffee
root.orders = this.orders.parse_csv()

# In:  {"orders":"foo,bar\nfoo 1,bar 1\nfoo 2,bar 2"}
# Out: {"orders":[{"foo":"foo 1","bar":"bar 1"},{"foo":"foo 2","bar":"bar 2"}]}
```

### `parse_json`

Attempts to parse a string as a JSON document and returns the result.

```coffee
root.doc = this.doc.parse_json()

# In:  {"doc":"{\"foo\":\"bar\"}"}
# Out: {"doc":{"foo":"bar"}}
```

### `quote`

Quotes a target string using escape sequences (`\t`, `\n`, `\xFF`, `\u0100`) for control characters and non-printable characters.

```coffee
root.quoted = this.thing.quote()
```

### `unquote`

Unquotes a target string, expanding any escape sequences (`\t`, `\n`, `\xFF`, `\u0100`) for control characters and non-printable characters.

```coffee
root.unquoted = this.target.unquote()
```

### `replace`

Replaces all occurrences of the first argument in a target string with the second argument.

```coffee
root.new_value = this.value.replace("foo","dog")

# In:  {"value":"The foo ate my homework"}
# Out: {"new_value":"The dog ate my homework"}
```

### `re_find_all`

Returns an array containing all successive matches of a regular expression in a string.

```coffee
root.matches = this.value.re_find_all("a.")

# In:  {"value":"paranormal"}
# Out: {"matches":["ar","an","al"]}
```

### `re_find_all_submatch`

Returns an array of arrays containing all successive matches of the regular expression in a string and the matches, if any, of its subexpressions.

```coffee
root.matches = this.value.re_find_all_submatch("a(x*)b")

# In:  {"value":"-axxb-ab-"}
# Out: {"matches":[["axxb","xx"],["ab",""]]}
```

### `re_match`

Checks whether a regular expression matches against any part of a string and returns a boolean.

```coffee
root.matches = this.value.re_match("[0-9]")

# In:  {"value":"there are 10 puppies"}
# Out: {"matches":true}

# In:  {"value":"there are ten puppies"}
# Out: {"matches":false}
```

### `re_replace`

Replaces all occurrences of the argument regular expression in a string with a value. Inside the value $ signs are interpreted as submatch expansions, e.g. `$1` represents the text of the first submatch.

```coffee
root.new_value = this.value.re_replace("ADD ([0-9]+)","+($1)")

# In:  {"value":"foo ADD 70"}
# Out: {"new_value":"foo +(70)"}
```

### `split`

Split a string value into an array of strings by splitting it on a string separator.

```coffee
root.new_value = this.value.split(",")

# In:  {"value":"foo,bar,baz"}
# Out: {"new_value":["foo","bar","baz"]}
```

### `strip_html`

Attempts to remove all HTML tags from a target string.

```coffee
root.stripped = this.value.strip_html()

# In:  {"value":"<p>the plain <strong>old text</strong></p>"}
# Out: {"stripped":"the plain old text"}
```

### `trim`

Remove all leading and trailing characters from a string that are contained within an argument cutset. If no arguments are provided then whitespace is removed.

```coffee
root.title = this.title.trim("!?")
root.description = this.description.trim()

# In:  {"title":"!!!watch out!?","description":"  something happened and its amazing! "}
# Out: {"title":"watch out","description":"something happened and its amazing!"}
```

### `uppercase`

Convert a string value into uppercase.

```coffee
root.foo = this.foo.uppercase()
```

[field_paths]: /docs/configuration/field_paths
[methods.encode]: #encode
[methods.string]: #string