# Proposal: `code` Property for ECMAScript Error Objects

## Status

Stage 2

## Champions

James M Snell

## Reviewers

Jordan Harband

## The Use Case

A standardized `code` property would enable:

1. **Precise programmatic error handling.** Application code can branch on specific
   error conditions without depending on message strings:

    ```js
    try {
      await db.connect();
    } catch (e) {
      if (e.code === 'ERR_CONNECTION_REFUSED') {
        // Retry with backoff
      }
    }
    ```

2. **Stable error contracts across versions.** An error code is a machine-readable
   identifier that can be documented, versioned, and relied upon — unlike messages,
   which are implementation-defined prose.

3. **Cross-realm error identification.** Unlike `instanceof`, a string `.code`
   property survives structured cloning, serialization, and cross-realm transfer.

4. **Improved error reporting and telemetry.** Aggregating errors by `.code` in
   monitoring systems (Sentry, Datadog, etc.) is more reliable and useful than
   aggregating by message, which varies across engines and versions.

5. **Error translation and i18n.** With a stable code, error messages can be
   localized or mapped to user-facing text without losing machine-readability.

6. **Ecosystem alignment.** Providing a standard property that the ecosystem already
   uses would reduce fragmentation and give library authors a blessed pattern to
   follow.

7. **Better developer experience.** Documentation and tooling can reference error
   codes. A developer can search for `ERR_INVALID_ARG_TYPE` and find definitive
   documentation, whereas searching for a message string is unreliable.

## Out of scope

When presenting this to TC-39 for Stage 1, the question was raised about whether
the committee should carve out a part of the `code` namespace for TC-39 defined
error codes; and whether the committee should be defining it's own `code` values
for spec-defined throws.

This becomes challenging for a few reasons:

1. It's difficult to retrofit. The spec defines a large number of throw conditions.
   It would be a significant lift for implementions to retroactively update error
   handling logic to add and propagate error codes through the existing logic. The
   benefit of doing so is likely marginal enough that it wouldn't be worth it.
2. Many implementation-level optimizations mean it's not often clear exactly which
   code would apply in any given situation. The throw might originate in a utility
   function used by multiple paths, for instance, or a single "observable" throw
   defined by the spec might actually originate from multiple places in the code
   or even shift around over time making it more difficult to manage.
3. What the `code` namespace would be is difficult to determine. While the ecosystem
   has adopted the use of codes they have not settled on a common naming or
   namespacing convention and anything we may try to impose will either conflict with
   or unreasonably constrain the ecosystem.

For the purposes of this proposal. We consider both the definition of spec-defined
`codes` and whether to assign them to spec-defined errors to be out of scope; and
something that is better addressed in a separate follow-on proposal, if at all.

## Prior Art Survey

The JavaScript ecosystem has overwhelmingly and independently adopted `error.code`
as the solution. This section surveys the landscape.

### JavaScript Runtimes

#### Node.js

The most mature implementation. Node.js has used string `.code` properties on errors
since v8.0 (2017), with hundreds of documented codes.

- **Type:** `string` (instance property)
- **Convention:** `ERR_*` prefix for Node.js errors (e.g., `ERR_INVALID_ARG_TYPE`,
  `ERR_BUFFER_OUT_OF_BOUNDS`, `ERR_HTTP2_INVALID_HEADER_VALUE`). POSIX codes for
  system errors (`ENOENT`, `EACCES`, `ECONNREFUSED`).
- **Scope:** 200+ documented `ERR_*` codes organized by subsystem, plus POSIX and
  OpenSSL codes.
- **Architecture:** Internal `NodeErrorAbstraction` classes extend native types
  (`TypeError`, `RangeError`, etc.) and set `.code` in the constructor.
- **Design choice:** Node.js deliberately chose **strings over numbers** so that
  error handling code is self-documenting without looking up magic numbers.
- **Stability commitment:** Error codes are part of the stable API. Removing or
  changing the semantics of a code is a breaking change.

```js
// Node.js error with .code
const fs = require('fs');
try {
  fs.readFileSync('/nonexistent');
} catch (e) {
  e.code;    // 'ENOENT'
  e.message; // "ENOENT: no such file or directory, open '/nonexistent'"
}
```

#### Deno

Deno mirrors Node.js `.code` exactly in its `node:*` compatibility layer:

- **Node compat:** Uses `ERR_*` string codes, identical to Node.js.
- **Native APIs:** Uses a class hierarchy instead (`Deno.errors.NotFound`,
  `Deno.errors.PermissionDenied`), relying on `instanceof` checks. POSIX errno
  strings (e.g., `ENOENT`, `ECONNREFUSED`) are attached as `.code` on I/O errors
  via the underlying OS error, but this is not documented as a public API.

The fact that Deno adopted Node.js error codes wholesale for its compat layer
demonstrates that the pattern is essential for interoperability. Deno's native
choice of `instanceof`-based error classes highlights the limitation of that
approach — it doesn't survive structured cloning or cross-realm transfer, and
requires importing specific error constructors.

```js
// Running in Deno
let e = new Deno.errors.PermissionDenied()
console.log(e instanceof Deno.errors.PermissionDenied) // true
console.log(e.name) // "PermissionDenied"

// The details of the error do not survive structured cloning
let clone = structuredClone(e)
console.log(clone instanceof Deno.errors.PermissionDenied) // false
console.log(clone.name) // "Error"

// Does not survive postMessage either
const { port1, port2 } = new MessageChannel();
port1.postMessage(e);
port2.onmessage = (event) => {
  const received = event.data;
  console.log(received instanceof Deno.errors.PermissionDenied) // false
  console.log(received.name) // "Error"
};
```

Deno also does not implement `DOMException` as structured cloneable, so it loses
all type information, including the modified `.name` property when cloned.

#### Bun

Bun mirrors Node.js `.code` for all `node:*` module errors, and has extended
the same convention to its own native APIs:

- Same `ERR_*` convention and POSIX codes for system errors.
- Bun-native APIs use original `ERR_*` codes: e.g., `ERR_POSTGRES_*` for the
  built-in Postgres client, `ERR_REDIS_*` for Redis, and `ERR_S3_*` for S3.

Bun's decision to adopt `ERR_*` codes for its own non-Node APIs — rather than
inventing a separate error identification scheme — is strong evidence that
`.code` is the natural extension point for JavaScript errors.

#### Cloudflare Workers

Cloudflare Workers also mirror Node.js error codes for `node:*` module errors

- Same `ERR_*` convention for compatibility.
- Native APIs throw standard `Error` without custom codes.

### Web Platform

#### DOMException

DOMException has both a legacy numeric `.code` and a modern string `.name`:

- **`.code`** (`number`): **Deprecated.** Legacy constants like
  `NOT_FOUND_ERR = 8`, `NOT_SUPPORTED_ERR = 9`. Returns `0` for all newer error
  types added after ~2012.
- **`.name`** (`string`): The modern identifier. PascalCase:
  `"NotFoundError"`, `"AbortError"`, `"DataCloneError"`.

**The web platform's explicit deprecation of numeric `.code` in favor of string
`.name` is a clear signal: string-based error identification is the right path.**

#### MediaError

- **Type:** `number`
- **Codes:** `MEDIA_ERR_ABORTED = 1`, `MEDIA_ERR_NETWORK = 2`,
  `MEDIA_ERR_DECODE = 3`, `MEDIA_ERR_SRC_NOT_SUPPORTED = 4`

#### GeolocationPositionError

- **Type:** `number`
- **Codes:** `PERMISSION_DENIED = 1`, `POSITION_UNAVAILABLE = 2`, `TIMEOUT = 3`

#### RTCError

- Inherits DOMException (`.code` + `.name`) but adds a **string**
  `.errorDetail` property (`"sdp-syntax-error"`, `"dtls-failure"`, etc.) for
  domain-specific identification.

#### Trend

Older Web APIs used numeric codes. Newer ones use strings. The platform has moved
toward string-based identification.

### Popular Libraries

The following major libraries independently adopted `error.code`:

| Library              | `.code` type | Convention                                   |
| -------------------- | ------------ | -------------------------------------------- |
| [**axios**][axios-err]            | `string`     | `ERR_NETWORK`, `ERR_CANCELED`, `ETIMEDOUT`   |
| [**Firebase**][firebase-err]         | `string`     | `"auth/user-not-found"`, `"storage/not-found"` |
| [**Stripe**][stripe-err]           | `string`     | `"card_declined"`, `"rate_limit"`            |
| [**Prisma**][prisma-err]           | `string`     | `"P2002"`, `"P2025"`, `"P1001"`             |
| [**pg** (Postgres)][pg-err]    | `string`     | SQLSTATE codes: `"23505"`, `"42P01"`         |
| [**mysql2**][mysql2-err]           | `string`     | `"ER_DUP_ENTRY"`, `"ER_ACCESS_DENIED_ERROR"` |
| [**MongoDB/mongoose**][mongo-err] | `number`     | MongoDB server codes: `11000`                |
| [**@grpc/grpc-js**][grpc-err]    | `number`     | gRPC status codes: `5` (NOT_FOUND)           |
| [**AWS SDK v3**][aws-err]       | `string`     | `"AccessDenied"`, `"NoSuchBucket"`           |
| [**Zod**][zod-err]              | `string`     | `"invalid_type"`, `"too_small"`              |

**String codes dominate.** Numeric codes appear only where an upstream protocol
defines them (gRPC, MongoDB).

### Other Languages

| Language  | Mechanism                                        | Type         |
| --------- | ------------------------------------------------ | ------------ |
| **Python**    | Exception class hierarchy + `OSError.errno`          | class + `int`  |
| **Rust**      | `std::io::ErrorKind` enum + `.raw_os_error()`        | enum + `i32`   |
| **Go**        | Sentinel values (`os.ErrNotExist`) + `errors.Is()`   | value identity |
| **Java**      | Exception hierarchy + `SQLException.getSQLState()`   | class + `String` |
| **C#/.NET**   | Exception hierarchy + `Exception.HResult`            | class + `int`  |

Most languages use type hierarchies as the primary mechanism, with optional
string/numeric codes for interop with external systems (OS, databases, protocols).
JavaScript lacks a practical type hierarchy for this purpose (limited built-in
subtypes, cross-realm issues with `instanceof`), making a property-based approach
more appropriate.

## Why Not Just Use `error.name`?

`error.name` already exists and defaults to the constructor name (`"TypeError"`,
`"RangeError"`, etc.). However:

- `.name` is **coarse-grained** — all TypeErrors share the same `.name`.
- `.code` provides **fine-grained** identification within an error type.
- They are complementary: `.name` says *what kind* of error; `.code` says *which
  specific* error condition.
- Overwriting `.name` to encode specific conditions conflates two concerns and
  breaks `instanceof`-based expectations.

DOMException is a cautionary example of what happens when `.name` is repurposed to
carry specific error identity. Every DOMException instance has its `.name` set to a
specific error string like `"NotFoundError"` or `"AbortError"` rather than
`"DOMException"`. This means:

- **`instanceof` and `.name` disagree.** `err instanceof DOMException` is `true`,
  but `err.name` is `"AbortError"`, not `"DOMException"`. This breaks the
  fundamental expectation that `.name` reflects the constructor/type.
- **`error.name` becomes load-bearing for dispatch.** Code must switch on `.name`
  to distinguish DOMException subtypes, making `.name` a de facto error code while
  still nominally being the "type name." This is exactly what `.code` should be for.
- **It forecloses subclassing.** Because `.name` already carries the specific error
  identity, there is no room for a DOMException subclass to have its own `.name`
  without losing the error identity, and no room for `.name` to reflect the actual
  class hierarchy.
- **Stack traces and logging are misleading.** A logged `DOMException` shows its
  `.name` as `"AbortError"`, which looks like a separate error class that doesn't
  exist. Developers search for an `AbortError` constructor and find nothing (or
  find that `AbortError` is just a `DOMException` with a specific `.name`).

Had `.code` existed as a standard property, DOMException could have used
`{ code: "AbortError" }` while keeping `.name` as `"DOMException"`, preserving the
natural relationship between `.name`, `instanceof`, and the class hierarchy.

## Proposed Design

### API

Extend the `Error` constructor options bag (introduced in ES2022 for `cause`) to
accept a `code` property. This applies to `Error`, all `NativeError` types
(`TypeError`, `RangeError`, etc.), `AggregateError`, and `SuppressedError`:

```js
new Error("something went wrong", { code: "ERR_SOMETHING" })
new TypeError("expected string", { code: "ERR_INVALID_ARG_TYPE", cause: original })
```

The `.code` property would be:

- **Defined on instances**, not on `Error.prototype`
- **Type:** any value (not restricted to strings, consistent with `cause`)
- **Default:** property is not present when not provided (`'code' in err` is
  `false`), consistent with `cause`
