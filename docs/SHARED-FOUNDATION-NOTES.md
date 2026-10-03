# Shared Sandbox Foundation — Future Implementation Notes

**Status:** Discovery and handoff notes only; no shared runtime has been implemented

**Candidate consumers:**

- <https://github.com/digigrant/devenv>
- <https://github.com/digigrant/study-room>

**Purpose:** Give a future agent an evidence-based starting point for extracting reusable infrastructure for self-contained, reproducible Docker Sandbox environments without prematurely coupling the two existing projects.

## 1. Instructions for a future agent

1. Treat `devenv` and `study-room` as independent products and sources of truth.
2. Read the current specifications and implementation before proposing extraction. These notes describe candidate seams, not guaranteed current APIs.
3. Do not refactor either consumer merely to make an abstraction look cleaner. Extract only code already proven substantially equivalent in at least two implementations.
4. Never make a trusted host command source or execute code from a sandbox-writable workspace.
5. Never make a reproducible build follow an unpinned `main`, `latest`, or mutable release URL.
6. Never place an Infisical credential, provider refresh token, provider access token, GitHub PAT, Obsidian credential, or real secret in this repository.
7. Work on branches, push as `gej-machine`, and open pull requests. Do not merge without the owner's explicit instruction.
8. Preserve each consumer's security and lifecycle decisions. A shared helper must not silently broaden permissions, mounts, network access, persistence, or secret scope.
9. Prefer a small versioned library plus conformance fixtures over a framework that owns each product's entrypoint.
10. Stop and ask before changing an existing host's global Docker Sandbox policy, keyring schema, Infisical role, or secret registration.

## 2. Why a foundation may be worthwhile

The two projects have different user experiences but repeat several trusted-host and reproducibility problems.

| Concern | `devenv` evidence | `study-room` evidence | Extraction outlook |
|---|---|---|---|
| Host support | Linux `sbx` on WSL2 and native Ubuntu; no hard-coded home | WSL2 and native Ubuntu Desktop 24.04 from V1 | Strong candidate |
| Host prerequisite checks | `devenv doctor`, `host-prepare`, documented host verification | Interactive check-plan-confirm-install requirement | Strong candidate after APIs are compared |
| Trusted checkout isolation | Host checkout is outside the writable workspace | Host checkout must not be mounted into the Pi sandbox | Strong invariant and reusable checks |
| Secret Service keyring | `devenv-infisical` namespace and fake-keyring tests | Separate `study-room` namespace required | Strong candidate with per-consumer namespace |
| Infisical machine identity | Existing `sbx-host` Universal Auth identity | Reuse `sbx-host`, but under an independent path/schema | Shared transport; authorization differs |
| Docker proxy placeholders | GitHub and Claude command-backed/custom secrets | GitHub, OpenAI, and Kimi placeholders | Strong candidate after rotating OAuth is proven |
| Artifact pins | `versions.env`, checksums, explicit bump | Lock manifest, checksums, explicit bump | Strong candidate |
| Drift warnings | `devenv check` and entry/status warnings | Cached entry warning and `check-updates` | Candidate contract; UI remains product-specific |
| Test fakes | Fake keyring, Infisical, sbx, kit simulation | Same fakes needed plus rotating OAuth | Strong candidate |
| Host verification | Detailed `docs/HOST-VERIFY.md` | WSL2/native/GUI acceptance required | Shared checklist format, not identical checks |
| Network policy | Balanced baseline and kit-scoped allow rules | Balanced default plus optional sandbox-scoped web mode | Share policy generation/inspection, not policy content |

The strongest initial opportunity is trusted-host plumbing and tests—not agent configuration, application entry behavior, or one universal sandbox kit.

## 3. Boundaries that should remain product-specific

Do not extract these merely because both products use Docker Sandboxes:

- agent harness and orchestration;
- Pi versus Claude/Firstmate configuration;
- tmux versus herdr;
- workspace contents and mount layout;
- Obsidian Desktop, vault, and Sync handling;
- Learn integration;
- Android SDK/emulator support;
- Tailscale and Magic Conch;
- persistence policy;
- entrypoint UI;
- provider selection and model settings;
- researcher tools and lifecycle;
- exact network allowlists;
- app-specific status lines;
- app-specific doctor checks;
- OAuth provider protocol details.

`devenv` currently uses a v2 kit extending Claude. Study Room will use a Pi-focused kit/environment. Do not force both through one generic kit until a stable common fragment is demonstrated.

## 4. Candidate shared modules

Names below are illustrative. A future design may use shell, a small compiled binary, or another minimal runtime, but it must preserve the contracts.

### 4.1 Host capability probes

Potential module: `host/capabilities`

Return structured results for:

- OS and Ubuntu release;
- architecture;
- native Linux versus WSL2;
- systemd and user-session availability;
- WSLg, Wayland, and X11;
- KVM device, access, and group membership;
- Docker and Docker Sandbox presence/version;
- Docker Sandbox daemon/login readiness;
- Docker policy initialization;
- Secret Service and keyring availability;
- available disk space;
- whether the trusted checkout intersects a sandbox workspace.

Requirements:

- probes are read-only;
- machine-readable output is separate from product-specific human rendering;
- a missing prerequisite is data, not an immediate install;
- installation plans and consent remain with the consumer;
- paths use XDG/home discovery and never assume `/home/gejoy`, `/home/grant`, or a Windows drive.

### 4.2 Trusted-checkout and workspace safety

Potential module: `host/workspace-safety`

Reusable checks should detect:

- a trusted infrastructure checkout inside a sandbox workspace;
- a sandbox workspace containing trusted host scripts outside the one approved data directory;
- lifecycle or secret commands resolving through a writable mount;
- symlink-based escapes or overlap;
- unsafe `git clean -x` implications;
- environment files placed inside a mounted workspace when the product requires them outside it.

The module should accept explicit approved paths. It must not choose each product's workspace design.

### 4.3 Keyring access

Potential module: `secrets/keyring`

Proposed contract:

- caller supplies a service/collection namespace;
- lookup is noninteractive by default;
- setup may store values through stdin after explicit user interaction;
- diagnostics go to stderr;
- secret values go only to stdout when the caller explicitly requests a secret-output operation;
- locked, missing, unavailable, and prompt-required states are distinguishable;
- no value appears in process arguments, logs, shell history, or temporary files;
- tests run against a real ephemeral keyring where possible and fakes otherwise.

Each consumer keeps an independent namespace. Foundation must not collapse `devenv-infisical` and `study-room` into one key namespace.

### 4.4 Infisical Universal Auth transport

Potential module: `secrets/infisical`

Reusable operations may include:

- log in using a client ID/secret read from the keyring;
- obtain a short-lived access token without persisting it;
- read one allowed secret/path;
- write or atomically replace one allowed structured secret/path;
- redact errors;
- enforce timeouts;
- place request bodies and authorization headers on stdin or in memory, never command arguments;
- expose a fake transport for tests.

The caller must provide project/environment/path and an operation allowlist. The foundation must not contain real IDs or names.

#### Permission mismatch that must be resolved first

`devenv` documents `sbx-host` as a project-level built-in Viewer and only reads static values. Study Room needs narrowly scoped write authority for per-host rotating OAuth records under a path such as `/study-room/oauth/`.

Do not generalize this by granting project-wide write access. Before extracting write helpers, prove that the deployed Infisical plan can enforce the required folder/path scope. If it cannot, stop for a design decision: a separate identity/project or another reviewed mechanism may be required. This is a security gate, not an implementation inconvenience.

### 4.5 Structured rotating-secret records

Potential module only after Study Room proves it: `secrets/rotating-record`

Possible responsibilities:

- one opaque structured record per provider and host installation ID;
- host-local lock with bounded wait and stale-lock recovery;
- compare/reload before write;
- atomic whole-record replacement;
- preservation of rotated refresh tokens;
- expiry/skew calculations;
- no token-bearing logs;
- crash tests around every write boundary.

Do not force `devenv`'s static Claude setup-token into this abstraction. Static read and rotating OAuth state are separate use cases that may share only transport and redaction helpers.

### 4.6 Docker Sandbox secret registration

Potential module: `sbx/secrets`

Candidate operations:

- inspect sandbox-scoped service/custom secret metadata without values;
- register/update a service secret backed by an absolute trusted command;
- register/update a custom env placeholder for one or more exact hosts;
- reuse the placeholder already held by sbx;
- set an explicit refresh mode/duration;
- remove only a named sandbox-scoped secret;
- verify expected scope, hosts, env name, source command, and refresh;
- render a redacted plan before changes.

Invariants:

- command text contains only trusted absolute paths, record names, and non-secret options;
- no command text contains a secret;
- global secrets are never changed implicitly;
- operations are idempotent;
- sandbox scope is mandatory unless a caller explicitly requests and confirms otherwise;
- placeholder values are non-secret but are not confused with real values;
- bindings and destination domains remain product-owned configuration.

The module should account for observed sbx behavior: custom secrets are keyed by env/placeholder within a scope, placeholders must be reused on update, and custom-secret and service-secret refresh defaults differ.

### 4.7 Docker Sandbox policy inspection

Potential module: `sbx/policy`

Useful shared behavior:

- inspect global and sandbox-scoped policy;
- detect uninitialized policy;
- initialize a baseline only after approval;
- compare intended sandbox-scoped rules to effective rules;
- add/remove only project-owned rules;
- report broad/global rules without silently deleting them;
- generate fixtures for `allow-all`, `balanced`, and `deny-all` baselines.

Policy content remains product-specific. Study Room's optional arbitrary web mode must not become a default for `devenv` or future consumers.

### 4.8 Artifact pinning and verified downloads

Potential module: `repro/artifacts`

Candidate data model:

```text
name
version
source URL or git remote
immutable ref/digest
architecture
sha256 or registry integrity
license/provenance metadata
installed-version probe
upstream-version probe
```

Candidate operations:

- select the exact architecture-specific artifact;
- download with timeout/retry into a temporary path;
- verify before installation;
- atomically install;
- verify the installed version;
- clone/fetch an exact commit and verify remote, commit, tree, and clean status;
- fail closed when the pin is missing or unavailable;
- never substitute latest.

Keep npm, git, apt, GitHub release, and direct-archive adapters separate rather than hiding meaningful differences behind one opaque downloader.

### 4.9 Cached drift checks

Potential module: `repro/drift`

A shared engine could:

- compare installed versus locked values;
- query upstream metadata with a cache and configurable TTL;
- return structured `ok`, `stale`, `unknown`, and `error` results;
- avoid blocking normal entry on an unavailable update service;
- never install or edit pins;
- provide data for product-specific entry banners and full doctor output.

Each consumer decides which dependencies are intentionally unpinned, how warnings appear, and how its bump command works.

### 4.10 Test fixtures

Potential module: `testing/fixtures`

High-value reusable fakes and assertions:

- fake Secret Service/keyring;
- fake Infisical login/read/write API;
- fake rotating-token endpoint;
- fake `sbx secret ls/set/rm` metadata;
- fake `sbx policy ls/check/allow/deny`;
- WSL2/native host-capability fixtures;
- malicious paths and symlink-overlap fixtures;
- checksum mismatch and interrupted-download fixtures;
- secret-redaction assertions;
- command-line/process-list assertions proving values are absent;
- crash points for lock/atomic-write tests.

Fixtures must never contain token-shaped strings that could be mistaken for live credentials without being clearly marked synthetic.

## 5. Suggested repository shape

Do not create this tree until the implementation language and first extraction are approved. It illustrates desired boundaries:

```text
sandbox-foundation/
├── README.md
├── docs/
│   ├── SHARED-FOUNDATION-NOTES.md
│   ├── THREAT-MODEL.md
│   ├── CONTRACTS.md
│   └── MIGRATION.md
├── src/
│   ├── host/
│   ├── secrets/
│   ├── sbx/
│   └── repro/
├── tests/
│   ├── fixtures/
│   ├── conformance/
│   └── integration/
├── schema/
└── versions.lock
```

A smaller initial repository is preferable. The first release might contain only capability probes, artifact-lock parsing, and test fixtures.

## 6. Consumption and release model

A shared foundation must not create a mutable supply-chain dependency.

Recommended properties:

- versioned releases or immutable git commits;
- checksum/integrity recorded by each consumer;
- no runtime `curl | sh`;
- no sourcing code from a remote branch;
- no requirement that the foundation checkout be mounted into a sandbox;
- consumer-owned adapters and UI;
- compatibility policy for supported `sbx` versions;
- changelog identifying security-contract changes;
- update only through each consumer's explicit bump/PR process.

Possible delivery forms to evaluate:

1. A small compiled host CLI with architecture-specific checksums.
2. A versioned shell library copied into a consumer at build/update time with provenance.
3. A pinned language package used only by trusted host code.

Do not choose based solely on implementation convenience. Compare bootstrap dependencies, auditability, secret handling, cross-architecture support, and how early the helper must run on a fresh host.

## 7. Migration plan

### Phase 0 — Inventory, no refactor

- Keep building Study Room independently.
- Record duplicated code and behavior as it appears.
- Compare actual function inputs, outputs, errors, and tests against `devenv`.
- Update these notes with evidence.

### Phase 1 — Contracts and conformance fixtures

- Define structured capability, error, and secret-command contracts.
- Move or recreate only fake services and conformance cases that do not alter production behavior.
- Run the same fixtures against both projects' existing implementations.

### Phase 2 — Lowest-risk pure extraction

Start with code that has no credentials and no host mutation, likely:

- platform/capability normalization;
- trusted-checkout/workspace overlap checks;
- lock-manifest parsing;
- checksum helpers;
- cached update-result schema.

Release and pin it. Migrate one consumer at a time through a PR with before/after acceptance evidence.

### Phase 3 — Read-only secret transport

Only after conformance tests are mature:

- extract keyring state detection;
- extract Infisical Universal Auth login and read transport;
- extract redaction helpers;
- retain separate per-project schemas and namespaces.

Re-run `devenv`'s host verification before removing its local implementation.

### Phase 4 — Rotating OAuth and sbx registration

Only after Study Room's trusted resolver has survived real refresh rotation and crash tests:

- consider shared structured-record locking/atomic replacement;
- consider shared sbx command-backed/custom-secret registration;
- preserve static-token versus rotating-token distinctions;
- prove path-scoped Infisical write authority first.

### Phase 5 — Consider kit fragments

Do this only if a third sandbox project demonstrates the same kit/config pattern. Before then, common kit generation is likely to hide important agent-specific differences.

## 8. Security invariants for any shared implementation

1. The sandbox sees placeholders, never real secrets or Infisical identity material.
2. Secret values never appear in argv, logs, shell history, committed fixtures, or plain-text temporary files.
3. Secret-producing commands emit exactly the requested value on stdout; diagnostics use stderr.
4. Commands registered with sbx use absolute paths in trusted, non-mounted checkouts.
5. Global sbx secrets and policy are not modified implicitly.
6. Per-project keyring namespaces and Infisical paths remain distinct.
7. Infisical permissions are least-privilege; read-only and rotating-write consumers are not conflated.
8. Redirects, retries, and errors cannot disclose authorization headers or response bodies containing tokens.
9. Trusted host setup shows a plan and obtains approval before sudo, package installation, group changes, or global policy initialization.
10. Integrity mismatch fails closed; no fallback to latest.
11. A hijacked agent's ability to use a proxy-managed credential is limited primarily by that credential's real scope. Placeholder secrecy is not authorization.
12. Tests use synthetic credentials and prove redaction on every error path.

