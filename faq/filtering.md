---
description: How to filter messages on the Topics → Messages page, including smart (CEL) filters
---

# Message Filtering

When you browse a topic on the **Topics → \{topic} → Messages** page, Kafbat UI gives you two ways to narrow down the messages you see:

* **Search** — a quick, case-sensitive substring match.
* **Smart filters** — expressions written in [CEL (Common Expression Language)](https://github.com/cel-expr/cel-spec/blob/master/doc/langdef.md) that give you full control over which messages match.

Both can be combined with the mode (Newest / Oldest / Live / …), partition and time-range selectors, and a message must satisfy **all** of the active filters to be shown.

## Search

The **Search** box performs a case-sensitive **substring** match. A message is kept when the entered text appears in any of:

* the message key,
* the message value,
* any header key or header value.

**Non-ASCII text.** If your search text contains non-ASCII characters (CJK, emoji, accented letters, …) or characters that require JSON escaping (`"`, `\`, or control characters), Search also matches when the raw key/value/header contains the equivalent `\uXXXX` JSON-escape sequence instead of the literal character — so a message can match even when your exact search text isn't visibly present in it.

This is handy for a quick "does this text appear anywhere in the message" lookup. For anything more precise — comparing a specific JSON field, matching a regular expression, filtering by partition/offset/header — use a smart filter.

## Smart filters

Smart filters are [CEL](https://github.com/cel-expr/cel-spec/blob/master/doc/langdef.md) expressions. Each message is evaluated against your expression, which **must return a boolean**: `true` keeps the message, `false` hides it. An expression that fails to **compile** (bad syntax, or a type problem the compiler can prove up front) is rejected immediately, with an error, when you try to save it. An expression that compiles but throws, or returns something other than a boolean, while evaluating one *specific* message behaves quite differently — see [Runtime errors during consumption](#runtime-errors-during-consumption) below.

To add one, open the **Messages** tab of a topic, click **Add Filters**, and write your expression in the **Filter code** editor. You can give it a display name and save it for reuse. The **?** icon next to the editor shows a short in-app reference.

### Context variables

These 8 fields — and no others — are exposed under a single `record` variable:

| Variable              | Type                  | Description                                                                 |
| --------------------- | --------------------- | --------------------------------------------------------------------------- |
| `record.key`          | dynamic               | Message key, parsed as a JSON object when possible (see below).             |
| `record.keyAsText`    | string                | Raw message key as a string.                                                |
| `record.value`        | dynamic               | Message value, parsed as a JSON object when possible (see below).           |
| `record.valueAsText`  | string                | Raw message value as a string.                                              |
| `record.headers`      | map\<string, string\> | Message headers. An empty map when the message has no headers. If several headers share a key, only the last one is kept (see below). |
| `record.partition`    | int                   | Partition the message was read from.                                        |
| `record.offset`       | int                   | Message offset within the partition.                                        |
| `record.timestampMs`  | int                   | Message timestamp, in epoch milliseconds.                                   |

**Not available in filters.** The message details panel shows several fields that are *not* exposed to `record` and can't be filtered on: key/value/header byte sizes, the timestamp type (`CREATE_TIME` / `LOG_APPEND_TIME` / `NO_TIMESTAMP_TYPE`), which serde/deserializer produced the shown key or value, and schema-registry metadata (e.g. subject/schema ID). There's also no per-message deserialization-error field — if a key or value fails to deserialize with the configured serde, Kafbat UI silently falls back to another serde rather than exposing anything a filter could test.

**Duplicate header keys.** Kafka allows a record to carry more than one header with the same key. Kafbat UI collapses headers into a single map before any filtering runs, so only the *last* header for a repeated key is visible as `record.headers` — earlier values with that key are discarded and can't be recovered, inspected, or counted from a filter (this applies to **Search** too, since it reads the same collapsed map).

**Absent fields.** When a message has no key, both `record.key` and `record.keyAsText` are **not set** (the same is true of `record.value` / `record.valueAsText` for a message with no value). Accessing a field that is not set raises an error, so guard it with the `has()` macro:

```
// keep only messages that have a key
has(record.key)

// keep only messages without a value
!has(record.value)
```

This isn't limited to the key and value fields — indexing **any** map with brackets (for example `record.headers['someKey']`) raises the same kind of error when the key is absent, rather than returning `false` or an empty value. Guard it with `has()` first, e.g. `has(record.headers.k2) && record.headers['k2'] == 'v2'`.

### JSON parsing

`record.key` and `record.value` are bound as **JSON objects only when the raw text is a JSON object**. In that case you can navigate into the fields directly:

```
// value is: { "name": { "first": "user1" } }
has(record.value.name) && has(record.value.name.first) && record.value.name.first == 'user1'
```

**`has()` only guards the last step of a path.** `has(record.value.name.first)` alone only checks whether `first` exists on `record.value.name` — it does not check whether `name` itself exists. If `name` can be missing too (for example, on a topic whose messages don't all share the same JSON shape), chain a `has()` check at every level, innermost last, as shown above.

If the text is **not** a JSON object (a plain string, a number, an array, or invalid JSON), `record.key` / `record.value` fall back to the **raw string**, identical to `record.keyAsText` / `record.valueAsText`:

```
// value is: not json
record.value == 'not json'
```

JSON `null`s nested inside a JSON **object** are preserved, so you can compare a field against `null`:

```
// value is: { "field": { "inner": null } }
record.value.field.inner == null
```

**Nulls nested inside a JSON array are not supported.** Comparing an array element to `null` (or iterating over such an array) raises an evaluation error instead of returning `false`:

```
// value is: { "arr": [1, null, 3] } — raises an error, does NOT evaluate to false
record.value.arr[1] == null
```

### Available functions

Smart filters have access to:

* All [CEL standard macros, operators, and string functions](https://github.com/cel-expr/cel-spec/blob/master/doc/langdef.md) — `has()`, `size()`, `exists()`, `all()`, `==`, `!=`, `<`, `in`, `&&`, `||`, `!`, `startsWith`, `endsWith`, `contains`, `matches`, and so on.
* The CEL **strings** extension — `split`, `lowerAscii`, `upperAscii`, `substring`, `charAt`, `indexOf`, `lastIndexOf`, `replace`, `trim`, `join`.
* The CEL **encoders** extension — `base64.decode` / `base64.encode` (standard base64, **not** URL-safe base64url — see the note under [Examples](#examples)).

**`matches` uses RE2 syntax, and matches anywhere in the string.** Patterns are compiled with [RE2](https://github.com/google/re2/wiki/Syntax), not Java's regex engine: no backreferences (`\1`), no lookahead/lookbehind (`(?=...)`), and named groups are Python-style (`(?P<name>...)`, not `(?<name>...)`). An invalid pattern is **not** rejected when you save the filter — compiling the CEL expression doesn't validate the regex — it only errors once the filter runs against an actual message, and that failure is silent (see [Runtime errors during consumption](#runtime-errors-during-consumption)). `matches()` also succeeds on a **substring**, not a full-string match, by default: `record.valueAsText.matches('info')` also matches a value of `information`. Anchor with `^...$` for a whole-string test.

### Examples

| Expression | Matches |
| ---------- | ------- |
| `record.partition == 1` | Messages on partition 1. |
| `record.offset == 100` | The message at offset 100. |
| `has(record.valueAsText) && record.valueAsText == 'some text'` | Value is exactly `some text`. |
| `has(record.keyAsText) && record.keyAsText.matches('.*[Ee]rror.*')` | Key contains `error` or `Error` anywhere (regex). |
| `has(record.valueAsText) && record.valueAsText.matches('^ERROR$')` | Value is exactly `ERROR` (anchored — without `^...$`, `matches` also matches e.g. `ERROR_CODE`). |
| `has(record.value.name) && has(record.value.name.first) && record.value.name.first == 'user1'` | JSON value whose `name.first` field equals `user1` (safe even when `name` itself is missing). |
| `record.headers.size() == 1 && has(record.headers.k2) && record.headers['k2'] == 'v2'` | Exactly one header, named `k2` with value `v2`. |
| `record.headers.size() == 0` | Messages with no headers. |
| `record.value.field.inner == null` | JSON value with a `field.inner` that is `null`. |
| `has(record.valueAsText) && record.valueAsText.split('.').size() > 1 && string(base64.decode(record.valueAsText.split('.')[1])).contains('user1')` | Decodes the payload segment of a JWT-style value and checks its contents; guarded so it evaluates to `false` (not an error) for messages with no value or a value that isn't JWT-shaped. |

> **The base64 / JWT example above uses standard base64, not base64url.** `base64.decode` decodes with the standard base64 alphabet (`+` / `/`), whereas JWTs are encoded with URL-safe **base64url** (`-` / `_`, and usually unpadded). A real JWT segment that contains `-` or `_` therefore fails to decode — and because that failure is a runtime throw, which the `has(...)` / `size()` guards do **not** catch, the message is silently skipped (see [Runtime errors during consumption](#runtime-errors-during-consumption)). Treat that row as an illustration of chaining `split` → `base64.decode` → `string` → `contains`, not a drop-in filter for real JWTs.

### Runtime errors during consumption

Compilation errors (invalid syntax, or a type problem the compiler can prove for every message — e.g. misusing a statically-typed field like `record.partition` or `record.headers`) are rejected immediately, with an error, when you save the filter.

`record.key` and `record.value` are dynamically typed, so the compiler can't always prove your expression is safe for every message. If evaluating a *specific* message throws — an unguarded absent field, an unguarded map or array index, a JWT-style split that finds no `.`, `base64.decode` on non-base64 input, and so on — or (only possible via `record.key`/`record.value`-based expressions) evaluates to something other than a boolean, that **one message is silently skipped**: it isn't shown, but browsing or tailing continues uninterrupted, with no restart needed. There's no toast, banner, or per-message detail — the only sign is a small error counter that appears in the metrics row above the message list once it's greater than zero, and it doesn't say which message, offset, or field caused it.

If you see that counter: add `has(...)` guards for every field, map key, and array index that isn't guaranteed on every message, and don't assume `record.value` / `record.key` has one consistent shape across an entire topic.

### Troubleshooting

* **"undeclared reference" error when saving.** Remember every field lives under `record.` — bare `partition == 1` doesn't compile (`undeclared reference to 'partition'`), use `record.partition == 1`. This is caught immediately when you try to save the filter — it isn't something that would silently show up as 0 or all results later.
* **"boolean should be returned instead".** Your expression evaluated to a non-boolean value. For statically-typed fields (`partition`, `offset`, `timestampMs`, `keyAsText`, `valueAsText`, `headers`), a mistake like this — e.g. bare `record.partition` instead of `record.partition == 1` — is caught immediately when you save the filter. For expressions built on `record.key` / `record.value` (dynamically typed), the same mistake can instead only surface at evaluation time, per message, with no visible error beyond the errors counter described in [Runtime errors during consumption](#runtime-errors-during-consumption).
* **Errors on missing fields.** Reading `record.key`, `record.keyAsText`, `record.value`, `record.valueAsText`, a nested JSON field, or a map key (e.g. `record.headers['x']`) that isn't present raises an error. Wrap the access in `has(...)`, e.g. `has(record.value.name) && record.value.name == 'x'` or `has(record.headers.x) && record.headers['x'] == 'y'`.
* **Filter throws instead of returning `false`.** It isn't only absent fields — indexing past the end of a list or array (e.g. `.split('.')[1]` on a value with no `.`), or calling a function with input it can't handle (e.g. `base64.decode` on a non-base64 string), also raises an error rather than evaluating to `false`. Guard these the same way you guard absent fields — check `has(...)` / `size()` before indexing.
* **A filter touching a JSON array element errors, or matches fewer messages than expected.** Smart filters can only compare `null` for fields nested inside JSON *objects* — a `null` nested inside a JSON *array* raises an error instead. Avoid `== null` (or `exists()` / `all()`) on array elements that might be `null`.
* **Filter seems to hide messages it shouldn't.** Check the small error counter in the metrics row above the message list. A non-zero count means some messages threw while being evaluated (most often an unguarded field, map key, or array index) and were silently excluded — they weren't judged not to match, the filter itself failed on them.
* **Comparing against a JSON field but getting no matches.** Confirm the value is actually a JSON object — if it's a plain string, use `record.valueAsText` instead of navigating into `record.value`.
