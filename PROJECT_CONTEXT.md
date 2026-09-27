# PROJECT_CONTEXT.md v0.3
## Custom Browser — Architecture Contract

**Status:** v0.3 — ✅ **PHASE 1 COMPLETE** (Windows-validated 2026-09-27)
**Authority:** Single source of architectural truth for the Custom Browser project
**Maintainer:** Project owner (Windows machine = execution authority)
**Last updated:** v0.3 — Phase 1 complete; V-5 resolved; ADRs 014-018 added

---

## Changelog

### v0.3 — Phase 1 COMPLETE

- **Phase 1 Windows validation confirmed:** `cargo check` passes (17.60s), `npm run tauri:dev` launches window, all 6 Phase-1 commands exercised end-to-end with correct backend logs and UI updates, horizontal layout renders correctly.
- **V-5 RESOLVED:** Trusted Command API allowlist implemented (F-8). Tauri 2's ACL does NOT auto-grant app-own commands — only plugin commands get auto-generated permissions. App-own commands require a hand-written permission file under `src-tauri/permissions/`. Pattern documented as ADR-014. Verified in both directions on Windows (grant present → works; grant removed → rejected with specific error).
- **New ADRs added (014-018):** Capturing architectural decisions made during Phase 1 implementation.
- **Section 13 updated:** Phase 1 DONE. Phase 2 (real tab system) is next.
- **Section 14 updated:** V-5 moved from "deferred" to "RESOLVED".
- **Known Phase-1 debt carried forward:** CSP `style-src 'unsafe-inline'` in production `csp` field (not just `devCsp`). Acceptable for Phase 1 only because no real web content is loaded. **MUST be resolved before Phase 3** (real WebView2 content). See ADR-017.
- **Deferred to v0.4 (Phase 2 prep):** CSP single-source-of-truth decision — `index.html` meta tag and `tauri.conf.json` both hardcode CSP independently. Build-time generation recommended.

### v0.2 — FROZEN FOR PHASE 1
- v0.2 incorporates ChatGPT review (2 blockers + 12 should-fix items) — see v0.2 entry below.
- **Frozen as architectural authority for Phase 1** per ChatGPT's explicit review decision.
- Phase 1 scope (allowed): Tauri 2 base, React + TypeScript + Vite, Rust backend, Cargo workspace/crate skeleton, Tauri capabilities structure, browser-shell scaffolding, basic native-controller interface, Windows build/test pipeline.
- Phase 1 scope (NOT allowed — gated by deferred items): Isolated Chromium runtime selection (D-1), Chromium capture-policy enforcement (V-7), `getDisplayMedia()` source switching (V-1), extension installation system (D-4), production capture system, final process-tree termination mechanism (D-3).
- **Implementation rule added (per ChatGPT Phase-1 guidance):** A Tauri command being callable from the browser UI is NOT sufficient authorization. Capabilities and command permissions MUST be defined explicitly. Browser UI's privileged surface MUST be minimal. Navigated web content MUST NOT inherit browser UI capabilities. CSP MUST be tight.
- **Deferred to v0.3 (cleanup, not blocking Phase 1):** Section 10.2 wording change — "Full access" → "May access only commands granted to the browser UI through the applicable Tauri capabilities and command allowlist."

### v0.2 (revision)
- **BLOCKER-1 fix (Section 10, ADR-009):** Trust boundary is now layered. Web content is never a trusted caller, even inside the app's own WebView2 environment. "React frontend" redefined as trusted browser UI only.
- **BLOCKER-2 fix (Section 9.6, ADR-010, V-7):** WGC is the native capture mechanism, NOT the enforcement mechanism for Chromium's `getDisplayMedia()` pipeline. Enforcement component is a separate deferred item (Phase-5 gate).
- **SHOULD-FIX 3 (Section 7.2):** Chrome for Testing description corrected — does not auto-update.
- **SHOULD-FIX 4 (Section 8.2, ADR-011):** `--remote-debugging-pipe` strongly preferred; `--remote-debugging-port` is last-resort only with conditions.
- **SHOULD-FIX 5 (Section 7.2, ADR-013):** System Chromium contradiction resolved — rejected for production, dev convenience only.
- **SHOULD-FIX 6 & 11 (Section 6.2, ADR-012):** Responsibility boundary between `crates/` and `native/` defined. Specific overlaps resolved.
- **SHOULD-FIX 7 (Section 10):** Tauri/React frontend wording tightened (related to BLOCKER-1).
- **SHOULD-FIX 8 (Section 10.3):** Profiles clarified — isolated session profile NEVER promoted to persistent.
- **SHOULD-FIX 9 (Section 9.6):** WGC capture target not overstated; browser's custom capture UX distinguished from system picker.
- **SHOULD-FIX 10 (Section 15.2 T-6):** Secure-desktop expected result made precise.
- **SHOULD-FIX 12 (Section 12.2):** UNVALIDATED vs DESIGN ONLY boundary clarified.

### v0.1
- Initial draft.

---

## 1. Purpose & Scope of this Document

### 1.1 Purpose
This document is the **single source of architectural truth** for the Custom Browser project. All agents (ChatGPT, Z.ai/GLM, and any future contributors) MUST implement against this contract. If code or proposals diverge from this document, this document wins — and divergence must be raised as an architectural question, not silently absorbed into the codebase.

### 1.2 Scope
This document defines:
- What the browser is and is not
- Architecture, technology stack, repository structure
- Security boundaries and trust model
- Implementation rules and status labeling
- Anti-invention rules
- Open/deferred questions requiring real Windows/Chromium validation

### 1.3 Out of scope
This document is NOT:
- A feature specification (features live in their own docs)
- A UI/UX specification (lives in `docs/ui/` if/when created)
- A release plan or roadmap
- An implementation guide for any single component
- A test plan

### 1.4 Modification policy
Changes to this document MUST be logged in **Section 16 (Decision Log)** with ADR-style entries. No silent edits.

---

## 2. What the Browser Is

A Windows-native, Chromium-compatible custom browser built on Tauri 2 + Rust + React/TypeScript + WebView2 (normal browser), with an on-demand Isolated Chromium runtime for controlled browsing sessions.

