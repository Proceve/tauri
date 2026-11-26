# IPC Origin-Header Fallback — Fix Design (container task PRCVTRFRKP-1)

**Role of this doc:** research-only output of container task PRCVTRFRKP-1. It does **not**
change any code. It hands a concrete fix design to the children:

- PRCVTRFRKP-1.1 — `crates/tauri/src/ipc/protocol.rs`: implement the fallback + unit tests.
- PRCVTRFRKP-1.2 — `PROCEVE-FORK.md`: permanent-fork maintenance policy.

**Input:** `/home/paul/workspace/pBlaiseWorkbench/docs/poc-results/item-a-ipc-mitigation-plan.md`
§1-§2, cross-checked directly against this repo's own source (this is a fork of `tauri`, so the
fix lands here in `crates/tauri/src/ipc/protocol.rs`, not in a vendored/patched dependency the way
the referenced plan describes for a downstream consumer of the crate).

## 1. The bug, confirmed in this repo's source

`crates/tauri/src/ipc/protocol.rs:484-490`, inside `parse_invoke_request`:

```rust
let url = Url::parse(
  parts.headers.get("Origin").ok_or("missing Origin header")?.to_str()
    .map_err(|_| "Origin header value must be a string")?,
).map_err(|_| "Origin header is not a valid URL")?;
```

No fallback for a missing, empty, or opaque (`"null"`) `Origin` — any of those fail and produce
the `"Origin header is not a valid URL"` / `"missing Origin header"` error. This matches the
referenced plan's description of the WebKitGTK behavior of sending a blank/opaque `Origin` for
custom-URI-scheme requests (`tauri-apps/wry#366`), now tripping the IPC call itself on
webkit2gtk 2.52.6.

## 2. (a) Is the handler already bound to a specific webview? Yes — but not where the parse happens

The referenced plan asserts "the protocol handler is already bound to a specific webview/window
at dispatch time, so the correct origin is available without trusting the header." This is **true
at the call site**, but **not yet true inside the function that does the parsing** — the two are
different places in this file:

- `get<R: Runtime>()` (`protocol.rs:38`) is the actual `UriSchemeProtocolHandler` closure. It
  receives `label: &str` and, for `Method::POST`, does
  `if let Some(webview) = manager.get_webview(label) { match parse_invoke_request(&manager, request) ... webview.on_message(request, ...) }`
  (`protocol.rs:~59-61`). So `webview: Webview<R>` **is** resolved and in scope before the Origin
  parse happens.
- `parse_invoke_request<R: Runtime>(manager: &AppManager<R>, request: http::Request<Vec<u8>>)`
  (`protocol.rs:434`) is where the actual `Url::parse(Origin)` call lives. Its signature only
  takes `manager` and `request` — **the resolved `webview` is not passed in**, even though the
  caller already has it.

**Conclusion: the plan's claim is correct in spirit, but the fix requires threading the
already-resolved `webview` reference into `parse_invoke_request` — it isn't available there
today.**

### How the correct origin is derivable (no header trust needed)

`Webview<R>::url()` (`crates/tauri/src/webview/mod.rs:1869`):

```rust
/// Returns the current url of the webview.
pub fn url(&self) -> crate::Result<Url> {
  self.webview.dispatcher.url().map(|url| url.parse().map_err(crate::Error::InvalidUrl))?
}
```

This asks the platform webview dispatcher (the actual browser engine) for its current navigated
URL. It is **not** derived from any header or page-controlled value, so it can't be spoofed by
page JS — it is strictly more trustworthy than the `Origin` header it would be replacing.

**Rejected alternative:** a config-derived fixed origin, e.g.
`crate::protocol::origin(protocol, use_https_scheme)` (used by `is_local_url` and
`convert_file_src`, `webview/mod.rs:1956`) which just formats `http(s)://tauri.localhost` from
`tauri.conf.json`'s `app.windows[].useHttpsScheme` (schema default `false`,
`tauri-utils/src/config.rs:2213`). **Do not use this as the fallback.** Reasoning below.

### Why this matters for security, not just error-avoidance

