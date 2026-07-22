---
theme: default
title: "Reconciling Error.code with DOMException"
info: |
  Addressing the DOMException conflict concern for the Error code property proposal.
  Stage 2 - Requirement for further advancement.
class: text-center
---

# Reconciling `Error.code` with `DOMException`

Error Code Property Proposal - Stage 2

James M Snell

---

# The Concern

[Issue #2](https://github.com/tc39/proposal-error-code-property/issues/2) - raised by Anne van Kesteren (WHATWG)

> I don't like this as DOMException has Error in its prototype chain so we'd end up
> shadowing this new property making it useless in a lot of web platform code.

> The web platform has established a pattern whereby if the `name` field is not granular
> enough you create a subclass of `DOMException` for the domain in question with specific
> fields that make sense.

<v-click>

Three claims to address:

1. There would be **shadowing** / a conflict
2. The new property would be **useless in a lot of web platform code**
3. `DOMException` subclasses are an **established pattern** that already solves this

</v-click>

---

# Claim 1: There Would Be Shadowing

`DOMException`'s constructor **never invokes `Error()`**

Per [WebIDL §4.4](https://webidl.spec.whatwg.org/#idl-DOMException) and [§3.14.1](https://webidl.spec.whatwg.org/#js-DOMException-specialness):

- `DOMException` objects are created via `MakeBasicObject()`, not through `Error()`
- The prototype chain to `%Error.prototype%` is set up statically
- `DOMException`'s constructor signature: `(optional DOMString message, optional DOMString name)`
- **No options bag. No `super()` call. No `InstallErrorOwnProperties`.**

The proposed `code` own property can *never* be installed on a `DOMException` instance:

```js
const e = new TypeError('bad', { code: 'ERR_INVALID_ARG_TYPE' });
Object.hasOwn(e, 'code');  // own property, installed by Error()

const de = new DOMException('abort', 'AbortError');
Object.hasOwn(de, 'code');  // prototype accessor
de.code; // 20
```

---

# Claim 1: No Shadowing (cont.)

`DOMException` `code` is a read-only prototype accessor

```
Prototype chain:

  DOMException instance     →  DOMException.prototype  →  Error.prototype
  (no own 'code' property)     (code getter: returns       (no 'code' property
                                legacy numeric constant)    added by this proposal)
```

- The proposal does **not** add `code` to `Error.prototype`
- `code` is only installed as an **own property** when explicitly passed via the options bag
- `DOMException`'s constructor never passes an options bag to `Error()`
- **No own property is created → no shadowing occurs → zero behavioral change**

This is the same as `cause`. `DOMException` doesn't pass `cause` to `Error()` either.

---

# DOMException Already Differs from Error

```js
// Construction differs
const e  = new TypeError('...', { cause: 123 });
const de = new DOMException('...', 'AbortError');
```

```js
// Own properties differ
Object.getOwnPropertyNames(e);   // ['stack', 'message']
Object.getOwnPropertyNames(de);  // ['stack'] message is a prototype property

// .name works differently
e.name;   // 'TypeError' reflects the constructor
de.name;  // 'AbortError' reflects the error condition (not "DOMException")
```

---

# DOMException Already Differs from Error (cont.)

```js
// .name descriptor differs
Object.getOwnPropertyDescriptor(e.__proto__, 'name');
// { value: 'TypeError', writable: true, enumerable: false, configurable: true }

Object.getOwnPropertyDescriptor(de.__proto__, 'name');
// { get: [Function], set: undefined, enumerable: true, configurable: true }

// .message is writable on Error, read-only on DOMException
e.message = 'changed';  console.log(e.message);   // 'changed'
de.message = 'changed'; console.log(de.message);   // '...' (original)
```

---

# Even `cause` Would Differ

The `DOMException` [`cause` PR](https://github.com/whatwg/webidl/pull/1179) (open since 2022) already diverges from `Error`

|                | `Error`                 | `DOMException` (proposed)   |
| -------------- | ----------------------- | --------------------------- |
| **Property type**  | Own data property | Prototype accessor (getter) |
| **When absent**    | Property doesn't exist | Returns `undefined` |
| **`'cause' in err`** | `false` if no cause given | Always `true` |
| **Writable**       | Yes | No (read-only) |

If DOMException can define its own `cause` semantics distinct from `Error`, it can certainly continue defining its own `code` semantics.

---

# Claim 2: "Useless in a Lot of Web Platform Code"

The web platform throws far more `TypeError` and `RangeError` than `DOMException`

**`fetch()`** probably the most-used web API:
- `TypeError`: network errors, invalid URLs, CORS failures, invalid headers, body errors
- `DOMException`: only for `AbortError`, `QuotaExceededError`, ser/deser errors

**Other major APIs:**
- **URL API**: `TypeError` for invalid URLs
- **Streams API**: `TypeError` and `RangeError` throughout
- **Web Crypto**: mix of `TypeError` and `DOMException`
- **Argument validation**: `TypeError` from WebIDL processing across *all* APIs
- **`Response.json()`**: `SyntaxError`

For all of these the proposed `code` property is useful.

It is only inapplicable to `DOMException` instances.

---

# The Legacy `.code` Is Itself Deprecated

The concern is about protecting a property the web platform itself deprecated

- `DOMException`'s numeric `code` returns `0` for all error names added after ~2012
- The WebIDL spec has moved away from DOMException for some error conditions entirely

A decade-old, established-in-pracice `code` property should not be blocked by
a deprecated numeric `code` property on one subclass that already returns `0`
for modern error names.

---

# Claim 3: The "Established Pattern" of Subclasses

The WebIDL spec defines a `DOMException` derived interface pattern with one instance

> "This standard so far defines **one** predefined DOMException derived interface"
>
> - [WebIDL §2.8.3](https://webidl.spec.whatwg.org/#idl-DOMException-derived-predefineds)

That single instance is `QuotaExceededError`, recently upgraded from a plain DOMException name.

Calling it "established" is generous.

---

# Why Renaming Is Worse

Using a name other than `code` would increase fragmentation, not reduce it

The ecosystem has already converged on `.code`:
- Node.js (200+ codes since 2017), Bun, Deno, Cloudflare Workers
- axios, Firebase, Stripe, Prisma, pg, mysql2, AWS SDK, Zod

Renaming creates **three** competing names:

1. **`.code`**; what the entire ecosystem actually uses
2. **`.name`**: what DOMException repurposes for error identification
3. **`.somethingElse`**: a standard property nobody asked for

**Picking a different name defeats the purpose.**

---

# DOMException's `name` Is An Argument *For* `code`

- Every `DOMException` has `.name` set to the error condition (`"AbortError"`), not the type (`"DOMException"`)
- `instanceof DOMException` is `true`, but `.name` is `"AbortError"`
- Stack traces show `"AbortError"`, a class that doesn't exist
- `name` in `DOMException` *is* a `code`
- `DOMException` is the only error type in common use that uses `name` this way

The derived interface pattern is a workaround for the absence of `code`.
It shouldn't be used as an argument against adding it.

---

# Reconciliation Path

Three concrete options for how DOMException handles this

### Option 1: Do nothing (recommended)
There is no conflict.

### Option 2: Add a spec note
A non-normative note in WebIDL clarifying that DOMException has its own `code` semantics and does not participate in the language-level `code` option.

---

# Summary

| Concern                        | Response |
| ------------------------------ | -------- |
| Shadowing / conflict           | 1DOMException1's constructor never invokes `Error()` |
| "Useless in web platform code" | The majority of web platform errors are `TypeError`, `RangeError`, etc. |
| Established subclass pattern   | One instance (`QuotaExceededError`). |
| Should rename the property     | The ecosystem already uses `code` |
| Developers would be confused   | Developers already handle `DOMException` differently from other error types |
| DOMException's legacy `.code`  | Deprecated by the WebIDL spec itself |

**This proposal does not constrain or conflict with `DOMException` in any way.**
`DOMException` can continue doing exactly what it does today, or evolve independently.

---
layout: center
class: text-center
---