**Design philosophy:**
- Hardened by default
- Flexible by choice
- Lightweight by design

**Capabilities (high-level, not a spec):**
- Normal browser: tabs, windows, profiles, workspaces, history, bookmarks, downloads, session restore
- Privacy & security: tracker blocking, fingerprinting protection, cookie controls, site permissions, security dashboard
- Browser Analysis Mode: clipboard isolation, download quarantine, notification/permission/external-launch control, redirect monitoring
- Isolated Chromium: disposable-profile Chromium runtime with multi-tab support and creator-controlled capture policy
- Capture system: video / audio / snipping across both browser surfaces
- Media, documents, dev tools, resource management, Windows integration, updates

---

## 3. What the Browser Is NOT

- **Not a Chrome clone** — the goal is Chrome-compatible behavior, not impersonation of official Google Chrome
- **Not claiming to be official Google Chrome** — UA, UA-CH, and navigator-visible properties must reflect a Chromium runtime, not impersonate Chrome
- **Not a VM-based sandbox** — the Isolated Chromium runs as a real Chromium process on the same real Windows machine; isolation comes from controlled profile, lifecycle, permissions, processes, downloads, capture surfaces, and external launches — not virtualization
- **Not a Linux/macOS/Android/iOS browser** — Windows-native only
- **Not a single-embedded-webview isolation tool** — Isolated Chromium supports multiple tabs and independent browser state
- **Not dependent on installed Google Chrome being present** — the Isolated Chromium runtime is bundled or built, not assumed
- **Not a malware lab or threat-analysis appliance** — explicitly out of scope
- **Not a forensic/anti-evasion training environment** — explicitly out of scope

---

## 4. Architecture

### 4.1 High-level structure

```
                         CUSTOM BROWSER
                              │
             ┌────────────────┴────────────────┐
             │                                 │
             ▼                                 ▼
      NORMAL BROWSER                    ISOLATED CHROMIUM
             │                                 │
       Tauri 2 + WebView2               Separate process tree
             │                                 │
       React / TypeScript                Chromium runtime
             │                                 │
       Rust native layer                Disposable profile
             │                                 │
             └──────────────┬──────────────────┘
                            │
                     Browser Controller
                            │
                ┌───────────┼───────────┐
                ▼           ▼           ▼
             Capture      IPC       Lifecycle
```

### 4.2 Security boundary

```
                    SECURITY BOUNDARY
                          │
                          ▼
              ┌─────────────────────┐
              │ Native Controller   │
              └─────────────────────┘
                   │            │
                   ▼            ▼
              Process        Capture
              Control        Policy
```

The webpage itself does **not** get to instruct the native controller to expose the physical desktop or change capture policy. Capture-source transitions are application-controlled decisions, not webpage-controlled.

### 4.3 Normal Browser vs Isolated Chromium

| Dimension | Normal Browser | Isolated Chromium |
|---|---|---|
| Rendering | Tauri WebView2 | Separate Chromium runtime |
| Process model | Within Tauri app | Separate process tree |
| Profile | Parent-owned, persistent | Disposable, per-session |
| Persistent state | Yes (bookmarks, history, downloads, etc.) | No (by design) |
| Lifecycle | App lifetime | On-demand launch → clean termination |
| Multi-tab | Yes | Yes |
| Capture source | Tauri window / WebView2 surface | Isolated Chromium window (policy-controlled) |

---

## 5. Technology Stack

### 5.1 Required technologies

- **Tauri 2** — desktop shell + native Windows integration
- **Rust** — native backend, process/lifecycle/security control
- **React + TypeScript + Vite** — browser UI
- **WebView2** — normal-browser rendering (Tauri default on Windows)
- **Chromium runtime (TBD selection)** — Isolated Chromium rendering engine
- **Git** — version control + cross-machine sync

### 5.2 Version policy

- Versions are pinned **only after** the corresponding toolchain/runtime has been selected and verified on the target Windows development environment.
- Pinned versions are recorded in **Section 5.3** and MUST include:
  - The exact version
  - The date pinned
  - The Windows channel it was last verified on (e.g., Windows 11 23H2, stable)
  - The validator (who ran the verification)
- Unpinned technologies are listed in Section 5.1 only; absence from 5.3 means "not yet validated."

### 5.3 Pinned versions

*(None pinned at v0.1. All technologies in 5.1 are currently UNVALIDATED pending Phase 1 Windows verification.)*

### 5.4 Version validation

Each pinned entry MUST be revalidated when:
- The Windows major version changes
- The runtime vendor ships a breaking change
- The project upgrades across a major version boundary

A pinned version whose last-verified date is older than 6 months MUST be re-validated before being relied on for new work.

---

## 6. Repository Structure

```
custom-browser/
│
├── PROJECT_CONTEXT.md          ← this document (architectural authority)
├── README.md
├── LICENSE
├── .gitignore
├── CONTRIBUTING.md
├── SECURITY.md
│
├── docs/
│   ├── architecture/
│   │   ├── overview.md
│   │   ├── browser-architecture.md
│   │   ├── isolated-chromium.md
│   │   ├── capture-architecture.md
│   │   └── windows-integration.md
│   │
│   ├── security/
│   │   ├── threat-model.md
│   │   ├── isolation-model.md
│   │   └── security-boundaries.md
│   │
│   ├── decisions/              ← ADR records (mirrors Section 16 of this doc)
│   │   └── README.md
│   │
│   └── implementation-status.md  ← central feature status registry (see Section 12)
│
├── apps/
│   └── browser/
│       ├── src/                ← React + TypeScript UI
│       ├── src-tauri/          ← Tauri config + Rust integration
│       ├── public/
│       ├── package.json
│       ├── vite.config.ts
│       └── tsconfig.json
│
├── crates/                     ← Rust workspace crates
│   ├── browser-core/
│   ├── browser-state/
│   ├── profile-manager/
│   ├── workspace-manager/
│   ├── download-manager/
│   ├── privacy-engine/
│   ├── permission-manager/
│   ├── process-manager/
│   ├── isolated-chromium/
│   ├── capture-manager/
│   ├── media-manager/
│   ├── resource-manager/
│   └── windows-integration/
│
├── native/                      ← Native (non-Rust) Windows components
│   ├── isolated-chromium/
│   │   ├── launcher/
│   │   ├── process-control/
│   │   ├── profile/
│   │   ├── ipc/
│   │   └── capture/
│   │
│   ├── capture/
│   │   ├── video/
│   │   ├── audio/
│   │   └── snipping/
│   │
│   └── windows/
│       ├── window-control/
│       ├── process-control/
│       └── media-capture/
│
├── shared/
│   ├── types/                  ← shared TypeScript types
│   ├── protocols/              ← IPC protocol definitions
│   └── constants/
│
├── scripts/
│   ├── setup/
│   ├── development/
│   ├── build/
│   └── packaging/
│
└── tests/
    ├── browser/
    ├── isolated-chromium/
    ├── capture/
    ├── privacy/
    └── integration/
```

