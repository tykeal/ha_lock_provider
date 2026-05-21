<!--
  Sync Impact Report
  ==================================================
  Version change: 1.0.0 → 1.0.1
  Change type: PATCH — clarifications and missing-coverage
  additions; no principle removed or materially redefined.

  Modified principles / sections (per rubber-duck review):
    - Principle IV (SemVer & Dual-Consumer Release
      Discipline): dual-consumer validation now uniformly
      required for every release tag (including PATCH and
      phase releases), with a narrow bootstrap-only
      exemption. Added explicit definition of the public API
      surface (documented `__all__` exports + documented
      top-level modules; underscore-prefixed symbols are
      private; `py.typed` marker required).
    - Principle V (Per-Provider Optional Extras): added
      dependency-version-policy bullet requiring explicit
      lower and upper bounds for HA core and vendor SDK
      extras, with changelog and dual-consumer validation
      on bound changes.
    - Principle VII (Testability Without Live Platforms):
      corrected "extras ship a test harness" wording —
      replaced with "repository MUST include provider-
      specific fake/stub fixtures under `tests/`."
    - Principle IX (Test-Driven Development): reworded the
      lead bullet to be reviewable from diffs ("every
      behavior change MUST include tests that would fail
      against the previous implementation") while keeping
      TDD NON-NEGOTIABLE.
    - Additional Constraints → Security: added a redaction
      bullet covering PINs, access codes, credential
      payloads, and raw service-call data; tests MUST cover
      redaction for provider error context.
    - Development Workflow & Quality Gates: removed the
      "non-PATCH" looseness — dual-consumer validation
      applies to every release.
    - Governance: added an early-release dual-maintainer
      gate (Phase 2 onward requires approval from at least
      one KM and one LCM maintainer on release-affecting
      PRs; release ownership documented before first
      release); org-hosting decision MAY remain deferred
      until Phase 3.

  Added sections: None (PATCH).

  Removed sections: None.

  Templates reviewed (all under `.specify/templates/`):
    - constitution-template.md ✅ reviewed, no change
    - plan-template.md ✅ reviewed, no change
    - spec-template.md ✅ reviewed, no change
    - tasks-template.md ✅ reviewed, no change
    - checklist-template.md ✅ reviewed, no change

  Deferred items / Follow-up TODOs:
    - Org-hosting model (neutral GitHub org vs. hosting
      under one consumer's org) remains deferred and MUST
      be resolved before Phase 3 (first non-utility
      extraction) begins.
    - Concrete branch-protection and code-owner mapping for
      the dual-maintainer gate MUST be ratified before
      Phase 2's first tagged release.
  ==================================================
-->

# ha_lock_provider Constitution

## Core Principles

### I. Library Scope Discipline (NON-NEGOTIABLE)

- `ha_lock_provider` is a **thin transport library** for the
  platform-translation layer between per-slot user-code
  semantics and vendor lock platforms (Akuvox, Schlage, Z-Wave
  JS, Matter, ZHA, Zigbee2MQTT, Virtual). It is jointly
  consumed by the Keymaster (KM) and Lock Code Manager (LCM)
  Home Assistant integrations and MUST remain neutral toward
  both.
- The library MUST NOT import from `keymaster`,
  `lock_code_manager`, or any consumer-specific package.
- The library MUST NOT assume or impose a coordinator shape,
  base-class shape, or provider lifecycle model. KM and LCM
  retain their own coordinators, slot-state semantics, and
  lifecycle management.
- The library MUST NOT define or enforce slot-state semantics
  (e.g., `Synced`, `SlotCode.EMPTY`, `SlotCode.UNREADABLE_CODE`
  sentinels are consumer policy and remain in the consuming
  project).
- The library MUST NOT fire project-specific events
  (e.g., `keymaster_lock_state_changed`, `lcm_*` events).
  Provider clients expose primitive event-subscription
  callbacks that consumers translate into their own event
  vocabulary.
- The library MUST NOT enforce a slot-numbering scheme or
  managed-slot range. Range-aware operations take the managed
  range as a caller-supplied parameter.
- Provider clients MUST return primitive types (plain
  `dataclass`, `list`, `dict`) only. Consumer-specific
  sentinel enums and wrapper types are PROHIBITED in library
  return values.
- The library MAY import from `homeassistant.core` and call
  `hass.services.async_call(...)` — it is an HA-aware library,
  not an HA-agnostic one. Vendor SDK imports (e.g.,
  `zwave_js_server`, `matter_server`) MUST be optional
  (see Principle V).

**Rationale**: KM and LCM have diverged substantially at the
orchestration layer (coordinator model, return shapes,
lifecycle gating, event firing) while their platform
translation code remains structurally similar. The shared
engineering value lives in the transport layer; merging
orchestration would force one project to adopt the other's
architecture. A thin, neutral transport library captures the
shared value (PR #616 multi-firmware Akuvox detection, BE469
clear-verification workaround, Schlage add-before-delete with
rollback, Z-Wave User Code CC version gating, activity-map
translation) without imposing architectural decisions on
either consumer.

### II. Backward-Compatible Tag Formats (NON-NEGOTIABLE)

- The library exposes `TagFormat(prefix: str)` parameterized
  over the tag prefix. Both `[KM:N]` and `[LCM:N]` literals
  MUST remain valid and functioning at all times within a
  major version.
- The exact existing regex semantics —
  `re.compile(r"^\[KM:(\d+)\]\s*(.*)")` and the symmetric
  `[LCM:N]` form — MUST be preserved byte-for-byte.
  Equivalent semantics for any future prefix MUST be
  produced by the same `TagFormat` machinery, not by
  consumer-side workarounds.
- The exact existing format string —
  `f"[{prefix}:{slot_num}] {base}"` (with the single space
  after `]` and the trimming behavior the current
  implementations exhibit) — MUST be preserved
  byte-for-byte.
- Any change to tag regex, format string, whitespace handling,
  or parse output MUST be treated as a MAJOR breaking change
  (Principle IV) with a documented migration path, and MUST
  NOT silently re-tag existing user-facing device records.
- Library releases MUST include explicit regression tests that
  exercise both the `[KM:N]` and `[LCM:N]` round-trips against
  representative production-style names (with whitespace,
  empty friendly-names, unicode names, and malformed inputs).

**Rationale**: Both KM and LCM have users with devices in
service whose lock-side user records and PIN-code entries are
already tagged `[KM:N]` or `[LCM:N]`. Drift in regex, format
string, or whitespace would re-tag every user on every
existing install on upgrade — an unacceptable user-visible
regression. This principle exists precisely because the
library's reason for existing is to be a single source of
truth for code that MUST NOT silently change behavior.

### III. Async-Only Public API (NON-NEGOTIABLE)

- The library's public API MUST be `async`-only. Synchronous
  public entry points are PROHIBITED.
- Blocking the Home Assistant event loop is PROHIBITED.
  Blocking work (vendor SDK calls that are themselves
  synchronous, file I/O, CPU-bound work) MUST be offloaded
  via `hass.async_add_executor_job(...)` or an equivalent
  executor pattern accepted from the caller.
- Internal helpers MAY be synchronous when they are pure (no
  I/O, no blocking) — pure utilities such as tag parsing,
  rate-limiter state transitions, and activity-map lookups
  are exempt from the async requirement.
- Tests MAY use synchronous fixtures and helpers; production
  surface area exposed to consumers MUST be async.

**Rationale**: Both consumers are Home Assistant
integrations and are async-only inside HA. A synchronous
public API would either block the HA event loop (unsafe) or
force every consumer to wrap every call in
`async_add_executor_job` (boilerplate that defeats the point
of a shared library).

### IV. SemVer & Dual-Consumer Release Discipline (NON-NEGOTIABLE)

- The library MUST follow Semantic Versioning 2.0.0.
- Within a single major version, the public API surface MUST
  remain backward compatible. Breaking changes MUST batch
  into a new major release.
- Every breaking change MUST land first as a MINOR release
  that adds the replacement API and marks the old API
  `DeprecationWarning`. The deprecation window MUST be at
  least one MINOR release before the next MAJOR removes the
  old API. Removing or redefining behavior without prior
  deprecation is PROHIBITED.
- Every release tag (MAJOR, MINOR, PATCH, and any phase
  release) MUST be validated against **both** KM's and LCM's
  test suites before the tag is pushed. A release that
  breaks either consumer is a release defect and MUST be
  yanked or superseded by an immediate fix release. The
  sole exemption is a release explicitly marked
  pre-consumer / bootstrap-only (e.g., the Phase 1
  placeholder PyPI release before either consumer imports
  the package); such releases MUST carry the exemption
  marker in the changelog entry.
- **Public API surface**: the SemVer-stable public API
  consists only of documented symbols exported from package
  `__all__` declarations and documented top-level modules.
  Underscore-prefixed modules and underscore-prefixed
  symbols are private and are NOT SemVer-stable; consumers
  rely on them at their own risk. The package MUST ship a
  `py.typed` marker so downstream `mypy` runs see the
  library as typed.
- Initial consumer pinning policy: KM and LCM SHOULD pin to
  `~=X.Y` (compatible-release) initially and MAY broaden to
  `>=X,<X+1` once joint-maintenance cadence is established.
- Each release MUST include a changelog entry that
  enumerates: added APIs, deprecated APIs, removed APIs
  (only in MAJORs), and any provider-extras
  dependency-version-bound changes (see Principle V).

**Rationale**: The library has two production consumers on
independent release cadences. A release that breaks either
consumer cascades into urgent fix work in two projects
simultaneously. Dual-consumer validation before tagging — and
a real deprecation window before removals — is the only
sustainable cadence for joint maintenance.

### V. Per-Provider Optional Extras

- Vendor SDK and platform-specific dependencies (e.g.,
  `zwave-js-server-python`, `matter-server`, `python-matter-server`,
  `zigpy`, `paho-mqtt`, and similar) MUST be declared as
  **optional extras** in `pyproject.toml`, not as required
  dependencies.
- The base install (`pip install ha_lock_provider`) MUST be
  lightweight and MUST NOT pull in any vendor SDK that is not
  required by every consumer.
- Each provider package MUST declare its own extra
  (e.g., `ha_lock_provider[akuvox]`,
  `ha_lock_provider[zwave_js]`, `ha_lock_provider[matter]`,
  `ha_lock_provider[zha]`, `ha_lock_provider[zigbee2mqtt]`,
  `ha_lock_provider[virtual]`).
- Importing a provider module without its extras installed
  MUST raise an actionable `ImportError` (or equivalent
  diagnostic) naming the missing extra and the install
  command.
- An `all` extra MAY be provided for convenience; it MUST NOT
  be the default install.
- **Dependency version policy**: Home Assistant core and
  every optional vendor SDK extra MUST declare explicit
  lower and upper version bounds compatible with the
  supported HA / Python matrix. Unbounded
  (`>=X`-only) declarations on HA core or vendor SDKs are
  PROHIBITED. Dependency-bound changes (raising a lower
  bound, raising an upper cap, dropping a Python or HA
  minor) MUST be listed in the CHANGELOG and validated
  against both KM's and LCM's test suites before release
  (see Principle IV).

**Rationale**: Consumers should not be forced to install
Matter SDKs to use Akuvox, or Zigbee stacks to use Schlage.
Per-provider extras keep the base install lightweight, keep
the dependency graph small for embedded HA installs (e.g.,
Home Assistant Yellow, HA Green, Raspberry Pi), and let the
library evolve provider-specific dependencies without
forcing global upgrades on every consumer.

### VI. Phased Delivery

- Extraction MUST proceed in defined, independently shippable
  phases. The canonical phase ordering is:
  1. **Bootstrap** — repository, packaging, CI, empty
     package skeleton, placeholder PyPI release.
  2. **Pure utilities** — `exceptions`, `tags`
     (parameterized prefix), `rate_limiter`, `models.UserCode`.
  3. **Akuvox** — smallest real client; PR #616
     multi-firmware fix lifted verbatim.
  4. **Schlage** — add-before-delete with rollback and
     eventual-consistency / 409 handling.
  5. **Z-Wave JS** — staged as (5a) activity tables +
     translator, (5b) connection + CRUD ops, (5c) push
     subscription with primitive callback.
  6. **LCM-only providers** — Matter, ZHA, Zigbee2MQTT,
     Virtual; each provider is its own additive sub-phase.
- Each phase MUST deliver an independently testable and
  independently releasable increment. A phase MUST NOT depend
  on code that lands in a later phase.
- Each phase MUST conclude with a checkpoint: all CI tests
  green, library release tagged (where appropriate), and
  the release validated against **both** KM's and LCM's
  test suites per Principle IV (unless the release carries
  the bootstrap-only exemption marker).
- Phase boundaries and exit criteria MUST be documented in
  the implementation plan (`plan.md`) and task list
  (`tasks.md`).
- Tests for a phase MAY be written during that phase (not all
  up front). Unit-level TDD (Principle IX) MUST NOT be
  deferred under any circumstance.

**Rationale**: A big-bang extraction across all providers
would amplify risk and stall both consumers simultaneously.
Phased delivery isolates risk, proves the joint-maintenance
workflow on low-controversy surfaces (utilities) before
touching production-critical paths (Z-Wave JS), and lets the
library deliver early value (shared `tags` and `exceptions`)
without committing to the full extraction inventory.

### VII. Testability Without Live Platforms (NON-NEGOTIABLE)

- Every provider client MUST be fully testable without a
  live vendor connection. The unit test suite MUST NOT
  require: a real Z-Wave network, a real Schlage cloud
  account, a real Akuvox device, a real Matter fabric, a
  real Zigbee coordinator, or a real MQTT broker.
- `HomeAssistant` service calls and vendor SDK calls MUST be
  mockable via dependency injection, fixture-based patterns,
  or service-stub harnesses. Hard-coded global imports that
  cannot be patched at test time are PROHIBITED.
- The repository MUST include provider-specific fake / stub
  fixtures under `tests/` (fake driver, fake cluster, fake
  MQTT client, etc.) sufficient to exercise each client's
  primary code paths without live platforms and without
  requiring unrelated extras to be installed.
- Tests that require network access, real credentials, or
  real hardware MUST be explicitly marked and excluded from
  the default CI run.

**Rationale**: A library that cannot be tested without live
infrastructure cannot be maintained at the cadence two
consumers require. The Z-Wave JS, Schlage, and Akuvox layers
exist precisely to encode hard-won workarounds (BE469 clear
verification, Schlage 409 eventual consistency, Akuvox
multi-firmware `_is_local_user`); regression tests against
those workarounds MUST be runnable on any contributor's
laptop and in CI without external dependencies.

### VIII. Code Quality Gates (NON-NEGOTIABLE)

- All source code MUST pass configured linting and static
  analysis checks (ruff, mypy, interrogate) with zero errors
  or warnings.
- Cyclomatic complexity MUST NOT exceed 10 per function
  (ruff rule `C901`). This limit MUST be enforced in the
  project's ruff configuration.
- Every function and class MUST include a docstring
  describing its purpose, parameters, return values, and
  raised exceptions.
- Interrogate MUST enforce 100% docstring coverage; commits
  that reduce coverage are PROHIBITED.
- Type annotations MUST be present on all public function
  signatures. `mypy --strict` (or the project's documented
  equivalent ruleset) MUST pass on the `src/` tree.
- All files MUST be REUSE-compliant, either via inline SPDX
  headers or via `REUSE.toml` annotations. The reuse-tool
  pre-commit hook enforces compliance; files not properly
  covered MUST NOT be committed.

**Rationale**: The library is consumed by two production
integrations. Quality gates that catch defects, type errors,
and licensing gaps before they reach a consumer release are
cheaper than chasing them across two downstream projects.
Strict static analysis is the lowest-cost form of dual-
consumer protection.

### IX. Test-Driven Development (NON-NEGOTIABLE)

- **Code-level TDD is mandatory.** Every behavior change
  MUST include tests that would fail against the previous
  implementation, or that characterize extracted behavior
  before migration. When branch history is reviewed, test
  commits SHOULD precede implementation commits within a
  phase. The Red-Green-Refactor cycle remains the working
  discipline:
  1. Write a failing test that defines the desired behavior.
  2. Implement the minimum code required to make the test
     pass.
  3. Refactor while keeping all tests green.
- Unit-level TDD discipline MUST NOT be deferred under any
  circumstance. Squash/reorder MAY collapse test and
  implementation commits, but the final branch diff MUST
  still contain tests that exercise the new or changed
  behavior.
- Higher-level tests (cross-provider integration tests,
  long-running soak tests, dual-consumer validation runs)
  MAY be deferred to the phase where their prerequisites
  land — these tests are scope-bound to the phase that owns
  the relevant code surface.
- CI tests MUST pass before any manual or exploratory
  testing is performed. Manual testing without green CI is
  PROHIBITED.
- Test coverage MUST be maintained or increased with every
  change; coverage regressions MUST be justified and
  approved.
- Regression tests covering verbatim-lifted workarounds
  (PR #616 Akuvox multi-firmware detection, BE469 Z-Wave
  clear verification, Schlage add-before-delete rollback)
  MUST exist before that workaround can be considered
  "extracted" into the library.

**Rationale**: TDD at the unit level is what makes the
verbatim-lift safety guarantees of Principle II (backward-
compatible tag formats) and the workaround-preservation
guarantees of the extraction proposal actually verifiable.
Without a failing-first test, "lifted verbatim" is a claim
nobody can audit.

### X. Atomic Commits & Compliance (NON-NEGOTIABLE)

- Every commit MUST represent exactly one logical change
  (one feature, one fix, or one refactor).
- Each commit MUST compile and run successfully; broken
  intermediate states are PROHIBITED.
- Commit messages MUST follow this repository's
  Conventional Commit-style format with capitalized types as
  defined in `AGENTS.md` and `.gitlint` (types: `Fix`,
  `Feat`, `Chore`, `Docs`, `Style`, `Refactor`, `Perf`,
  `Test`, `Revert`, `CI`, `Build`).
- Subject lines MUST be ≤50 characters and MUST NOT end with
  a period. Body lines MUST wrap at ≤72 characters (URL
  lines are exempt per the configured `ignore-by-body`
  rule). Subjects MUST use the imperative mood.
- Large features MUST be broken into multiple atomic commits.
  Mixing unrelated changes in a single commit is PROHIBITED.
- Task tracking document updates (e.g., `tasks.md`) MUST be
  committed separately from the code they track.
- Pre-commit hooks MUST pass on every commit. Bypassing
  hooks with `--no-verify` is PROHIBITED under all
  circumstances. The failure-recovery protocol is: fix the
  issue, `git add` the fix, retry the commit as if the
  prior attempt never happened — do NOT use `git reset`
  after a failed commit attempt.
- Any commit that introduces new files MUST include the
  appropriate SPDX license header for those files, or the
  files MUST be covered by an existing `REUSE.toml`
  annotation. Every commit MUST carry a DCO sign-off
  (`git commit -s`).

**Rationale**: Atomic commits enable clean git history for
debugging, easy reversion of specific changes without
collateral impact, focused code review, and bisect-friendly
debugging. Pre-commit hooks are the first line of defense
against defects and licensing violations; bypassing them
creates risk for the whole codebase and for every downstream
consumer.

### XI. Agent Co-Authorship & DCO Requirements (NON-NEGOTIABLE)

- All commits authored or co-authored by AI agents MUST
  include proper attribution and sign-off.
- Every agent-assisted commit MUST include a
  `Co-authored-by` trailer identifying the AI agent (see the
  `AGENTS.md` table for the canonical email mapping for
  Claude, ChatGPT, Gemini, and Copilot).
- Every commit MUST carry a DCO sign-off added via
  `git commit -s`:
  ```
  Signed-off-by: Andrew Grimberg <tykeal@bardicgrove.org>
  ```
- The `Co-authored-by` trailer goes in the commit message
  body; `git commit -s` appends the `Signed-off-by` line
  last, automatically.

**Rationale**: Transparency in authorship is required for
Developer Certificate of Origin compliance, audit trails for
code provenance (especially important for verbatim-lift
claims from KM and LCM upstream code), and maintaining trust
when two upstream consumer projects merge library updates.

## Additional Constraints

- **Language & Runtime**: Python 3.x with full type
  annotation coverage enforced by mypy. The supported Python
  version matrix MUST match the versions Home Assistant
  itself supports at the time of each release.
- **Package Layout**: The project uses a `src/` + `tests/`
  layout. The package source lives under `src/ha_lock_provider/`
  and tests under `tests/`. The library is NOT a Home
  Assistant custom component — it is a pure Python library
  distributed via PyPI.
- **Dependency Management**: Dependencies MUST be managed
  via `uv`. A locked dependency file (`uv.lock`) MUST be
  committed to the repository.
- **Home Assistant Coupling**: The library MAY import
  `homeassistant.core.HomeAssistant` and call
  `hass.services.async_call(...)`. It MUST NOT import from
  KM, LCM, or any other consumer-specific package. See
  Principle I.
- **Vendor SDK Coupling**: Vendor SDK imports MUST be
  guarded by per-provider extras (Principle V). The base
  install MUST NOT pull in vendor SDKs.
- **Backward Compatibility**: The `[KM:N]` and `[LCM:N]` tag
  formats are user-visible artifacts on lock devices in
  production. Changes that alter their byte-level
  representation are MAJOR breaking changes and require the
  full deprecation window (Principle IV).
- **License Compliance**: The project follows the REUSE
  specification. Every file MUST be covered by an SPDX
  header or an entry in `REUSE.toml`. Apache-2.0 is the
  default project license; the `.specify/**` tree is
  MIT-licensed per the upstream Spec Kit (see `REUSE.toml`).
- **Security**: Secrets, API tokens, and vendor credentials
  MUST NEVER be committed to source control. Tests that
  require credentials MUST receive them via environment
  variables or test fixtures, never via committed files.
- **PIN / access-code redaction**: PINs, access codes,
  credential payloads, and raw service-call data containing
  codes MUST NOT be logged or included unredacted in
  exception messages, exception `__cause__` chains, or
  diagnostic context surfaced to callers. Provider clients
  MUST redact code material before raising library
  exceptions or emitting log records. Tests MUST cover
  redaction for provider errors that surface request /
  response context (e.g., Schlage 409 already-exists,
  Z-Wave SET_USER_CODE rejection, Akuvox add/modify
  failures).

## Development Workflow & Quality Gates

1. **Write tests** for the current phase or story (TDD red
   phase).
2. **Implement** the minimum code required to make those
   tests pass (TDD green).
3. **Refactor** while keeping all tests green.
4. **Run linting & type checks** locally:
   `uv run ruff check src/ tests/` and `uv run mypy src/`.
5. **Run the test suite** locally:
   `uv run pytest tests/ -x -q`.
6. **Stage and commit atomically** with sign-off, the
   appropriate `Co-authored-by` trailer (per `AGENTS.md`),
   and SPDX headers on any new files.
7. **Pre-commit hooks** run automatically — fix any
   failures and re-commit (do NOT reset; do NOT bypass).
8. **CI pipeline** MUST pass. No manual or exploratory
   testing is permitted until CI is green.
9. **Dual-consumer validation** before every release tag
   (MAJOR, MINOR, PATCH, and phase releases): the candidate
   library version MUST be exercised against both KM's and
   LCM's test suites. A release that breaks either consumer
   MUST NOT be tagged. The only exemption is a release
   explicitly marked pre-consumer / bootstrap-only in the
   changelog (per Principle IV).
10. **Pull request review** MUST verify constitutional
    compliance, atomic commit structure, proper licensing
    headers, agent co-authorship (when applicable), and
    library-scope discipline (Principle I) — no consumer-
    specific imports, no coordinator-shape assumptions, no
    project-specific event firing.
11. **Manual validation** may proceed only after CI confirms
    all automated checks pass.

## Governance

- This constitution supersedes all other development
  practices for the `ha_lock_provider` project. In case of
  conflict, this document prevails.
- The library is **jointly maintained** by Keymaster (KM)
  and Lock Code Manager (LCM) maintainers. Neither project's
  coordinator model, base-class shape, or release cadence
  takes precedence in library design decisions.
- **Early-release dual-maintainer gate**: before the first
  non-placeholder release (Phase 2 onward), release-
  affecting PRs MUST receive approval from at least one KM
  maintainer and one LCM maintainer. Release ownership
  (who tags releases, who publishes to PyPI, who holds the
  publish credentials) MUST be documented before that first
  release is tagged. Org-hosting (neutral GitHub org vs.
  hosting under one consumer's org) MAY remain deferred
  until Phase 3.
- The remaining concrete governance details — full code-
  owner mapping across all provider packages, branch-
  protection ruleset, and any rotating-release-manager
  policy — MUST be ratified before Phase 3 (first
  non-utility extraction) begins.
- Amendments MUST be documented with a version bump,
  rationale, and migration plan if existing code or
  consumer integrations are affected.
- Version increments follow semantic versioning:
  - **MAJOR**: Backward-incompatible principle removals or
    redefinitions.
  - **MINOR**: New principles or materially expanded
    guidance.
  - **PATCH**: Clarifications, wording, or non-semantic
    refinements.
- All pull requests and code reviews MUST verify compliance
  with these principles. Non-compliance MUST block merge.
- Amendment history MUST be preserved in the Sync Impact
  Report comment at the top of this file.
- All `.specify/templates/*.md` files MUST be reviewed for
  consistency when amendments are made.
- Use `AGENTS.md` for runtime development guidance (commit
  format, co-authorship trailers, pre-commit failure
  recovery, worktree placement) that supplements this
  constitution.

**Version**: 1.0.1 | **Ratified**: 2026-05-21 | **Last Amended**: 2026-05-21
