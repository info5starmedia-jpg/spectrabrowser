# SpectraBrowser — Research Summary, Legal Map & Action Plan

**Date:** 2026-11-08
**Status:** v0.1.0 shipped (34/34 tests green) · daily upstream watch active · next milestone: P1 engine installer
**Repo:** https://github.com/info5starmedia-jpg/spectrabrowser (public, MIT)
**This document is the project bible.** Read it first when resuming work after a break.

---

## 1. Mission

Build the best multi-browser antidetect tool — best of the best, and stable. We take the strongest legally-available parts of the two repos that matter (Fury for detection, AliasMode for lifecycle/product), reimplement everything else ourselves, and keep the whole project clean enough to go commercial later if we want. brodev3 is excluded by user decision.

---

## 2. The Landscape — what we learned from the source

### Fury (furyteamtop/fury-antidetect-browser) — rating 9/10 — the detection teacher

- **Shape:** Rust workspace. `agent/` (ipc.rs 198 KB, relay.rs, store.rs — mega-files), `desktop/` (Tauri), `server/` (Linux team server), `core/patches/` (42 numbered C++ patches against Chromium), `core/verify/` (900+ claim release gate), `shared/` (persona schemas).
- **Engine:** own Chromium fork, now on **157.0.8103.39** (0.5.0/0.5.1, Nov 5). macOS + Windows only; no Linux browser; Widevine unsolved.
- **Why it wins detection:** the spoof lives **inside the engine's C++**, not in injected JavaScript. Detectors run in the page; there is no shim in the page to catch — no `toString` leaks, no getter-timing artifacts, no `defineProperty` tells.
- **Discipline:** every claim measured against real checkers (demo.fingerprint.com, fv.pro, pixelscan) and published, including failures ("macOS profiles flagged more on 157, cause open"). Release artifacts ship SHA256SUMS + PGP signatures.
- **Weaknesses:** solo dev (bus factor 1), almost no app-side tests (one sync_tests.rs), mega-files, slower product velocity, machine-translated UI.

### AliasMode (aliasmode/aliasmode) — rating 7.5/10 — the lifecycle/product teacher

- **Shape:** TypeScript/Bun monolith, flat root ~120 files. `launcher.ts` 257 KB (the brain), `cli.ts` 121 KB, `cloud-browser.ts` 116 KB, `ui.ts` 114 KB. Test files rival source size (`launcher.test.ts` 240 KB).
- **Engines:** own pinned builds, SHA-256-verified — Chromium 155 (build 5) + Firefox r14 (Camoufox-based, canvas keyed to profile seed).
- **Detection approach:** launch flags + JS injection (`deriveFingerprintFlags()`: FNV-1a seed → Mulberry32; storage-quota override, cores capped at host, per-profile `--fingerprint-noise`; deliberately **no** forced `--user-agent`). Good injection — but injection, shallower than Fury's by architecture.
- **Why it wins lifecycle:** obsessive correctness in the 1%-of-1% cases — launch-identity snapshots, post-stop cache-delete joins, named launch refusals, edit-profiles-while-running. Best test culture in the space. Ships daily (beta.73+).
- **Trajectory warning:** monetization appeared Nov 7–8 — affiliate "Services" page, batched purchase-event tracking. Free product becoming a storefront; engineering priorities may shift.

### brodev3 — EXCLUDED (user decision, 2026-11-08)

Pristine process (contract-first, ADRs, failure-first tests) but almost no product and a closed engine. We took the architectural lessons already; no code, no further tracking.

### Scorecard (weighted for "accounts don't get banned")

| Category | Fury | AliasMode |
|---|---|---|
| Anti-detection depth | **9.5** | 6.5 |
| Engine currency | **9** (157) | 7 (155) |
| Measurement rigor | **9.5** | 6 |
| Test coverage | 4 | **9.5** |
| Lifecycle correctness | 6 | **9.5** |
| Code structure | 5 | 4 |
| Product completeness | 7 | **8.5** |
| Velocity | 7 | **9.5** |
| License freedom for us | 8 (patches BSD) | **9** (Apache-2.0) |
| Bus factor | 3 (solo) | 5 |

---

## 3. Detection Knowledge Base (hard-won facts — ours to use)

Facts and measurements are not copyrightable. These are the crown jewels of the research:

1. **C++ patches beat JS injection, structurally.** Injection runs in the same room as the detector; engine patches leave nothing in the room.
2. **Canvas noise must be done in premultiplied space** (mimicking rasterizer coverage differences), deterministic per absolute pixel position, so `getImageData`, `toDataURL`, and `toBlob` agree **byte-for-byte** — detectors cross-check exactly these three. Solid rects must stay byte-identical across platforms (that's a probe).
3. **deviceMemory 16 GB is a flag on Chrome 157.** Fury's measurement (2026-11-05, fresh seeds, Windows): 16 GB scored 0.82–0.94 on demo.fingerprint.com, flagged 3/3; 8 GB scored 0.09–0.12, flagged 0/6. On 155 the same 16 scored 0.10. Cause unknown. → **Already ported: our generator caps Windows identities at 8 GB** (commit af39607, regression test included).
4. **Hardware coherence:** memory↔core-count must match real shipping machines (4 GB never claims 32 cores). Our `coresForMem` map implements this.
5. **Storage quota** must not look like incognito (quota override per profile).
6. **navigator.connection** must come from the persona, not the host (Fury patch 0122 — host network leaked behind mobile exits).
7. **Hidden fonts must be hidden at the platform cache**, not just the API layer (Fury patch 0051 — lookups bypassing `GetFontData()` found them).
8. **WebGPU:** a persona with no adapter must answer `null` — not show the host's GPU (an "apple" behind a Mali GPU is instant detection).
9. **Sensor realism:** real phones fire `deviceorientation` ~58–60×/4 s; naive fakes fire 140+. Jitter must be persona-driven.
10. **Do NOT force `--user-agent`** — UA/kernel desync is itself a signal (AliasMode's deliberate choice).
11. **Verify-suite traps:** detectors/APIs move between Chrome versions ("browser refused" vs "API moved" are different); unfocused windows suspend sensors in real Chrome too — don't flag it.
12. **Widevine is a three-line detection** (`com.widevine.alpha` refusal) and its blob must never be redistributed — open problem, even for Fury.

## 4. Lifecycle Knowledge Base (from AliasMode's scars — Apache-2.0, ours to reuse with attribution)

1. **Launch-identity snapshot:** the run row stores the identity the browser *started with*. Profile edits while running apply **at next start** ("Applies at next start" + Restart), never mutate the live browser. We currently block edits — adopting their model is queued (Phase 2).
2. **The engine's materialized values are the identity of record**, not what you intended (their Chromium-155 first-open core-count bug broke every Cloud session restore).
3. **Profile deletion must join engine teardown** — `stop()` returns before cache deletion finishes; deleting right after close fails. All delete paths wait (they use 30 s).
4. **Name every launch refusal** (`process_spawn`, `cdp_readiness`, `profile_changed`...) — bare "preflight failed" cost them real debugging time.
5. **A failed/crashed tab is not a dead browser** — don't kill a Firefox the user is looking at because one saved tab timed out.
6. **Cookie import rules:** Firefox silently drops cookies expiring >400 days out (cap them); one bad `__Secure-`/`__Host-` prefix makes Chromium refuse the whole batch (skip offenders); ms-vs-second expiry confusion; `cookies.txt` subdomain flag → domain cookie.
7. **Proxy credentials must be percent-encoded in URLs** (`# ? /` in passwords break naive URL building — Fury got bitten; our `proxyUrl()` already encodes).
8. **Updater/installer resilience:** gzip manifests, 30 s timeouts with retry, resume via Range requests, fail only after 60 s without bytes, verify signature on complete bytes. Their users are on slow links; ours will be too.
9. **Double-launch protection twice-enforced** (app-level mutex + database partial unique index) — already in our build; the 10-way-parallel-start test proves exactly one winner.

---

## 5. Legal Map — who we take from, and how to stay clean

### The tiered truth about Fury's license (from their NOTICE file, verified 2026-11-08)

| Part | License | What we do |
|---|---|---|
| `core/patches/` — **the engine patches (crown jewels)** | **BSD-3-Clause** (deliberately) | **Usable, even commercially.** Only duty: keep the copyright notice. Author's own words: "no copyleft obligation reaches somebody who only wanted a browser." |
| `shared/` — persona schemas + reference personas | **Apache-2.0** | Usable with attribution (NOTICE). |
| `agent/`, `server/`, `desktop/`, `tools/`, `core/build\|args\|verify/`, `deploy/`, `docs/` | **AGPL-3.0-or-later** | **Never copy a line.** Ideas/measurements only, clean-room reimplemented. |
| AliasMode (entire repo) | **Apache-2.0** | Usable with attribution (NOTICE + license headers retained). |
| brodev3 | MIT | **Excluded** — nothing taken, by user decision. |
| Camoufox | its own license | Binary user-installed, never auto-downloaded (supply-chain). |

### The rules that keep us safe

1. **No AGPL file ever enters our repo or our binaries' source tree.** Our MIT license survives only because of this. If we ever run Fury's *binary*, it's a separate child process we talk to over CDP — process separation, attribution, link to their source.
2. **Attribution is cheap insurance.** Maintain `NOTICE` in our repo: credit Fury (BSD patch series, Apache personas) and AliasMode (Apache-2.0 patterns/code) with copyright lines.
3. **Retained headers.** Any file we adapt from BSD/Apache sources keeps its original license header; we add ours below it.
4. **Clean-room for AGPL ideas:** read → understand → write our own from the idea. Facts, measurements, and techniques aren't copyrightable; expression is. (Precedent: the deviceMemory cap — ported as a measurement, zero code copied.)
5. **Trademarks:** never use "Fury", "AliasMode", "Chromium", or "Chrome" in our branding. Our engine, if built from the patch series, is "SpectraCore".
6. **Widevine:** never distribute a build containing the blob; it must be staged from a licensed local Chrome at build time only.
7. **Quiet research:** never fork/star/watch their repos with the user account (forks and stars are visible to owners; cloning/downloading ZIPs is not). Our repos carry no fork linkage to theirs.
8. **If we ever need more:** the Fury author is reachable and human — Bogdan Shapovalov, **@shapovalovbogdan on Telegram** (his stated preferred contact), or the GitHub issue tracker; site furybrowser.dev. For the BSD/Apache parts we already have permission; asking would be courtesy, not requirement.

---

## 6. What we already built (v0.1.0 — the foundation is DONE)

- **Contract-first control plane** — Zod contracts are the single source of truth (profiles, proxies, fingerprint identities, browser runs, AdsPower envelope).
- **Lifecycle core** — `BrowserLifecycleService`: double-launch guard (per-profile mutex + SQLite partial unique index), intent-before-action run rows, idempotent stop, boot-time reconciliation against live OS PIDs. 34/34 tests including a 10-way parallel start with exactly one winner.
- **AdsPower-compatible Local API** on 127.0.0.1:50325 — `/api/v1/browser/start|stop|active`, `/api/v1/profile/list|create`, envelope `{code, msg, data}`.
- **Engine port** — `BrowserRuntime` interface: FakeRuntime (deterministic tests) + Camoufox adapter (spawns real engine, user-installed binary, SIGTERM→SIGKILL escalation). **This port is the multi-browser abstraction** — new engines plug in without touching the control plane.
- **Fingerprint generator** — seed-deterministic identities, hardware-coherent, Windows memory capped at 8 GB (Fury 157 measurement).
- **Daily upstream watch** — cron job (12:00 PM) tracking Fury + AliasMode commits/releases, state in the task's state directory, ports queued to this document's backlog.

**User's manual to-do:** push `package-lock.json` + `.github/workflows/ci.yml` from the v0.1.0 zip (token here lacks workflow scope).

---

## 7. Action Plan — main focus, in order

**The main focus: the engine layer and stability. Nothing else matters until profiles launch real browsers reliably and pass checkers.**

### Phase 1 — Engine installer (P1, next 1–2 sessions)
Checksummed, pinned Camoufox download (`engines/` dir, SHA-256 verify, resume-via-Range per AliasMode's updater lessons, never silent-auto-download without consent). First real end-to-end: create profile → launch real Camoufox → AdsPower API returns working CDP endpoint → navigate → stop clean.

### Phase 2 — Launch-identity snapshot model (from AliasMode)
Store start-identity on the run row; allow profile edits while running; "applies at next start" semantics; named launch refusals (`process_spawn`, `cdp_readiness`, `profile_changed`); profile deletion joins engine teardown. This is the stability phase — AliasMode's scar tissue, transplanted before we grow the same wounds.

### Phase 3 — SpectraCore spike (the Fury route, decision point)
Evaluate building our own Chromium 157 core from Fury's **BSD-3-Clause** patch series. Questions to answer: build cost (~39 GB checkout, ~3 h), patch-series freshness vs Chromium stable, which patches matter most for our targets (0030 canvas, 0122 netinfo, 0051 fonts, 0032 WebGPU). Output: go/no-go + a second `BrowserRuntime` adapter skeleton. Camoufox stays the default engine regardless — SpectraCore would be the premium option behind the same port.
**Fallback if no-go:** keep Camoufox + adopt Camoufox's own C++-level config surface (it is also engine-patched, Firefox-based), and port Fury's *measurements* into our generator configs.

### Phase 4 — Detection CI
Automated checker runs per engine version (demo.fingerprint.com and friends), baseline captures like Fury's detect-suite, fail-on-regression. This is what turns "we think we're undetectable" into "we measured it" — the single biggest differentiator we can copy from Fury's culture.

### Phase 5 — Multi-engine profiles
Engine choice per profile in contracts (`engine: camoufox | spectracore`), persona↔engine coherence checks (a Windows-Chrome persona never launches on Firefox).

### Ongoing — the watch loop
Daily noon watch reports; port decisions appended here. The loop already paid for itself once (deviceMemory cap shipped within a day of the upstream measurement).

### Explicitly NOT doing (for now)
Mobile personas (parked with Fury's WebGPU-null and sensor-jitter lessons noted), cloud sync/team server (local-first by design), account-import pipelines (post-stability), UI beyond API (API-first is the differentiator).

---

## 8. Resume checklist (when coming back after weeks away)

1. Read this file.
2. Read the watch log in the automation task's state directory — everything upstream did while away.
3. Check repo state: latest commit on `main`, `npm run typecheck && npm test` green.
4. Current phase from §7; next action from the top of that phase.
5. Legal rules §5 apply to every commit, always.

---

*Compiled 2026-11-08 from the full research session: repo source reviews (Fury, AliasMode, brodev3), the v0.1.0 build, two upstream watch runs, and the license verification of Fury's NOTICE file.*