- **Enumerable:** `false` (consistent with `cause` and `message`)
- **Writable:** `true` (consistent with other Error properties)
- **Configurable:** `true`

### Why not restrict the type to `string`?

While strings are the dominant convention in the ecosystem (Node.js, axios, Firebase,
Stripe, etc.), the spec should not constrain the type. Several major libraries use
numeric codes (gRPC, MongoDB, TypeScript diagnostics), and `cause` already
established the precedent of accepting any value without type restriction.

The ecosystem strongly favors strings for the reasons outlined in the prior art
survey — self-documenting, no lookup tables, no collision risk — but this is best
left as a convention rather than a language-level constraint.

### Why absent by default, not mandatory

1. Backward compatibility: existing `new Error("msg")` calls should work unchanged.
2. Not all errors have meaningful codes (e.g., ad-hoc `throw new Error("bug")`).
3. Follows the `cause` precedent, which is also absent when not provided.

## Relationship to Existing Proposals

- **`Error.cause`** (ES2022): Established the options bag pattern on the `Error`
  constructor. This proposal extends that same bag with `code`.
- **Error Stacks** (Stage 1): Focuses on standardizing `error.stack`. Orthogonal
  but complementary — stacks could include codes. Related proposals will make use
  of the options bag for other metadata.
- **`Error.isError`** (Stage 2): Cross-realm error identification. Complementary —
  `.code` provides fine-grained identification within a confirmed error.

## FAQ

### Doesn't this encourage stringly-typed programming?

No more than `error.message` and `error.name` already do in practice. The key
difference is that `.code` is explicitly intended as a **machine-readable
identifier**, ideally with stability guarantees, whereas `.message` is
human-readable prose. A `.code` is semantically equivalent to a discriminant in
a tagged union — it just uses a string (typically) instead of a type tag.

### Won't every library invent its own codes, leading to chaos?

They already do — that's the current situation. Standardizing the *property*
doesn't require standardizing the *values*. It provides a blessed location for
codes (instead of ad-hoc properties like `.errno`, `.errorCode`, `.errCode`, etc.)
and enables tooling to be built around a single convention.

### Are we defining a new error taxonomy with this proposal?

No. The proposal does not prescribe any specific codes or taxonomies. It simply
provides a standard property for libraries and applications to use if they choose
to. The ecosystem can evolve organically, and popular codes will emerge as de
facto standards (e.g., `ERR_INVALID_ARG_TYPE` in Node.js).

### Why not encourage use of Symbol-based codes instead of strings?

Symbols would prevent collisions but sacrifice the key advantages of string codes:
serialization, logging, telemetry aggregation, human readability, and cross-realm
transfer. The Node.js ecosystem has demonstrated that string codes with prefix
conventions (`ERR_*`) are practical and collision-resistant.

But since the code value can be any type, libraries and applications can use
symbols if they prefer - the proposal does not preclude that.

### Can't you solve this with error subclasses?

In theory, yes — a class hierarchy can encode any error taxonomy. In practice:

- `instanceof` breaks across realms.
- The built-in error hierarchy is too shallow (only ~7 types).
- Creating deep class hierarchies for every error condition is ergonomically heavy.
- Error subclasses cannot be pattern-matched in `catch` (no `catch (e if ...)` in
  standard JS).
- The ecosystem has already voted with its feet: `.code` on instances.
- Structured cloning and cross-realm transfer of errors is common (e.g., `postMessage`),
  and classes don't survive that.

We see these limitations demonstrated in Deno's namespace of `Deno.errors.*`
classes, which cannot be reliably identified across cloning boundaries, and
variable runtime support of `structuredClone`, etc.

<!-- Reference links for the Popular Libraries table -->
[axios-err]: https://axios-http.com/docs/handling_errors
[firebase-err]: https://firebase.google.com/docs/reference/js/auth#autherrorcodes
[stripe-err]: https://docs.stripe.com/error-codes
[prisma-err]: https://www.prisma.io/docs/orm/reference/error-reference
[pg-err]: https://node-postgres.com/apis/client
[mysql2-err]: https://sidorares.github.io/node-mysql2/docs
[mongo-err]: https://www.mongodb.com/docs/manual/reference/error-codes/
[grpc-err]: https://grpc.github.io/grpc/node/grpc.html
[aws-err]: https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/Package/-smithy-smithy-client/
[zod-err]: https://zod.dev/error-handling
