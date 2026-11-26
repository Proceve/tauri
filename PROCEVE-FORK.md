# Proceve Fork of `tauri` — Maintenance Record

This repository is a **Proceve-maintained fork** of `tauri-apps/tauri`, carrying one local
security/compatibility patch that is not upstream. Per user directive (2026-10-02), this patch
set will **not** be contributed upstream — it is maintained here permanently, for as long as the
underlying platform defect exists.

## 1. Why this fork exists

`tauri 2.12.1` / `wry 0.57.0`, run against system `webkit2gtk 2.52.6` (Linux), fails every IPC
`invoke()` call with `Origin header is not a valid URL` / `missing Origin header`.

Root cause and investigation are recorded in
`pBlaiseWorkbench/docs/poc-results/item-a-ipc-mitigation-plan.md` (container task PBLSWRK2-6):
WebKitGTK sends a missing, empty, or opaque (`"null"`) `Origin` header for requests made from a
custom URI scheme (the same underlying webview behavior as `tauri-apps/wry#366`, there observed
for external CORS fetches in 2021 — here it trips the *internal* IPC call itself). That plan
confirmed:

- `tauri-v2.12.1` and `wry-v0.57.0` are each the latest published release of their line —
  `Cargo.lock` was already pinned to both, so there is no upstream version bump that fixes this.
- Neither tracker (`tauri-apps/tauri`, `tauri-apps/wry`) had an open issue matching this exact
  failure mode against `webkit2gtk 2.52.6`.

Because the fix requires editing `tauri`'s own IPC source, and the dependent project
(`pBlaiseWorkbench`) consumes `tauri` as a crates.io dependency, the only way to ship the fix is
a local source patch — hence this fork, pinned at tag `tauri-v2.12.1`, carrying the patch
developed under task PRCVTRFRKP-1.1 (`crates/tauri/src/ipc/protocol.rs`, see
`docs/ipc-origin-fallback-fix-design.md` for the fix design that preceded it).

## 2. Patch inventory

Exactly one file is patched: `crates/tauri/src/ipc/protocol.rs`. The change:

1. Propagates the webview's current URL (`Webview::url()`, resolved by the existing caller
   `get()` before the parse) into `parse_invoke_request` as a new `fallback_origin: &Url`
   parameter.
2. Uses that fallback only when the `Origin` header is absent, empty, or the literal `"null"`.
   Any other value is parsed exactly as before — **a well-formed but mismatched `Origin` is still
   rejected**, so cross-origin IPC rejection is unchanged.
3. Adds unit tests covering: missing header, empty header, `"null"` header, a valid/matching
   header (regression — fallback must not shadow a good header), a garbage-but-present header
   (must still hard-error), and the same missing/empty/`"null"` cases repeated under the
   `isolation`-feature test, since the fallback sits upstream of that feature's decrypt step.

Final diff:

```diff
--- a/crates/tauri/src/ipc/protocol.rs
+++ b/crates/tauri/src/ipc/protocol.rs
@@ -59,7 +59,21 @@ pub fn get<R: Runtime>(manager: Arc<AppManager<R>>) -> UriSchemeProtocolHandle
     match *request.method() {
       Method::POST => {
         if let Some(webview) = manager.get_webview(label) {
-          match parse_invoke_request(&manager, request) {
+          let fallback_origin = match webview.url() {
+            Ok(url) => url,
+            Err(e) => {
+              respond(
+                http::Response::builder()
+                  .status(StatusCode::INTERNAL_SERVER_ERROR)
+                  .header(CONTENT_TYPE, mime::TEXT_PLAIN.essence_str())
+                  .body(e.to_string().into_bytes().into())
+                  .unwrap(),
+              );
+              return;
+            }
+          };
+
+          match parse_invoke_request(&manager, &fallback_origin, request) {
             Ok(request) => {
               ...
@@ -433,6 +447,7 @@ fn handle_ipc_message<R: Runtime>(request: Request<String>, manager: &AppManager

 fn parse_invoke_request<R: Runtime>(
   #[allow(unused_variables)] manager: &AppManager<R>,
+  fallback_origin: &Url,
   request: http::Request<Vec<u8>>,
 ) -> std::result::Result<InvokeRequest, String> {
   ...
-  let url = Url::parse(
-    parts
-      .headers
-      .get("Origin")
-      .ok_or("missing Origin header")?
-      .to_str()
-      .map_err(|_| "Origin header value must be a string")?,
-  )
-  .map_err(|_| "Origin header is not a valid URL")?;
+  // WebKitGTK sends a missing, empty, or opaque (`"null"`) `Origin` header for requests made
+  // from a custom URI scheme (see tauri-apps/wry#366), which otherwise hard-fails the IPC call.
+  // Fall back to the current URL of the webview this request is already bound to — it comes from
+  // the platform webview dispatcher, not from page-controlled input, so it can't be spoofed by
+  // page JS. A well-formed but mismatched Origin is NOT covered by this fallback and is still
+  // rejected below, preserving cross-origin IPC rejection.
+  let url = match parts.headers.get("Origin") {
+    Some(value) => {
+      let origin = value
+        .to_str()
+        .map_err(|_| "Origin header value must be a string")?;
+      if origin.is_empty() || origin == "null" {
+        fallback_origin.clone()
+      } else {
+        Url::parse(origin).map_err(|_| "Origin header is not a valid URL")?
+      }
+    }
+    None => fallback_origin.clone(),
+  };
```