## 9. Product differences that tests must preserve

| Dimension | `devenv` | `study-room` |
|---|---|---|
| Host GUI | Not central | Required for Obsidian Desktop |
| Architectures | x86_64 and aarch64 targets | x86_64 V1 |
| Agent | Claude/Firstmate | Pi teacher plus researcher |
| Multiplexer | herdr | tmux |
| Main workspace | `dev/` state and Firstmate home | Obsidian vault's `Study Room/` |
| Session persistence | Selected Firstmate/Claude memory | Pi sessions disposable |
| Provider secret | Static Claude setup-token | Rotating OpenAI/Kimi OAuth records |
| Infisical operation | Read | Read plus narrowly scoped rotating write |
| Network | Balanced plus explicit domains | Balanced or explicit broad-web mode |
| Host applications | Tailscale/emulator/hub options | Graphical Obsidian Desktop |
| Update behavior | Product-specific status line/doctor | Entry banner and Study Room doctor |

A shared test suite should parameterize these differences, not normalize them away.

## 10. Questions a future design interview must settle

1. Can the current Infisical plan enforce Study Room's path-scoped write without broadening `sbx-host` project access?
2. What implementation language can run early on a fresh Ubuntu host with the smallest trustworthy bootstrap?
3. Should releases be a compiled CLI, pinned source library, or both?
4. What `sbx` version range is supported, and how are behavior changes detected?
5. Which APIs are stable enough to version versus still consumer experiments?
6. Does aarch64 become a foundation requirement because `devenv` supports it, even though Study Room V1 does not?
7. How are host package installation plans represented without taking consent/UI control from consumers?
8. How are foundation security updates propagated without automatic unreviewed consumer upgrades?
9. Which live host checks can safely run in CI, and which require owner-operated WSL/native acceptance?
10. What evidence threshold is required before extracting kit fragments or lifecycle orchestration?

## 11. Initial implementation acceptance criteria

The first real foundation release should not be considered successful merely because both projects import it. It should demonstrate:

- immutable version/integrity pinning in each consumer;
- no new runtime dependency on a mutable checkout;
- unchanged workspace and mount boundaries;
- unchanged secret scopes and destination bindings;
- unchanged entry behavior;
- conformance tests passing in both consumers;
- WSL2 and native Ubuntu host-fixture coverage;
- no secret-bearing output under forced failures;
- rollback by reverting one consumer commit;
- documentation showing which duplicated code was actually removed;
- a reviewed migration PR per consumer, never a flag-day rewrite.

## 12. Useful source material

Before implementation, read the current versions of:

### `devenv`

- `docs/SPEC.md`
- `docs/SECRETS.md`
- `docs/HOST-VERIFY.md`
- `devenv.conf`
- `versions.env`
- `provision.sh`
- `sbxenv.yaml`
- `kits/devenv/spec.yaml`
- secret, host-check, artifact, update, and test helpers under `lib/` and `tests/`

### `study-room`

- `docs/SPEC.md`
- the eventual lock manifest;
- host setup and doctor implementation;
- OAuth resolver and crash/rotation tests;
- Docker secret-registration code;
- WSL2/native host acceptance documentation.

Use repository URLs and committed refs in design documents. Local paths are observations, not portable interfaces.

## 13. Recommended first task for a future agent

Do not begin with secrets or OAuth. Begin with a read-only comparison PR that:

1. inventories the two projects' platform probes, lock formats, download verification, and workspace-overlap tests;
2. proposes one small structured capability-result schema;
3. supplies fixtures for native Ubuntu and WSL2;
4. demonstrates adapters in tests without changing either project's production setup;
5. reports what remains genuinely different.

That task produces evidence for the first extraction while keeping the highest-risk trusted-host and credential paths unchanged.
