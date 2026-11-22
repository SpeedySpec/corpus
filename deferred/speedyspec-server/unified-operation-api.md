# Unified operation API

`speedyspec-operation-contracts`, `speedyspec-operation`, `speedyspec-operation-dispatch`,
`speedyspec-transport-contracts`, `speedyspec-transport`, `speedyspec-transport-hooks-wp`, `speedyspec-operation-php`,
`speedyspec-rest-contracts`, `speedyspec-rest`, `speedyspec-rest-php`.

The API defines each public client operation once, as a protocol-neutral specification. It binds one handler to each
operation, and lets REST, XML-RPC, Abilities (and later WebSocket and GraphQL) expose it. It was built from four plans:
- #62: the specification core and dispatch;
- #63: the transport contracts;
- #72: the PHP facade (its review fixes landed with #81);
- #81: the REST transport.

These are the Minor findings from each plan's final whole-branch review. The Critical and Important findings were fixed
before merge. None of these breaks ordinary use. They concern edge-case input, WordPress fidelity details, speed, test
coverage and documentation.

## Specification core (#62)

### A nullable enum

- **Where:** `S` builder and `validate`.
- **What happens:** a schema that is both `nullable()` and has `enum_values` refuses `null` unless `null` is one of
  the values.
- **Suggested fix:** treat `nullable` as adding `null` to the allowed values.

### `uniqueItems` compares structurally, in O(n²)

- **Where:** `validate`, the `uniqueItems` rule.
- **What happens:** large arrays are slow to validate, and `1` and `1.0` count as different.
- **Suggested fix:** hash normalized values.

### `S::raw` loosens schemas

- **Where:** `S::raw`, `S::from_value`.
- **What happens:** keywords outside the supported set are refused. A schema with only some supported keywords is
  accepted, without the constraints the caller expected.
- **Suggested fix:** document exactly what is read, or warn on keywords that are read loosely.

### A discriminated `oneOf` branch without an enum

- **Where:** registration checks.
- **What happens:** a branch whose discriminator property has no single-value enum is accepted, so the discriminator
  cannot pick it.
- **Suggested fix:** refuse it at registration.

### Relation arguments can leave required inputs unfilled

- **Where:** `Registries::close`, relation checks.
- **What happens:** a relation that fills only some of the target operation's required inputs passes, then fails at
  call time.
- **Suggested fix:** check required inputs at close.

### Regexes recompiled per validation

- **Where:** `validate`, the `pattern` rule.
- **What happens:** every call compiles each pattern again.
- **Suggested fix:** cache compiled patterns by schema.

### A misleading `Closed` message

- **Where:** `RegistrationError::Closed`.
- **What happens:** it says the definition "differs" when the name was never registered at all.
- **Suggested fix:** separate "unknown after close" from "differs".

### Type documents lack `"type": "object"`

- **Where:** `export::type_document`.
- **What happens:** generators expecting JSON Schema objects have to infer the type.
- **Suggested fix:** emit `"type": "object"`.

### Enum export formats

- **Where:** `export::json_schema`.
- **What happens:** binary and date-time enum values export in their internal form.
- **Suggested fix:** export them in their wire form (base64, ISO 8601).

### Document export lists properties alphabetically

- **Where:** `Registries::document` (`serde_json` without `preserve_order`, workspace-wide).
- **What happens:** documents do not show declaration order, though the registries keep it.
- **Suggested fix:** write documents with an order-keeping writer, as the REST encoder does.

## Transport contracts (#63)

### Valid UTF-8 binary comes back as a string

- **Where:** `speedyspec-transport-hooks-wp`, `from_hook`.
- **What happens:** bytes that happen to be valid UTF-8 cross back as `String`, not `Binary`.
- **Suggested fix:** schema-guided normalization covers inputs; outputs through hooks still need it.

### Key `1` and key `"1"` collapse

- **Where:** `to_hook`/`from_hook`.
- **What happens:** a map with string key `"1"` becomes PHP int key `1`, and comes back as a list when it is the
  only key.
- **Suggested fix:** document it as PHP's array semantics, or guard it with the schema.

### Transport names are unvalidated `&str` in the exposure APIs

- **Where:** `ExposureLookup::find`/`exposed`, `OperationSpec::exposed_on`.
- **What happens:** a typo finds nothing rather than failing.
- **Suggested fix:** take `TransportName`.

### No test for an operation with no default mapping

- **Where:** `speedyspec-transport` tests.
- **What happens:** the "skipped when the transport has no default" path is untested.

### `Bypass` documentation

- **Where:** `speedyspec-operation-dispatch`, `Bypass`.
- **What happens:** the doc does not say that a bypass skips output validation.

### One overlong doc line

- **Where:** `speedyspec-transport/src/lib.rs`.
- **What happens:** a cosmetic issue.

## PHP facade (#72)

### `owner_of` assumes type names have no dots

- **Where:** `Registries::close_lenient`, `owner_of`.
- **What happens:** type names are not validated. A type named `acme.report` maps to owner `acme`, which drops
  nothing or the wrong definition.
- **Suggested fix:** validate type names, or carry the owner on each error variant.

### A date object inside an array input is refused

- **Where:** `Native\call`.
- **What happens:** `Operations::call('x', ['due' => new DateTimeImmutable()])` fails with `php_object_not_allowed`,
  because only object inputs go through `Mapper::extract`.
- **Suggested fix:** run `Mapper::extract` on array input too.

### Objects in exception details replace the error

- **Where:** `invoke.rs`.
- **What happens:** an `OperationException` whose details hold an object becomes `php_object_not_allowed`.
- **Suggested fix:** map details through `Mapper::extract` in the invoker.

### Gaps in the class check