(Test-module additions: new `base_headers`/`test_manager` helpers and five new `#[test]`
functions — `parse_invoke_request_origin_missing_falls_back`,
`parse_invoke_request_origin_empty_falls_back`, `parse_invoke_request_origin_null_falls_back`,
`parse_invoke_request_origin_valid_matching_is_unchanged`,
`parse_invoke_request_origin_garbage_present_still_errors` — plus the equivalent cases added to
`parse_invoke_request_isolation`. Omitted above for brevity; see `git diff` /
`git log` on branch `proceve/ipc-origin-fallback` for the complete change.)

**Security reasoning, restated:** the fallback only fires for the three specific shapes WebKitGTK
is documented to emit for custom-URI-scheme requests (absent, empty, `"null"`) and substitutes a
value obtained from the platform webview dispatcher — not from any header or page-controlled
input, so it cannot be influenced by page JS. A present-but-invalid `Origin` (e.g. `"not a
url"`) is a different, unexplained failure mode and is deliberately **not** covered — it still
hard-errors exactly as before. A present-and-valid-but-mismatched `Origin` is untouched by this
patch and continues through the existing `is_local_url` / `Origin::Remote` classification in
`crates/tauri/src/ipc/authority.rs`, so cross-origin IPC calls are still rejected. The fallback
narrows an unconditional hard failure into a conditional, scope-limited one; it does not widen
what IPC traffic is accepted.

## 3. Maintenance policy

- **Base tag:** `tauri-v2.12.1`.
- **Branch:** `proceve/ipc-origin-fallback`.
- **On each upstream `tauri` release:**
  1. Check that release's changelog for a native fix to the WebKitGTK missing/empty/`"null"`
     `Origin` behavior. If one exists, see Retirement below instead of re-applying the patch.
  2. If no native fix: create a new `proceve/*` branch from the new release tag, rebase/re-apply
     this patch onto it.
  3. Re-run `crates/tauri/src/ipc/protocol.rs`'s unit tests (`cargo test -p tauri ipc::protocol`).
  4. Re-run `pBlaiseWorkbench/scripts/measure-item-a.sh` against the rebased fork before adopting
     the new branch as the active patch source.
- **Retirement:** once an upstream release natively fixes this defect, turn off the
  `[patch.crates-io]` override in consuming projects (§4) and resume consuming `tauri` directly
  from crates.io. Keep this fork branch archived (not deleted) for historical reference — do not
  delete `proceve/ipc-origin-fallback` or this file at retirement.
- **No upstream contribution:** per the 2026-10-02 directive, this patch is not filed upstream in
  any form (no issue, no PR). This fork is the sole distribution mechanism for the fix for as long
  as it is needed.

## 4. Consumers

- `pBlaiseWorkbench/src-tauri/Cargo.toml` — intended consumer via a `[patch.crates-io]` entry
  pointing `tauri` at this fork/branch (`proceve/ipc-origin-fallback`). As of this writing that
  entry has not yet been added to `pBlaiseWorkbench/src-tauri/Cargo.toml`; wiring it up is tracked
  separately from this maintenance record, which documents the policy the entry must follow once
  added — any future `[patch.crates-io]` source reference must track this fork's current active
  `proceve/*` branch, updated per the rebase procedure in §3 on every upstream release.
