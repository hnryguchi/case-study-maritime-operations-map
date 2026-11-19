# Decisions (ADR-style)

Short records of decisions that shaped the public map. Each one is grounded in the production code base; the code itself is not published.

## 1. The LLM may veto, never approve on its own

**Context.** Daily position reports are typed by people at sea. Typos happen (a missing digit, a swapped hemisphere), and some errors pass simple numeric checks while still being implausible in context. An LLM can judge context, but it can also be wrong or unavailable.

**Decision.** Hard rules run first (newer than published, inside a regional bounding box, implied speed at most 15 knots). Only reports that pass are sent to the model, with recent history, the operation period and the work-area polygon. The model answers in a strict JSON schema; it can turn a pass into a hold, never the reverse. The position text in the notification is always generated deterministically; the model only adds a comment.

**Consequences.** The worst case of a model error is a held report that a person releases, not a wrong position on a public map. Unparseable or out-of-schema answers are treated as "reviewer unavailable".

## 2. Fail open on the reviewer, fail closed on the rules

**Context.** The reviewer depends on an external API key and network call. If it is down, the map could stop updating.

**Decision.** When the reviewer is unavailable, publish on the rules alone and say so in the notification sent to the operator. When a rule fails, never publish.

**Consequences.** The map keeps updating during an API outage, and the operator can see exactly which publications had no second opinion. The trade-off is accepted because the rules already bound the plausible error.

## 3. No database in the map application

**Context.** Operations are already managed, reviewed and published in the compliance portal. A second database would mean synchronisation and a second source of truth.

**Decision.** Read operations server-side from the portal's public endpoint with a 60-second cache. Store the last reported position and a ten-entry history as one JSON file on the App Service's persistent storage. Serialize ingests in-process.

**Consequences.** Content updates never need a rebuild or redeploy, and the app is cheap to run. A single JSON file and an in-process lock assume one instance; scaling out would need a shared store.

## 4. Custom WebGL2 terrain instead of the built-in raster-DEM terrain

**Context.** The first 3D view used MapLibre's raster-DEM terrain built from the same bathymetry. Frame time and main-thread long tasks while navigating were the bottleneck.

**Decision.** Write a MapLibre custom layer in raw WebGL2: quadtree LOD, patches generated in the vertex shader without vertex buffers, one shared height function to avoid cracks, per-pixel shading and isobaths, a quality governor and context-loss recovery. Build the height tiles offline from a public GEBCO grid as lossless WebP with exact integer metres.

**Consequences.** In the A/B recorded in the project notes: about 2.7x less frame time, no long tasks while navigating, roughly a quarter of the JavaScript heap. The cost is owning shader and geometry code that a library would otherwise maintain, mitigated by tests that tie the client's tile mapping to the builder's metadata.

## 5. Public by default, authentication kept as a runtime switch

**Context.** The first version required a login. The intended users (fishers) should not need an account to see where a survey is working.

**Decision.** Open the interface to everyone, but keep the authentication code (scrypt hash, HMAC-signed cookie) intact and tested, enabled by an app setting read at runtime.

**Consequences.** The product can return to a gated mode with a restart, not a rebuild. Opening the interface did not widen data exposure: the data was already limited to the portal's double-gated public projection.