- **Where:** `shape::check_shape`.
- **What happens:**
  - A reference to a type registered later in the same init is skipped, and close never re-checks it.
  - Nested classes are not checked.
  - A nullable or optional output property against a required, non-nullable schema is not reported, so it fails at
    runtime.

### A value object's constructor exception loses its code

- **Where:** `mapper.rs`, hydration.
- **What happens:** an `OperationException` thrown while building the input becomes `hydration_failed`.
- **Suggested fix:** route hydration through the invoker.

### About seven PHP↔Rust crossings per call

- **Where:** `Operations::call` and the dispatcher.
- **What happens:** the input is hydrated twice, the PHP `Caller` is built twice, and the permission callback,
  handler and extraction each cross the boundary. Each request's init re-run also rebuilds specifications and
  compares documents.
- **Suggested fix:** cache the PHP `Caller` per dispatch, then measure.

### A second `OperationArea` takes over the native functions

- **Where:** `OperationArea::new`.
- **What happens:** the global state is overwritten by the last area built.
- **Suggested fix:** assert there is one area, or document it.

### `PhpSiteKey` falls back to blog 1

- **Where:** `sites.rs`.
- **What happens:**
  - On failure it uses `{blog: 1, fingerprint: ""}`, replacing blog 1's site.
  - The site id is cached for the request, so `switch_to_blog` does not move calls.
  - `by_blog` grows without bound on large multisites.

### Numeric property names give an unclear error

- **Where:** `S::object`, `TypeBuilder`.
- **What happens:** PHP turns `'0'` keys into integers, which reads as a list and gives "properties must be an
  object".

### `Caller::can()` uses the area's identity

- **Where:** `Native\can`.
- **What happens:** a transport with its own `Identity` is bypassed in PHP permission callbacks. The README's
  Limits section does not say so.

### The README omits the container dependency

- **Where:** `speedyspec-operation-php` README and `lib.rs`.
- **What happens:** neither says that `OperationArea` needs a `ContainerArea` over the same container. Without one,
  request-lifetime handler bindings are never dropped.

### No end-to-end binary input test

- **Where:** `speedyspec-operation-php` tests.
- **What happens:** binary is pinned only on output and in a unit test, not through `Operations::call`.

## REST transport (#81)

### Links rebuilt per item

- **Where:** `links::for_item`.
- **What happens:** every link walks every operation, clones the route list and runs the named-group regex. The
  `self` link comes from the first `get` operation by name.
- **Suggested fix:** precompute a "type → get operation and route" map when the table is built.

### Links are added before `rest_post_dispatch`

- **Where:** `RestServer::dispatch`.
- **What happens:** trunk adds `_links`/`_embedded` after `rest_post_dispatch` and `rest_pre_serve_request`
  (`response_to_data`). Here plugins may see them inside `data`. List items embed before `after_callbacks`, single
  items after.

### Embedded requests do not set `context=embed`

- **Where:** `RestServer::embedded`.
- **What happens:** this matters once WordPress controllers are bridged (Phase 5).

### Unknown methods fall through

- **Where:** `decode::rest_request`.
- **What happens:** an unknown `?_method=`, and `OPTIONS`, leave REST and reach the next handler. They should answer
  404 `rest_no_route` under `/wp-json`.

### Malformed JSON bodies are ignored

- **Where:** `decode::rest_request`.
- **What happens:** WordPress answers 400 `rest_invalid_json`. Multipart bodies are not decoded either.

### Coercion differs from WordPress

- **Where:** `coerce::coerce`.
- **What happens:** WordPress accepts `"5.0"` as an integer and `TRUE`/`False` in any case. Union and nullable
  schemas are not coerced.

### 204 responses carry a body

- **Where:** `encode::http_response`.
- **What happens:** the body is `null`.

### `fill()` breaks on unusual routes

- **Where:** `links::fill`.
- **What happens:** parentheses inside character classes, escaped parentheses and `(?:…)` groups give broken hrefs.

### Top-level alternation escapes the anchors

- **Where:** `RouteTable::build`, `RestTransport::check_binding`.
- **What happens:** a bound route like `/a|/b` breaks `^…$`, as in WordPress, so one plugin's route can capture
  another's paths.
- **Suggested fix:** wrap patterns as `^(?:…)$`.

### A new request proxy per hook

- **Where:** `PhpRestHooks`.
- **What happens:** WordPress passes one `WP_REST_Request` through all hooks. Plugins keying state on object
  identity (`spl_object_id`, WeakMap, dynamic properties) lose it.
- **Suggested fix:** cache one proxy per `SharedRequest` for the request.

### `function_exists('apply_filters')` on every hook

- **Where:** `PhpRestHooks::apply`.
- **What happens:** the check doubles the PHP crossings.
- **Suggested fix:** check once per request.

### Native functions use `borrow()`/`borrow_mut()`

- **Where:** `speedyspec-rest-php` natives.
- **What happens:** nothing re-enters today. A future re-entry would panic across FFI.
- **Suggested fix:** `try_borrow`, mapped to `LogicException`.

### Returned errors and responses

- **Where:** `Internal\Adapter::describe`, `errors::wp_error`.
- **What happens:**
  - A multi-error `WP_Error` keeps only its first error.
  - A string `status` such as `"418"` falls back to its category's status.

### A PHP handler returning `WP_REST_Response`

- **Where:** `PhpOperationHandler`.
- **What happens:** the response is extracted as `{data, headers, status}` and fails output validation, though the
  Phase 4 decision lists it as an allowed handler return.

### Internal error text reaches clients

- **Where:** `operation_no_handler`.
- **What happens:** the container's error text is sent to the client, unlike WordPress without debug.

### The route table and codec are not ports

- **Where:** `RestServer::new`, `decode`, `encode`.
- **What happens:** replacing them needs a code change. Deferred until a second implementation needs them.