All directories are **DESIGN ONLY** at v0.2. No code exists yet.

### 6.2 Responsibility boundaries: `crates/` vs `native/`

To prevent duplication between Rust and native components (per ChatGPT SHOULD-FIX 6 & 11):

| Layer | Owns |
|---|---|
| `crates/` (Rust) | Application-level orchestration, state, policy, IPC contracts, lifecycle coordination. Each crate exposes a Rust-facing interface. |
| `native/` (Windows-specific) | Platform/runtime-specific implementation requiring APIs/SDKs better handled outside Rust. Each native component MUST have exactly one Rust-facing interface (in the corresponding `crates/` crate). |

**Rule:** No functionality is implemented in both `crates/` and `native/`. If logic exists in `crates/process-manager`, the corresponding `native/windows/process-control/` contains only the platform-specific primitives called BY the Rust crate — never a parallel implementation.

**Specific overlaps resolved:**

- `crates/profile-manager` owns **profile lifecycle policy** (creation, disposal ordering, session scoping). `native/isolated-chromium/profile/` contains only the Chromium-runtime-specific profile creation/teardown primitives.
- `crates/process-manager` owns process-tree lifecycle policy. `native/windows/process-control/` contains only the Windows-specific process-tree termination primitives (Job Objects, etc.).
- `crates/capture-manager` owns capture policy + session orchestration. `native/capture/` and `native/windows/media-capture/` contain only the platform-specific capture primitives (WGC, audio, snipping).

---

## 7. Isolated Chromium — Definition + Lifecycle

### 7.1 Definition

The Isolated Chromium is a **separately launched, Chromium-based browser runtime** with its own disposable profile, running as a separate process tree on the same real Windows machine.

It is **NOT**:
- A virtual machine
- Official Google Chrome
- A single embedded webview

It IS:
- Chromium runtime + Chrome-compatible web behavior + Chrome-extension compatibility + our own browser controls + our own lifecycle/capture integration

### 7.2 Runtime selection (DEFERRED)

The exact runtime mechanism is **deferred to Phase 4 validation**. Named candidates:

1. **CEF (Chromium Embedded Framework)** — embeddable, mature, but different integration surface
2. **Chrome for Testing** — Google-published, versioned Chrome binary intended for testing/automation; **does not auto-update**. Provides reproducible pinned binaries (explicitly designed for stable, versioned testing).
3. **Custom Chromium build** — full control, high maintenance cost
4. **System Chromium** — depends on user having installed Chromium. **Rejected for production architecture** (we don't want a hard dependency on user-installed Chromium). Retained only as a development/testing convenience, NOT a production runtime candidate.

**Hard requirement:** whatever runtime is selected MUST support **CDP (Chrome DevTools Protocol)** via `--remote-debugging-pipe` or `--remote-debugging-port`, because ADR-003 depends on it.

### 7.3 Lifecycle

```
[ Isolate Browser ]
        │
        ▼
Launch Chromium process tree
        │
        ▼
Create disposable profile (real Chromium profile dir under temp session dir)
        │
        ▼
Create isolated browser session
        │
        ▼
User browses normally (multi-tab)
        │
        ▼
[ Close Isolation ]
        │
        ▼
Terminate Chromium process tree (mechanism TBD — Windows Job Objects leading candidate)
        │
        ▼
Confirm process tree terminated (audit step)
        │
        ▼
Delete disposable session data
```

The parent browser maintains knowledge of the isolated session: tabs, processes, capture surfaces.

### 7.4 Profile format

Real Chromium-style profile directory, created under a unique temporary/session directory. Cleanup happens **after** the child process tree is confirmed terminated (not before; not concurrent).

---

## 8. Parent ↔ Isolated Chromium IPC Contract

### 8.1 Two channels

1. **CDP (Chrome DevTools Protocol)** — for browser/page-level control where supported (navigation, tabs, DOM, network inspection, screencast, etc.)
2. **Native IPC channel** — for lifecycle/state events not covered by CDP (launch, terminate, capture policy changes, profile cleanup signals)

### 8.2 Transport (DEFERRED)

CDP transport policy:
- **Preferred:** `--remote-debugging-pipe` (no TCP exposure, stdio-based)
- **Last-resort fallback only:** `--remote-debugging-port` — ONLY acceptable if ALL of the following are true:
  - Pipe is unavailable on the chosen runtime
  - Bound to loopback only (`127.0.0.1`)
  - Inaccessible to untrusted processes/connections
  - Authenticated/isolated as required
  - Explicitly validated on Windows

`--remote-debugging-port` is NOT a normal fallback. It is a last-resort compatibility path with elevated security review. A runtime that cannot support `--remote-debugging-pipe` becomes a hard constraint on runtime selection (D-1), not a reason to fall back to port.

Native IPC transport: candidates are stdio pipes, Windows named pipes, or local WebSocket. Selection TBD (depends on D-1).

### 8.3 Security boundary

**CDP is NOT the security boundary.** CDP is a control channel with privileged access to the child browser. The security boundary is the **Native Controller** (Section 10), which decides:
- Who can issue which commands
- When commands are allowed (e.g., user-gesture requirements)
- What capture-source transitions are permitted

The child Chromium process itself is treated as **untrusted** from the parent's perspective — it cannot, for example, request capture-source transitions on its own behalf.

---

## 9. Capture Architecture

### 9.1 Unified capture

Capture belongs directly to the browser, covering both surfaces:

```
CAPTURE
│
├── VIDEO
│   ├── Normal Browser (per-tab, per-window)
│   └── Isolated Chromium (per-tab, per-window, policy-controlled)
│
├── AUDIO
│   ├── Normal Browser
│   └── Isolated Chromium
│
└── SNIPPING
    ├── Normal Browser
    └── Isolated Chromium
```

### 9.2 Snipping flow

```
Focused Browser Surface
        ↓
[ Snip ]
        ↓
Freeze Current Frame
        ↓
Rectangle Selection
        ↓
OCR / Annotation / Crop / Redaction
        ↓
Copy or Save
```

Rectangle-only capture. Includes OCR, QR detection, annotation, image/text copying.

### 9.3 First implementation target

**Windows Graphics Capture API** (per-window / per-HWND capture) is the first implementation target for video capture.

**CDP `Page.startScreencast`** is an additional browser-level capture mechanism for Isolated Chromium, NOT a replacement for Windows Graphics Capture.

### 9.4 Isolated capture policy

The Isolated Chromium's webpage does NOT get access to the physical desktop through the browser's capture layer. Capture-source transitions are native-controller decisions:

```
Isolated Chromium visible
        ↓
Capture source = Isolated Chromium window

Isolated Chromium minimized
        ↓
Capture source = Real Windows Desktop (or other surface, per policy)

Isolated Chromium restored
        ↓
Capture source = Isolated Chromium window
```

### 9.5 Mid-stream source switching (UNVALIDATED)

The desired behavior — switching an already-running media stream's source without a fresh website-level permission prompt — is **UNVALIDATED**.

- **Status:** UNVALIDATED
- **Owner:** Project owner + Z.ai (must be proven on real Windows + real Chromium)
- **Fallback (must be defined before implementation begins):** candidates include:
  - Terminate stream + require fresh permission (least-desirable UX)
  - Black-frame transition + silent source swap (if technically possible)
  - Restrict capture to Isolated Chromium surface only (drop desktop fallback)
- **The fallback selection is itself an architecture decision and MUST be raised with ChatGPT before implementation.**

### 9.6 Native capture vs Chromium `getDisplayMedia()` enforcement (BLOCKER-2 fix)

WGC (Windows Graphics Capture) is the **native capture mechanism** for capturing a specific HWND (e.g., the Isolated Chromium window). Microsoft's API supports `GraphicsCaptureItem.CreateForWindow(HWND, ...)` for targeting a specific window.

But WGC is **NOT, by itself, the enforcement mechanism** for Chromium's webpage `getDisplayMedia()` pipeline.

When a webpage calls `navigator.mediaDevices.getDisplayMedia(...)`, the call enters **Chromium's own media-capture / source-selection pipeline**. WGC does not automatically constrain that pipeline.

The architecture therefore requires an **explicit component** that mediates:

```
Webpage
   │
   │ getDisplayMedia()
   ▼
Chromium capture pipeline
   │
   ▼
CUSTOM CAPTURE POLICY (native controller)
   │
   ├── allow isolated HWND (per current policy)
   ├── allow real desktop (only when isolated window minimized, per policy)
   └── deny
```

**Status:** UNVALIDATED — see V-7 in Section 14.2.

**Required:** Determine how the selected Chromium runtime enforces application-controlled capture-source policy for `getDisplayMedia()`. Candidates include:
- Chromium source modification (custom build only — D-1 candidate #3)
- Browser-level media-capture hook / interceptor (if supported by chosen runtime)
- CDP-based media-capture redirection (if supported)
- Other integration point TBD

This is a **Phase-5 capture-system gate**, NOT a Phase-1 browser-shell gate. But it MUST be resolved before implementation of the capture feature begins.

**Note:** The browser's custom capture UX (snipping, capture overlays, etc.) MUST be distinguished from invoking the standard Windows system capture picker (`GraphicsCapturePicker`). The browser's capture surface is its own application-controlled layer, not a thin wrapper around the system picker.

---

## 10. Security Boundaries & Trust Model

### 10.1 Security boundary

The security boundary is the **Native Controller**, which mediates:
- **Process Control** — launch, terminate, job-object management of the Isolated Chromium
- **Capture Policy** — what surfaces may be captured, when, by whom

The Native Controller exposes a **Trusted Command API** — an explicitly-allowlisted set of commands. **Web content is NEVER a trusted caller**, even when rendered inside the application's own WebView2 environment.

Microsoft's WebView2 security model explicitly warns that hosted web content must be treated as insecure, and that exposing native APIs to web content creates security vulnerabilities. (Reference: [Microsoft Learn — Develop secure WebView2 apps](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/security))

The trust model is therefore **layered**:

```
                    NATIVE CONTROLLER
                           ▲
                           │
                    TRUSTED COMMAND API
                   (explicitly allowlisted)
                           ▲
                           │
                 Tauri privileged layer
                 (Rust-side command router)
                           ▲
                           │
                    Browser UI only
              ┌────────────┴────────────┐
              │                         │
       Trusted application UI       Web content
       (browser chrome, tabs,         (WebView2 / Isolated
        workspace, etc.)              Chromium page content)
                                      │
                                      X
                              NO native authority
                  (commands not transitively callable
                   from web content)
```

### 10.2 Trust model

**Trusted callers (may invoke Trusted Command API):**
- The Tauri privileged layer (Rust-side command router), invoked from trusted browser UI via Tauri IPC
- Trust is established by **process identity** — the trusted caller runs in the Tauri app process. Tauri IPC is in-process, not network-exposed, so no separate auth token is required at v0.2. (If the architecture later introduces network-exposed IPC, this rule must be revisited — V-5.)

**Untrusted callers (may NOT invoke Native Controller directly OR transitively):**
- Webpage content in the Normal Browser (WebView2 renderer)
- Webpage content in the Isolated Chromium
- The Isolated Chromium child process itself
- Any code path reachable from untrusted content

**Command allowlist rules:**
- **Webpage-originated commands:** NONE. Web content has no direct native authority.
- **Frontend (browser UI)-originated commands:** Full access to the Trusted Command API, subject to:
  - Explicit allowlist (V-5 enumerates per-command)
  - User-gesture requirements for destructive operations
  - Origin/process verification — must originate from the Tauri app process, NOT from web content
- **Child-process-originated commands (Isolated Chromium → parent):** Lifecycle notifications only; no policy-changing authority

**Anti-confusion rule (per ChatGPT SHOULD-FIX 7):**
"React frontend" in this document means **trusted browser UI rendered in the Tauri app context**, NOT arbitrary React-rendered web content. Web content rendered inside WebView2 — even if it happens to be served by the browser itself — is **untrusted** unless it is explicitly part of the browser chrome.

### 10.3 State ownership

**Parent-owned persistent state:**
- Normal browser profiles (with bookmarks, history, cookies, etc.)
- Profile metadata
- Preferences / settings
- Workspaces
- Downloads (persistent record + files)

**Isolated Chromium state:**
- Temporary disposable session profile only
- **NEVER promoted to persistent browser profile**
- Deleted after the child process tree is confirmed terminated

This rule applies uniformly across all features that touch storage. **No exceptions.** Specifically: an isolated session's profile, history, cookies, downloads, and cache are never persisted — they are destroyed at session end.

---

## 11. Implementation Rules

1. **All code must conform to this document.** Divergence is raised as an architectural question, not absorbed silently.
2. **New design questions raised during implementation must be escalated** to ChatGPT for resolution before the implementation proceeds. Do not improvise answers that would cause architectural drift.
3. **Status flows through `docs/implementation-status.md`**, not per-file labels. See Section 12.
4. **Feature branches** are used for all implementation work:
   - `feature/browser-shell`
   - `feature/tab-system`
   - `feature/isolated-chromium`
   - `feature/capture`
   - `feature/privacy`
   - (additional branches as needed)
5. **Git remote is the synchronization mechanism**, not the architectural authority. The architectural authority is this document.
6. **Windows build/test results are the only valid evidence of implementation.** Code that compiles in any non-Windows environment is DESIGN ONLY until verified on Windows.

---

## 12. Status Labeling System

### 12.1 Central registry

All feature/component status is maintained in **`docs/implementation-status.md`** at the repository root.

No per-file status labels. The codebase is not annotation soup.

### 12.2 Status labels

| Label | Definition |
|---|---|
| **IMPLEMENTED** | Code exists, builds on Windows, verified against real Chromium/WebView2/Win32. Behavior matches spec. |
| **EXPERIMENTAL** | Code exists, partially verified, known gaps remain. Use with caution. |
| **UNVALIDATED** | Functional implementation exists OR a concrete implementation candidate exists, but it has NOT passed required real-world validation on real Windows/Chromium. Code may exist as a draft but must not be relied on. |
| **BLOCKED** | Validation attempted, returned negative or unknown result. Needs unblock (architectural decision, runtime upgrade, or external fix). |
| **DESIGN ONLY** | Intentional spec exists; **NO functional implementation exists**. No code beyond scaffolding or type signatures. |

**Boundary rule (per ChatGPT SHOULD-FIX 12):** If functional code exists (even as a draft) but is unverified, it is **UNVALIDATED**, not DESIGN ONLY. DESIGN ONLY means no functional code exists at all.

### 12.3 Initial status (v0.2)

| Feature | Status |
|---|---|
| Tauri shell | DESIGN ONLY |
| Browser UI | DESIGN ONLY |
| Tab management | DESIGN ONLY |
| Isolated Chromium runtime | UNVALIDATED |
| Chromium IPC | UNVALIDATED |
| Video capture | DESIGN ONLY |
| Audio capture | DESIGN ONLY |
| Snipping | DESIGN ONLY |
| Privacy engine | DESIGN ONLY |
| Process manager | DESIGN ONLY |
| Workspaces | DESIGN ONLY |
| Downloads | DESIGN ONLY |
| DevTools integration | DESIGN ONLY |
| Resource manager | DESIGN ONLY |
| Windows integration | DESIGN ONLY |
| Updates | DESIGN ONLY |

### 12.4 Status update rules

- Status transitions require evidence (build log, test result, screenshot)
- Transitions are recorded as a single line in `docs/implementation-status.md` with date and validator
- No status may regress silently; regressions are BLOCKED entries with reason

---

## 13. Current Phase

**Phase 1: Tauri 2 + React + Rust base project — ✅ COMPLETE**

PROJECT_CONTEXT.md v0.3 reflects Phase 1 completion. Phase 2 (real tab system) is next.

### Phase 1 completion evidence (Windows-validated 2026-09-27)

- ✅ `npm install` → 261 packages, clean
- ✅ `cargo check` → Finished, 17.60s
- ✅ `npm run tauri:dev` → window opens, UI renders, dev server serves
- ✅ All 6 Phase-1 Tauri commands exercised end-to-end with correct backend logs and UI updates
- ✅ F-8 allowlist verified in both directions (grant present → works; grant removed → rejected)
- ✅ F-9 horizontal layout confirmed on Windows
- ✅ No CSP warnings in DevTools console

**Environment:** Rust 1.97.1, Cargo 1.97.1, Node 24.18.0, npm 11.16.0, Tauri 2.12.0, tauri-build 2.7.0, tauri-runtime 2.12.0, Windows 11 build 26200.9457

### Phase 1 scope (delivered)

- Tauri 2 base project ✅
- React + TypeScript + Vite frontend ✅
- Rust backend (Cargo workspace + 13 crate skeletons) ✅
- Tauri capabilities structure (explicit, minimal, with F-8 allowlist) ✅
- Browser-shell scaffolding (tab strip, address bar, window controls) ✅
- Basic native-controller interface (Rust command router) ✅
- Windows build/test pipeline ✅

### Phase 1 scope (NOT delivered — correctly gated by deferred items)

- Isolated Chromium runtime selection (D-1) — Phase 4+
- Chromium capture-policy enforcement (V-7) — Phase 5+
- `getDisplayMedia()` source switching (V-1) — Phase 5+
- Extension installation system (D-4) — Phase 6+
- Production capture system — Phase 5+
- Final process-tree termination mechanism (D-3) — Phase 4+

### Phase 2 scope (next)

- Real tab system (replacing stub `TabStub` state in `native_controller.rs` with actual tab lifecycle management)
- Tab switching (`set_active_tab` command)
- Tab state persistence
- Workspace management foundation

### Phase roadmap (reference only — not authoritative)

| Phase | Focus | Status |
|---|---|---|
| 0 | Architecture + repository contract | ✅ DONE |
| 1 | Tauri 2 + React + Rust base project | ✅ DONE |
| 2 | Basic browser shell / tabs / navigation | NEXT |
| 3 | Native Windows integration | Pending |
| 4 | Isolated Chromium | Pending (gated by D-1) |
| 5 | Capture system | Pending (gated by V-1, V-7) |
| 6 | Privacy / security controls | Pending |
| 7 | Advanced features | Pending |

---

## 14. Open Questions / Deferred Validation Items

The following items are **deferred** and require real Windows/Chromium validation before they can leave their current status. They are not silently designable.

### 14.1 Architecture decisions deferred

| # | Item | Candidates | Why deferred |
|---|---|---|---|
| D-1 | Isolated Chromium runtime selection | CEF / Chrome for Testing / custom Chromium build / system Chromium (rejected at framework level) | Affects UA, extension story, IPC transport, capture mechanism. Requires Windows validation. |
| D-2 | Native IPC transport | stdio / Windows named pipe / local WebSocket | Depends on D-1 and on real-world performance/reliability testing. |
| D-3 | Process tree termination mechanism | Windows Job Objects (leading) / `taskkill /T` / WMI | All candidates need Windows validation; correctness of tree-kill is critical for profile cleanup ordering. |
| D-4 | Extension sideloading / install mechanism | Manual CRX sideload / unpacked load (`--load-extension`) / curated allowlist / Web Store proxy | "Maximum practical Chrome Web Store compatibility" is a target, not a mechanism. Each candidate has different compatibility and security implications. |

### 14.2 Implementation validation deferred

| # | Item | Why deferred |
|---|---|---|
| V-1 | `getDisplayMedia()` mid-stream source switching | UNVALIDATED. Requires real Windows + real Chromium test matrix (see Section 15.2). |
| V-2 | Mid-stream capture fallback selection | Depends on V-1 outcome. Architecture decision required before implementation. |
| V-3 | UA / UA-CH version pin | Pins to chosen runtime's actual Chromium major (depends on D-1). |
| V-4 | WebView2 ↔ Chromium feature parity for capture/extension behaviors | WebView2 is Edge-based; may diverge from Chromium target. Requires audit. |
| V-5 | ~~Trust model command allowlist enumeration~~ **RESOLVED in v0.3** | ✅ RESOLVED. F-8 implemented the allowlist pattern: app-own commands require hand-written permission files under `src-tauri/permissions/`, granted via capability files. Pattern documented as ADR-014 and `docs/security/trust-model.md`. Verified in both directions on Windows. |
| V-6 | Anti-invention test matrix execution (Section 15.2) | Must be run on real Windows/Chromium before mid-stream swap can leave UNVALIDATED. |
| V-7 | **Chromium media-capture policy integration** (BLOCKER-2 enforcement mechanism) | UNVALIDATED. WGC is native capture, NOT Chromium pipeline enforcement. Need to identify the integration point (Chromium source mod / browser-level hook / CDP redirection / other) on the chosen runtime (D-1). **Phase-5 capture-system gate.** |

---

## 15. Anti-Invention Rules

### 15.1 The rule

```
If a feature depends on Windows, Chromium internals, WebView2,
Win32/COM, graphics capture, process behavior, or browser
media-capture behavior:

DO NOT invent an implementation merely because it looks plausible.

Mark it:
    IMPLEMENTED
    EXPERIMENTAL
    UNVALIDATED
    BLOCKED
or
    DESIGN ONLY

Code that merely looks correct is not considered validated.
```

Specifically:
- `getDisplayMedia()` mid-stream source switching → NO ASSUMPTION; real Windows + real Chromium test required.
- Process-tree cleanup across Windows → NO ASSUMPTION; mechanism must be proven.
- Fullscreen interception behavior → NO ASSUMPTION.
- Capture behavior under window occlusion / minimization → NO ASSUMPTION.
- Extension installation / sideloading → NO ASSUMPTION.

### 15.2 Test matrix for `getDisplayMedia()` mid-stream source switching

Before this feature can leave UNVALIDATED, the following scenarios MUST be tested on real Windows + real Chromium:

| # | Scenario | Expected behavior |
|---|---|---|
| T-1 | Isolated Chromium visible → minimize → restore | Stream source transitions cleanly, no re-prompt |
| T-2 | Isolated Chromium visible → occluded by another window | Behavior defined and tested |
| T-3 | Isolated Chromium visible → display disconnects | Graceful degradation; no crash |
| T-4 | Multi-monitor: isolated window on secondary display, primary display disconnects | Defined behavior |
| T-5 | RDP session: capture behavior under RDP | Defined behavior |
| T-6 | Secure-desktop transition (UAC prompt) | Capture must NOT leak pixels from the secure desktop. **Acceptable outcomes:** stream failure, black frame, or fallback transition. **Stream MUST NOT continue rendering secure-desktop pixels.** (No "normal continuation" interpretation allowed.) |
| T-7 | Source swap latency measurement | Must be imperceptible to user OR fallback applies |
| T-8 | Permission prompt suppression across swap | Whether prompt can be avoided; if not, fallback applies |

Failure of any scenario triggers fallback selection (V-2).

---

## 16. Decision Log

ADR-style records. Each entry: Context, Decision, Consequences.

### ADR-001: Normal browser = Tauri 2 + WebView2

- **Context:** Normal browser needs Windows-native rendering with rich UI.
- **Decision:** Use Tauri 2 with WebView2 as the normal-browser rendering engine.
- **Consequences:** Normal browser is independent from the Isolated Chromium runtime. WebView2 is Edge-based and may diverge from Chromium target — see V-4.

### ADR-002: Isolated Chromium = separately launched Chromium runtime with disposable profile

- **Context:** Isolation must come from controlling profile, lifecycle, permissions — not virtualization.
- **Decision:** Isolated Chromium is a separately launched Chromium-based browser runtime with its own disposable profile. Exact runtime mechanism deferred (D-1).
- **Consequences:** Project does NOT depend on installed Google Chrome being present. Hard requirement: chosen runtime MUST support CDP (see ADR-003).

### ADR-003: Parent ↔ child IPC = CDP + native IPC; CDP not security boundary

- **Context:** Need both browser-level control and lifecycle/state event communication.
- **Decision:** CDP for browser/page-level control where supported; native IPC channel (transport TBD — D-2) for lifecycle/state events. CDP is NOT the security boundary.
- **Consequences:** Security boundary is the Native Controller (Section 10). Child Chromium is untrusted from parent's perspective.

### ADR-004: Extensions target MV3 compatibility; Web Store install is compatibility target, not guarantee

- **Context:** Goal is practical Chrome-extension compatibility, not Chrome impersonation.
- **Decision:** Target MV3 / Chrome-extension compatibility. Web Store installation is a compatibility target, NOT guaranteed by UA spoofing. Sideloading/installation mechanisms implemented and tested separately (D-4).
- **Consequences:** Some Web Store extensions may not install. Compatibility is best-effort, not contractual.

### ADR-005: UA pins to actual Chromium major of chosen runtime

- **Context:** UA must be internally consistent; we don't impersonate official Chrome.
- **Decision:** UA + UA-CH pin to the actual Chromium source/runtime version used by the isolated browser build. Version not pinned at v0.1 (depends on D-1).
- **Consequences:** UA changes if runtime selection changes. UA must not be invented as a fake permanent version.

### ADR-006: Capture first target = Windows Graphics Capture per-window

- **Context:** Need reliable per-window capture for both browser surfaces.
- **Decision:** First implementation target is Windows Graphics Capture API (per-HWND). CDP `Page.startScreencast` is an additional browser-level mechanism, not a replacement.
- **Consequences:** Implementation depends on Win32/COM Graphics Capture API. Must be validated on Windows.

### ADR-007: Mid-stream capture source switching = UNVALIDATED

- **Context:** Desired UX is minimized → real desktop → restored → isolated surface without re-prompting. Unknown feasibility.
- **Decision:** Mark UNVALIDATED. Fallback selection (V-2) is a required architecture decision before implementation begins.
- **Consequences:** This feature may not be implementable as desired. Fallback strategies exist but require ChatGPT input before selection.

### ADR-008: Disposable profile = real Chromium profile dir under temp session dir

- **Context:** Isolated Chromium needs a clean profile per session, deleted after termination.
- **Decision:** Use a real Chromium-style profile directory created under a unique temporary/session directory. Cleanup happens after the child process tree is confirmed terminated.
- **Consequences:** Profile creation is standard Chromium behavior. Cleanup ordering matters; process-tree termination confirmation is required before deletion.

### ADR-009: Trust boundary = layered; web content never trusted (BLOCKER-1 fix)

- **Context:** v0.1 said "Tauri/React frontend = full native controller access." That is dangerously broad — Microsoft's WebView2 security model explicitly warns that hosted web content must be treated as insecure, and exposing native APIs to web content creates security vulnerabilities.
- **Decision:** Trust model is layered: Native Controller → Trusted Command API (explicitly allowlisted) → Tauri privileged layer (Rust command router) → Trusted browser UI → Web content (untrusted). Web content is **never** a trusted caller, even inside the app's own WebView2. Commands exposed to the frontend must be explicitly allowlisted and must not be transitively callable by arbitrary page content.
- **Consequences:** Requires per-command allowlist (V-5). Trust established by process identity (Tauri in-process IPC). "React frontend" in this doc = trusted browser UI only, NOT arbitrary React-rendered web content. If network-exposed IPC is introduced later, this ADR must be revisited.

### ADR-010: Capture policy enforcement ≠ native capture mechanism (BLOCKER-2 fix)

- **Context:** v0.1 listed WGC as the capture mechanism and stated "webpage cannot expose physical desktop." But WGC is a native capture mechanism (it captures an HWND), NOT an enforcement mechanism for Chromium's webpage `getDisplayMedia()` pipeline. The webpage's `getDisplayMedia()` call enters Chromium's own media-capture / source-selection pipeline, which WGC does not automatically constrain.
- **Decision:** Architecture requires an **explicit component** mediating between Chromium's webpage media-capture pipeline and the native capture policy. WGC remains the native capture primitive; the enforcement mechanism (V-7) is a separate component whose integration point depends on the chosen runtime (D-1).
- **Consequences:** V-7 added as deferred validation. Phase-5 capture-system gate (not Phase-1). Browser's custom capture UX must be distinguished from invoking the standard Windows system capture picker (`GraphicsCapturePicker`).

### ADR-011: CDP transport — pipe strongly preferred; port is last-resort only

- **Context:** `--remote-debugging-port` exposes a TCP endpoint with materially different attack surface than `--remote-debugging-pipe` (stdio-based, no TCP exposure). v0.1 treated them as near-equivalent fallbacks.
- **Decision:** Pipe is the strongly preferred transport. Port is a last-resort compatibility path requiring loopback binding, untrusted-process inaccessibility, authentication/isolation, AND explicit Windows validation.
- **Consequences:** If the chosen runtime (D-1) does not support `--remote-debugging-pipe`, this becomes a **hard constraint** on runtime selection — not a reason to fall back to port. Port use requires elevated security review and is not the default path.

### ADR-012: `crates/` vs `native/` responsibility boundary

- **Context:** Repository structure had overlapping responsibilities (e.g., `crates/process-manager` vs `native/windows/process-control`, `crates/capture-manager` vs `native/capture`, `crates/profile-manager` vs `native/isolated-chromium/profile`).
- **Decision:** `crates/` (Rust) owns application-level orchestration, state, policy, IPC contracts, lifecycle coordination. `native/` owns platform/runtime-specific implementation primitives. Each native component has exactly one Rust-facing interface in the corresponding crate. **No functionality is implemented in both.**
- **Consequences:** Specific overlaps resolved (see Section 6.2): `profile-manager`, `process-manager`, `capture-manager` each own policy; their `native/` counterparts own only platform primitives called BY the Rust crates.

### ADR-013: System Chromium is not a production runtime candidate

- **Context:** v0.1 said system Chromium was "rejected at framework level" but also "retained only as fallback" — contradictory.
- **Decision:** System Chromium is **rejected for production architecture**. Retained only as a development/testing convenience, NOT a production runtime candidate.
- **Consequences:** Production builds never depend on user-installed Chromium. Any dev-convenience use must be explicitly marked as such in code/config.

### ADR-014: Trusted Command API allowlist pattern (F-8, V-5 RESOLVED)

- **Context:** During Phase 1 Windows validation, all six app-own Tauri commands (`get_app_info`, `navigate`, `get_browser_state`, `create_tab`, `close_tab`, `list_tabs`) were callable from the frontend via `window.__TAURI_INTERNALS__.invoke()` even though `capabilities/main.json` granted zero permissions for them. Investigation revealed: Tauri 2's ACL only auto-generates `allow-<command>` / `deny-<command>` permissions for **plugin** commands (via a plugin's `build.rs` `COMMANDS` list). App-own commands registered directly in `lib.rs` via `tauri::generate_handler!` are NOT auto-covered — they are callable by default with no permission grant.
- **Decision:** All app-own commands require a hand-written permission file under `src-tauri/permissions/`, declaring an `allow-<group-name>` permission with `commands.allow` listing the command names. The capability file (`capabilities/main.json`) grants `"allow-<group-name>"` for the window to use those commands. This is the standard pattern for ALL future app-own commands.
- **Consequences:** V-5 is RESOLVED. The trust model from ADR-009 is now concretely enforced: web content cannot invoke app commands unless explicitly granted. Verified in both directions on Windows (grant present → commands work; grant removed → commands rejected with specific error: `"X not allowed. Permissions associated with this command: allow-browser-shell-commands"`). Pattern documented in `docs/security/trust-model.md`.

### ADR-015: Custom window chrome via `decorations: false` (F-9)

- **Context:** Phase 1 needed a browser-like layout with horizontal tab strip + window controls in one row (matching Chrome/Edge/Opera/Brave/Vivaldi). Native OS title bar would double the chrome (OS title bar + custom WindowControls component both visible).
- **Decision:** Set `decorations: false` in `tauri.conf.json` `windows[0]`. Use custom `WindowControls` component at the right end of the tab strip row. Parent row carries `data-tauri-drag-region` for window dragging. Window control buttons stop propagation on `mousedown` to prevent drag conflicts.
- **Consequences:** Window is only draggable from the tab strip row (the `data-tauri-drag-region` element). If the drag region is missing or broken, the window cannot be moved (Alt+F4 still closes it). This is the standard pattern for all future browser windows.

### ADR-016: Centralized frontend invoke layer (F-10)

- **Context:** Components calling `invoke()` inline made the trust boundary (Section 10) spread across component files rather than auditable in one module. Adding commands in Phase 2+ would scatter more invoke calls further.
- **Decision:** All Tauri command calls from the frontend MUST go through `apps/browser/src/lib/tauri.ts`. Each function wraps a single `invoke()` call. No component imports `invoke()` directly. The barrel file `lib/index.ts` re-exports the API.
- **Consequences:** Frontend's callable surface is inspectable in one file — the trust boundary from Section 10 is auditable. Scales when Phase 2+ adds more commands: each new command adds one function to `lib/tauri.ts`.

### ADR-017: Phase-1 CSP trade-off — `'unsafe-inline'` on `style-src` (F-9, TEMPORARY DEBT)

- **Context:** Vite dev mode injects compiled CSS via inline `<style>` tags (not a linked `.css` file — that only happens in production builds). `style-src 'self'` blocked this, causing all Tailwind classes to exist in markup but never be applied — plain unstyled `<div>`s stacked block-level by default. Additionally, `index.html` had its own hardcoded `<meta http-equiv="Content-Security-Policy">` tag, also with `style-src 'self'`, completely out of sync with `tauri.conf.json`. A meta-tag CSP and a Tauri-injected CSP are both enforced simultaneously — the browser satisfies the strictest combination. Editing `tauri.conf.json` alone had no effect because the meta tag alone was sufficient to keep blocking inline styles.
- **Decision:** Added `'unsafe-inline'` to `style-src` in THREE places: `tauri.conf.json` `csp.style-src`, `tauri.conf.json` `devCsp.style-src`, and `index.html`'s meta CSP tag. This is TEMPORARY Phase-1 debt.
- **Consequences:** Acceptable for Phase 1 ONLY because no real web content is loaded (no WebView2 page embedding yet). **MUST be resolved before Phase 3** (real WebView2 content) — `'unsafe-inline'` becomes an actual XSS-relevant weakness when untrusted content is rendered. Resolution requires: (1) CSP single-source-of-truth (generate `index.html` meta tag from `tauri.conf.json` at build time), and (2) nonce-based or hash-based CSP instead of `'unsafe-inline'`. Deferred to v0.4.

### ADR-018: Monotonic ID counter pattern (F-7)

- **Context:** Tab ID generation derived from `tabs.len()` caused ID collisions after close+create cycles. Trace: create tab-1/2/3 (len=3), close tab-2 (len=2, tabs: tab-1, tab-3), create new tab → id = `"tab-3"` collides with existing tab-3.
- **Decision:** All entity IDs in the project use monotonic counters that NEVER decrease. Closed entity IDs are never reused. Pattern: `next_id: u64` field on the controller state, `checked_add(1)` on each create, NOT decremented on close.
- **Consequences:** IDs are unique for the lifetime of the controller. This pattern applies to all future entity types (windows, workspaces, profiles, sessions, etc.). The `"tab-N"` format is preserved for log/UI readability.

---

## End of PROJECT_CONTEXT.md v0.3