`InvokeRequest.url` (`crates/tauri/src/webview/mod.rs:~143`, doc comment: *"URL of the frame that
requested this command"*) is not cosmetic. It is consumed at
`crates/tauri/src/webview/mod.rs:2007`:

```rust
pub fn on_message(self, request: InvokeRequest, responder: ...) {
  let is_local = self.is_local_url(&request.url);
  ...
```

`is_local` gates `Origin::Local` vs `Origin::Remote { url }` for the entire ACL/capability
resolution path in `crates/tauri/src/ipc/authority.rs` (`Origin` enum, `resolve_access`,
`is_denied`, etc.) — i.e. **this value decides which commands a given IPC call is allowed to
invoke.** A webview that has actually navigated to remote content must still resolve to
`Origin::Remote`.

This is exactly why the config-derived `tauri.localhost` fallback is wrong: it would
unconditionally report "local" whenever the header happens to be empty/missing, regardless of
what the webview is actually displaying — silently upgrading a remote page's effective
capabilities to local ones. `webview.url()` instead reports the webview's real, engine-verified
current location, so remote navigations are still correctly classified as remote. It reproduces
what the `Origin` header *should* contain, rather than assuming the best case.

## 3. (b) Equivalent/consistency check — the isolation pattern and other platforms already do this

Searched the whole crate for `Origin` header reads: `grep -rn '"Origin"' crates/tauri/src/` has
**exactly one hit** — the line in §1. There is no second, already-fixed copy of this check
elsewhere to copy from. But there *is* a precedent for the right design:

`handle_ipc_message<R: Runtime>(request, manager, label)` (`protocol.rs:185`) is the
**postMessage-based IPC path** (registered as `ipc_handler` from
`crates/tauri/src/manager/webview.rs:555`; used where the custom-protocol `get()` handler isn't
used — the code comments call out Android/mobile explicitly, and it's also the transport the
isolation pattern's decrypt step runs inside, `protocol.rs:203-218`). It:

1. Resolves `webview` directly at the top: `let Some(webview) = manager.get_webview(label) else { return; };` (`protocol.rs:186-188`).
2. **Never reads an `Origin` header at all.** It builds `InvokeRequest.url` as:
   `url: Url::parse(&request.uri().to_string()).expect("invalid IPC request URL")` (`protocol.rs:~303`).

So the existing, working design in this same file already derives `url` from context that's
intrinsically bound to the specific webview/request, never from a trust-the-header string — this
is the pattern `parse_invoke_request` should be brought in line with. The isolation feature's
`#[cfg(feature = "isolation")]` decrypt block (`protocol.rs:458-472`, inside
`parse_invoke_request`) runs *before* the Origin parse and never touches `url`/`Origin`, so the
fix is orthogonal to isolation — no isolation-specific branching is needed in the fix itself, only
in the test matrix (see §5).

## 4. Fix design for PRCVTRFRKP-1.1

### 4.1 Signature change

Change `parse_invoke_request` to accept the fallback origin value from its one caller, rather than
re-resolving the webview itself:

```rust
fn parse_invoke_request<R: Runtime>(
  #[allow(unused_variables)] manager: &AppManager<R>,
  fallback_origin: &Url,
  request: http::Request<Vec<u8>>,
) -> std::result::Result<InvokeRequest, String>
```

`get()` already has `webview` in scope before calling this function
(`if let Some(webview) = manager.get_webview(label) { ... }`); it should pass `&webview.url()...`
in. **Open decision for the implementer:** compute `webview.url()` unconditionally per request (one
extra dispatcher round-trip even when `Origin` is present and valid), or only lazily when the
header turns out to be missing/empty (requires passing `&Webview<R>` itself, not a precomputed
`Url`, pushing the `.url()` call inside `parse_invoke_request`). Passing a precomputed `&Url` is
recommended: it keeps `parse_invoke_request` trivially testable (callers can hand it any `Url`
without constructing a real `Webview<R>`/runtime window), at the cost of one extra `.url()` call
per IPC message — `webview.url()` is a local dispatcher query, not I/O, so this is cheap. Flag this
trade-off explicitly in the PR description if a different choice is made.

### 4.2 Parse logic

Replace the strict parse with:

```rust
let url = match parts.headers.get("Origin") {
  Some(value) => {
    let origin = value
      .to_str()
      .map_err(|_| "Origin header value must be a string")?;
    if origin.is_empty() || origin == "null" {
      fallback_origin.clone()
    } else {
      Url::parse(origin).map_err(|_| "Origin header is not a valid URL")?
    }
  }
  None => fallback_origin.clone(),
};
```

Key points:
- Only fall back for the specific known-bad shapes: header absent, empty string, or the opaque
  `"null"` literal (WebKitGTK's documented behavior for custom-URI-scheme requests). A header that
  is *present but garbage* (e.g. `"Origin: not a url"`) is a different, unexplained failure mode
  and should still hard-error rather than silently falling back — don't widen the fallback beyond
  what's actually observed.
- `fallback_origin` itself came from `webview.url()`, which returns `crate::Result<Url>` and can
  fail. `get()` must handle that `Result` (propagate as the same kind of `String` error this
  function already returns, not `.unwrap()`/`.expect()`).

### 4.3 No isolation-specific change needed

Per §3, the isolation decrypt block in this function doesn't touch `url`/`Origin`, so no branching
is needed there. Do confirm this during implementation by re-reading `protocol.rs:446-472` against
the final diff, in case upstream drift has changed that since this doc was written.

## 5. (c) Existing test layout — and the one gap to fix it exposes

`#[cfg(test)] mod tests` at the bottom of `crates/tauri/src/ipc/protocol.rs` (`protocol.rs:~550`):

- `fn parse_invoke_request()` — unconditional. Builds an `AppManager<Wry>` via
  `generate_context!("test/fixture/src-tauri/tauri.conf.json", crate, test = true)`, constructs an
  `http::HeaderMap` including `(ORIGIN, HeaderValue::from_str("tauri://localhost"))`, builds an
  `http::Request`, and calls `super::parse_invoke_request(&manager, request)` directly — then
  asserts on the returned `InvokeRequest`'s `cmd`, `callback`, `error`, `invoke_key`, `url`,
  `headers`, `body`.
- `fn parse_invoke_request_isolation()` — gated `#[cfg(feature = "isolation")]`, same shape against
  the `test/fixture/isolation/src-tauri/tauri.conf.json` fixture.

**Gap exposed by the signature change (§4.1):** neither existing test constructs a real
`Webview<R>`/window — they only build an `AppManager` and call `parse_invoke_request` standalone.
Once the function takes a `fallback_origin: &Url` parameter, the existing calls just need an extra
arbitrary `Url` argument (e.g. `&"tauri://localhost".parse().unwrap()`) — no real webview required,
which is exactly why passing a precomputed `Url` (vs. a `&Webview<R>`) was recommended in §4.1.

**New tests to add, same shape as the existing ones:**
1. Omit the `ORIGIN` header entirely from the `HeaderMap` → assert `invoke_request.url` equals the
   `fallback_origin` passed in.
2. Set `ORIGIN` to an empty string → same assertion.
3. Set `ORIGIN` to the literal `"null"` → same assertion.
4. (Regression) keep the existing valid-`Origin` case asserting the header value is still used
   verbatim when present and well-formed — i.e. the fallback must not shadow a good header.
5. Garbage-but-present `Origin` (e.g. `"not a url"`) still errors with
   `"Origin header is not a valid URL"` — confirms §4.2's scope-limiting is in effect.

Repeat the missing/empty/`"null"` cases under `parse_invoke_request_isolation` too, since that test
exists specifically to catch isolation-feature regressions in this function and the fallback logic
sits upstream of the isolation-specific branching.

## 6. Out of scope for PRCVTRFRKP-1.1 / 1.2

- `handle_ipc_message` / the postMessage path: no change needed (§3) — it never had this bug.
- Any change to `is_local_url`, `authority.rs`, or the ACL/capability resolution logic itself —
  this fix only repairs how `InvokeRequest.url` is *obtained* when the header is unusable; it does
  not change how that value is subsequently used for local/remote classification.
- Upstream contribution — per PRCVTRFRKP-1.2's scope (permanent-fork maintenance policy, no
  upstream contribution), this stays a local fork patch.
