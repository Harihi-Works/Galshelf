# Future Development Guide: Game Discovery and Safe Save Synchronisation

**Review date:** 2026-10-02  
**Status:** Reviewed planning baseline; not an implementation or release claim.  
**Scope:** GalShelf-led Windows game/save discovery, Galdex evidence, and Galdrive backup, synchronisation and restoration.  
**Research policy:** Adopt newer techniques only when representative measurements establish meaningful benefit without weakening correctness, compatibility, privacy or recoverability.

[Development index](README.md) · [GalShelf](../../README.md) · [Galdex](../../GALDEX.md) · [Galdrive](../../GALDRIVE.md) · [Galguard](../../GALGUARD.md)

## 1. Purpose, authority and limits

The intended experience is to find relevant local games or saves quickly, associate them with the correct work and compatible release, preserve their directory hierarchy in backups, and restore only the progress needed on this machine. Unambiguous operations should be automatic inside an authorised scope. Real divergence requires a choice; uncertainty must not silently become permission to overwrite.

This guide is additive to existing owner-approved save-sync work, including R05-S. It is not authority to replace existing protocols, alter production trust roots, change account consent, run unknown games, merge implementation branches, publish a release, deploy websites or modify installed applications.

The repository boundary review used `GALDEX.md` and `GALDRIVE.md` on `main` at commit `66f3a9f1c56ca64d05f4cc6cccf0e4bffef72511`. These documents establish that fingerprint matches are not safety proofs, locator rules are not write permissions, and work identity does not establish save compatibility. This review did not audit the current implementation in other repositories. Before coding, reconcile this plan with the actual R05-S implementation, accepted owner directives, current schemas and integration receipts; report conflicts rather than silently replacing earlier decisions.

Carry forward the established product decisions: GalShelf remains the primary product; Galdex is an internal maintainer tool; Galdrive remains independently usable. Keep the separate Sync, Upload backup and Download/restore operations, the metadata-only public fingerprint index, ordinary overlay restore and explicitly selected advanced exact restore. Preserve the owner-approved default automation and fingerprint-contribution settings, while keeping contribution scope and revocation separate from backup permission. Retain the default **10 generations per save line**, the existing configured capacity ceiling and warnings, and protection of the last usable backup. This guide sets no new global capacity value. Windows comes first; Android is outside this increment.

No benchmark on the owner's real library, real cloud account or clean Windows installation was performed for this document. All performance targets below are proposed acceptance gates, not measured achievements. This is a second design review, not independent security certification.

## 2. Second-review findings and required corrections

| ID | Risk in an oversimplified design | Required correction |
| --- | --- | --- |
| RV-01 | A leftover save folder becomes proof that a game is installed. | Keep save presence, installation presence and recovery eligibility separate. |
| RV-02 | Common filenames or changing save bytes become a unique work fingerprint. | Separate identity evidence, locator features and exact content digests; retain ambiguity. |
| RV-03 | A cloud-restored save is later treated as independent local evidence. | Record provenance; prevent restore-discover-upload and evidence self-confirmation loops. |
| RV-04 | Missing, inaccessible or unscanned paths are treated as empty. | Use explicit unknown/incomplete states. Absence requires a complete check of the relevant scope. |
| RV-05 | An empty target is automatically overwritten without checking release or lineage. | Require a verified install, compatible save profile, authorised target and an unambiguous complete remote head. |
| RV-06 | Directory watchers or USN make periodic validation unnecessary. | Detect gaps, restart windows, volume changes and rule changes; invalidate affected cache entries and rescan. |
| RV-07 | Authentic metadata can instruct arbitrary filesystem writes. | Enforce independent local path capabilities and write-time containment, even after authentication. |
| RV-08 | Copying multiple files or roots is described as atomic. | Use staged data, a recovery log and validated resume/rollback; state the limits of cross-root atomicity. |
| RV-09 | A commit marker means all cloud objects arrived. | Validate the complete referenced object set before treating a snapshot as restorable. |
| RV-10 | Ten-generation retention deletes the history needed to detect offline divergence. | Retain sufficient causal metadata or a validated compaction summary after payload pruning. |
| RV-11 | A paper's chunking speedup becomes a product-level speed claim. | Separate algorithm throughput from discovery, upload, restore, request count and user-visible latency. |
| RV-12 | An encrypted sidecar protects public payloads or hides a proprietary protocol. | Define confidentiality and integrity separately; use reviewed cryptography and a cross-device recovery design. |

These findings are resolved as requirements in this plan. Their implementation remains subject to the acceptance cases in Section 15.

## 3. Identity and state model

### 3.1 Keep identity, compatibility and location orthogonal

Use a VNDB work ID as the public work identity. Use verified VNDB release identity where available, or explicit existing release/build evidence; never manufacture a verified release association. Internal installation IDs, profile IDs and snapshot IDs are operational identifiers, not a replacement public work numbering system.

Represent at least:

- Work identity and the evidence supporting it.
- Release/build applicability and a reviewed save-compatibility group.
- Local installation instance, resolved path and observation provenance.
- Save profile ID, revision, logical roots and selected files.
- Save content state, snapshot lineage and current local binding.

Two installations of the same work may need different bindings. Conversely, two installations may resolve to the same physical save directory. Detect such overlaps before scheduling writes; serialize access to the shared physical target and do not invent independent save ownership.

Without a VNDB identity, this trusted fingerprint-driven auto-restore workflow remains unsupported. This does not remove any separately authorised manual-directory backup feature that already exists.

### 3.2 Use categorical evidence, not invented probabilities

Useful evidence categories are `candidate`, `corroborated`, `identity_verified`, `release_applicable`, `ambiguous`, `stale` and `unsupported`. Record why a result has its category. Names, common directory structures, shared engine binaries and partial hashes must not independently produce `identity_verified`.

A numeric ranking may order candidates internally, but it is not a calibrated probability or restore permission. Do not label a match as 98% trustworthy without a held-out, labelled validation study.

For presence, distinguish `install_verified`, `save_only`, `cloud_only`, `not_present_in_checked_scope` and `unknown_or_incomplete`. A stale library entry alone is not a verified installation.

For saves, distinguish `absent_verified`, `initialisation_only_verified`, `present`, `busy`, `inaccessible`, `placeholder_unavailable`, `ambiguous`, `security_blocked` and `unsupported`. A bundled `savedata` directory in a newly extracted distribution counts as existing content, not automatically as an empty or disposable save.

### 3.3 Track provenance and prevent feedback loops

Each observation should identify its origin: direct local installation evidence, direct local save observation, user binding, imported third-party rule, or a specific Galdrive restore operation. Evidence generated by restoration must not independently prove that the game was installed before restoration, create a new trusted Galdex rule, or expand the user's authorised scope.

Associate self-generated filesystem events with an operation ID, then reconcile actual content instead of blindly dropping events. Equal logical content should not trigger another upload merely because timestamps changed. If the game writes during reconciliation, that independent change must remain visible.

## 4. Discovery architecture: compile rules into bounded search plans

### 4.1 Prioritise evidence already available

Use three coordinated entry points: GalShelf's known installation instances; works already represented in the user's authorised cloud catalogue; and Galdrive's bounded independent discovery in standard save roots and user-selected game roots. Cloud catalogue entries reduce search priority, not the amount of evidence required to establish a local installation.

Use a shared discovery library and a versioned local index contract first. Do not require a fourth product, an always-running system service, administrator privileges or a running GalShelf instance. A shared database needs explicit writer coordination and migrations; do not assume several applications can safely modify the same database without a protocol.

### 4.2 Compile location rules by root and fixed prefix

A rule should specify the logical root, fixed relative prefix, bounded variable segments, candidate features, validation predicates, applicability, exclusions and resource limits. Compile common prefixes into one traversal plan. For example, several profiles under `{ROAMING}/Vendor/` should share the parent enumeration instead of repeatedly traversing it.

Use exact directory maps or a trie and inverted candidate indexes as a straightforward baseline. Order probes using measured selectivity and I/O cost. A filter should not exclude a valid alternative when its evidence is only advisory; model required features, alternatives and optional features explicitly.

An absent fixed prefix can prune a branch only after a successful, complete check. Access denial, an unplugged volume, a truncated enumeration or a stale rule generation yields unknown, not a negative match. Cache negative results with the root identity, rule generation and invalidation conditions, not indefinitely.

Never expand one broad wildcard into an unbounded recursive AppData scan. Bound entries, depth, bytes read, parser work and elapsed time; expose the unchecked remainder. Limits are configurable scheduling budgets, not evidence that remaining files do not exist.

### 4.3 Resolve Windows roots rather than constructing C-drive paths

Resolve logical folders for the intended signed-in user through supported Known Folder APIs. Documents and similar locations may be redirected; use the resolved location, not a hardcoded username or drive. Microsoft's Known Folder guidance supports this approach. [S1]

Support reviewed profiles for `GAME_ROOT`, `ROAMING`, `LOCALAPPDATA`, `LOCALLOW`, `DOCUMENTS` and `SAVED_GAMES`. `USERPROFILE` and `PROGRAMDATA` require narrowly scoped rules; their existence is not authority to enumerate or back up their whole contents. Optional locations such as Steam user data, per-user virtualised locations or registry data must have explicit adapters and applicable rules. Do not claim that all games store progress in ordinary files under a few folders. Preserve any already-supported legacy paths without introducing broad new registry import/export.

A user-selected installation root may move to another volume. Re-resolve and revalidate it; a drive letter alone is not stable volume identity. Keep original filename spelling and bytes where supported, with a documented comparison policy. Do not blindly lowercase or Unicode-normalise every path and then write it back: detect aliases and collisions on the target filesystem.

### 4.4 Bound content work and archive handling

The fast path enumerates metadata and performs direct path probes. Only surviving candidates receive bounded structure checks, safe static metadata parsing and selected immutable-file verification. A content hash verifies a located file; it does not locate an unknown file without an index.

Save content digests change with progress and primarily establish equality, not work identity. Partial hashes may be candidate filters only when tied to a tested profile. Shared runtimes and helper binaries must not be treated as unique work anchors. Preserve existing public SHA-256 fingerprint semantics and any already-approved first-stage identity-read budget; an identity budget is not a save-size limit or a CDC threshold.

Do not open nested archives, disguised archives, mounted images or unknown executables in the default discovery path. An archive is not an installed game. Any future deep-discovery adapter needs separate authorization, bounded decompression and read-only parsing. Exclude Galdrive backup stores, staging areas and known restore workspaces from ordinary local discovery unless explicitly inspecting them as backups.

### 4.5 Engine rules and external manifests are cold-start aids

Use engine detection to propose likely save layouts. Ren'Py, for example, documents configurable save locations and a Windows path based on `config.save_directory`; its folder need not equal a display title. [S5] Parse only known, bounded static forms. Do not execute Python, game scripts or configuration expressions, deserialize unsafe object formats, or start an unknown game just to discover its save location. Unresolved dynamic paths remain unresolved.

Ludusavi's manifest format is a useful external source of location candidates. [S6] Retain source, revision and applicable attribution/permission information; distinguish the format's licence from upstream data obligations. Map to VNDB, validate release applicability and verify samples before publishing trusted Galdex rules. AI may assist initial review, but the existing final human-review and signing boundaries remain in place. Imported labels such as `reviewed` are not trust.

## 5. Incremental indexing and consistency

Use directory notifications as hints to revisit affected scopes. `ReadDirectoryChangesW` may lose records on buffer overflow and does not report changes to the watched directory itself. Handle overflow and root moves/deletion explicitly; perform the required scope reconciliation. [S2]

An initial scan should establish a change-observation boundary before enumeration, scan the bounded scope, reconcile intervening events, and publish a generation only when its coverage is known. If the platform cannot provide that boundary, use a compensating rescan and retain uncertainty. Ordinary notifications do not persist through arbitrary application downtime.

Treat volume-level USN as an optional, separately permissioned accelerator. Microsoft documents administrator requirements for the conventional change-journal operations; a normal installation must retain a non-elevated path. [S3] Record volume identity, journal ID, cursor and coverage; invalidate on gaps, recreation, truncation, unsupported filesystems or permission failure. Do not create/resize a journal or install an elevated helper as a hidden default.

Persist scan generation, rule generation, roots covered, completion state, skip reasons, notification health and pending rechecks. Size and modification time help schedule checks but are not proof that bytes are unchanged. After offline gaps or unreliable observation, revalidate relevant content before a destructive sync decision. Even a valid change journal is not an antivirus verdict or an application-consistent save snapshot.

Cloud placeholders require special handling: opening a file can trigger hydration, and a cloud reparse point is not automatically a symbolic link. [S4] Prefer metadata-only probes, record unhydrated/unavailable state, and do not silently download a large cloud tree to compute candidate hashes. Permit supported cloud-placeholder behaviour separately from link-escape rules.

## 6. SaveSet structure and portable path mapping

Treat one logical snapshot as a SaveSet with one or more named sources. Preserve relative hierarchy within each source. Resolve each source root independently on the receiving machine; never replay an originating absolute username/drive path.

Illustrative layout, not a new approved wire schema:

```text
snapshot/
  manifest.json
  manifest.auth
  private-metadata.enc       (optional, only when justified)
  payload/
    source-1/slots/slot01.dat
    source-1/system.dat
    source-2/preferences.dat
```

The manifest should describe the work identity, release evidence, compatibility group, profile revision, snapshot ID, parent/causal references, source IDs, logical roots, relative paths, selected files, sizes, full content digests and required/optional source groups. Algorithm IDs and format versions must be explicit. Examples must not be mistaken for production protocol versions.

`{GAME_ROOT}/savedata/` restores under the verified receiving installation. `{ROAMING}/Vendor/Game/` resolves under the receiving user's Roaming folder. An original absolute path may remain a local diagnostic, but it is not needed in an ordinary cloud snapshot. Never infer a target by arbitrary environment expansion from remote metadata.

Capture logical empty directories only when an applicable rule requires them. Empty-directory existence alone is not identity evidence. Separate progress, machine-specific configuration and optional screenshots; a broad file extension or root directory is not a backup allowlist.

If multiple sources overlap or resolve to the same physical target, consolidate only under an explicit compatible rule or report a binding conflict. Required roots form a consistency unit. Independent restoration of optional groups must be specified by the profile; silently restoring half a required SaveSet is prohibited.

## 7. Automatic operation decision table

All automatic actions assume current authorisation, a verified compatible profile, valid cloud metadata, acceptable security state and a safe operational window. Discovery itself remains read-only.

| Situation | Automatic behaviour |
| --- | --- |
| Cloud entry only; no current installation evidence | Leave progress in the cloud; do not create local save trees. |
| Verified save only; game absent or unknown | Offer/maintain association and permit backup within an existing authorised scope; no installation-based auto-restore. |
| Verified installation; target genuinely absent or reviewed initialisation-only content; one compatible complete remote head | Permit initial recovery under the enabled policy, including safe creation of the reviewed target path. |
| Local and remote logical contents are equal | Record reconciliation; do not duplicate payload or overwrite identical files. Preserve relevant causal information. |
| Common base known; local equals base; compatible remote is its descendant | Permit remote-to-local recovery after safety checks. |
| Common base known; remote equals base; local independently changed | Permit upload without first downloading over local progress. |
| Both diverged from the base, or remote has competing heads | Preserve alternatives and request a progress choice. |
| Existing local content with no trustworthy base | Preserve it; do not infer that a later timestamp wins. Back it up safely and treat direction as unresolved. |
| Busy, blocked, unsupported, unreachable or incomplete | Keep data unchanged; display a non-destructive pending/reason state and retry only within policy. |

A read-only refresh must not itself write local saves. It may update state and schedule a separately authorised automation action. Keep those command paths distinguishable in code and logs.

The no-save case must not require launching the game to create the very evidence needed for recovery. Verified install identity and a reviewed rule that allows target creation can establish the destination. Conversely, generic first-run files or bundled saves may be ignored only under a specifically reviewed initialisation-content rule.

User-requested restoration of an older version is a deliberate action with backup/rollback protection, not a forged claim that the older version is the newest. Ordinary missing files do not imply delete intent; exact restoration remains an explicit advanced mode.

## 8. Snapshot creation, lineage and cloud publication

### 8.1 Obtain a coherent backup before hashing or uploading

Prefer capture after the game has exited and expected writers have finished. A debounce interval or one unchanged timestamp is insufficient proof of consistency. Coordinate the suite's processes, check relevant writers where possible, and revalidate file identity/content during capture. Windows file sharing modes can constrain concurrent access but must be used and tested correctly. [S7]

Copy to an operation-owned staging area, compute complete digests from the staged bytes and confirm that the capture belongs to a coherent save generation. If a required file changes mid-capture, retry with a bound or postpone; never publish a mixed snapshot as verified. Any future live-backup or VSS adapter requires its own application-consistency contract. Do not kill or suspend games as an implicit sync action.

### 8.2 Prefer file-level incrementality first

Hash only the files that need content revalidation, not the whole installation. Upload changed files, reuse authenticated existing objects where the backend and format allow it, and represent an equal logical snapshot without duplicate progress versions caused solely by device identity or timestamps.

Use canonical logical content identity separately from the snapshot envelope: work/profile/source-relative paths, sizes and digests identify content; parent references, device operations and timestamps describe history. Equal payload does not erase necessary lineage. Syncthing's version-vector design is a useful example of causal metadata, but this guide does not mandate replacing the existing lineage representation with Syncthing's protocol. [S8]

### 8.3 Publish data before publishing recoverability

Write immutable data, then an authenticated immutable manifest/commit record. A receiving client verifies all required referenced objects before declaring the snapshot restorable. Neither a filename ending in `complete` nor a visible commit marker alone proves completeness.

For direct providers, use their supported conditional update or object-version facilities only after verifying each adapter's semantics. For sync-folder backends, assume files may arrive out of order and concurrent clients may produce conflict copies. Discover and retain competing immutable heads rather than letting a mutable `latest.json` silently erase a branch. A single mutable pointer may be a cache, not the sole history authority.

Record backend-specific states such as local-store committed, provider upload acknowledged and remote snapshot verified. A completed local write is not proof that an external sync client has uploaded it; this distinction already exists in the Galdrive product boundary.

### 8.4 Retention and garbage collection

Keep the existing 10-generation-per-line policy with configured capacity enforcement, but do not prune the last usable version, unresolved conflict heads, explicitly protected versions or objects used by in-flight operations. When headroom is insufficient for a safe new snapshot, stop and warn rather than destroy the only recovery point.

Distinguish payload retention from causal-history retention. Keep enough metadata, tombstones or validated summaries to recognise an offline device's ancestor after old payloads are pruned. Missing ancestry is uncertainty, never permission for last-writer-wins.

A deduplicated store requires reachability-based garbage collection, a safe grace/coordination strategy for offline or concurrent clients, and complete reference enumeration. A failed or partial cloud listing must never authorize deletion. Do not introduce chunk-level garbage collection before proving this lifecycle end to end.

## 9. Safe staged restoration

Implement a recoverable state machine, for example `PLANNED -> STAGED -> PREIMAGE_SAVED -> APPLYING -> VERIFIED -> COMMITTED`, with explicit pending, failed, rollback and recovery-required states. The names are illustrative; integrate with the existing staged-restore implementation rather than creating a competing one.

Before application, authenticate metadata, validate schema and bounds, resolve roots locally, check release compatibility and overlapping targets, verify complete staged bytes, confirm the required save unit is idle and reserve enough space for temporary data plus protected preimages. Recheck the destination preimage immediately before replacement. A changed preimage returns to reconciliation.

Use safe per-file replacement under validated parent handles and a durable operation log. A rename may help an individual replacement; it does not make several directories or volumes transactional. Across required roots, interrupted operations must resume or roll back from verified state. Do not describe this as guaranteed all-root atomicity. If free space, filesystem behaviour or open handles prevent a safe operation, leave it pending.

A suite lock is cooperative: it does not prevent a game launched externally from writing. Recheck process/activity and destination state at the relevant boundary. If activity cannot be safely excluded, defer restoration. After interruption, a later game write must not be overwritten by a blind automatic rollback; preserve both the preimage and new content and require recovery resolution.

Continue ordinary overlay semantics and advanced exact semantics. Overlay is not a licence to combine incompatible save generations. Profiles whose correctness requires deletion or exact set replacement need an explicit supported policy; do not silently convert ordinary recovery into directory mirroring. Never restore executables, launcher scripts, DLLs or arbitrary personal files merely because they were placed beside a save.

## 10. Path, parser and trust boundaries

A signed Galdex rule describes evidence and allowed candidate patterns; it does not grant filesystem authority. A user's authenticated snapshot proves integrity under that user's backup key; it is not a publisher signature or a malware verdict. Keep these trust domains and keys separate. A successful work match must never bypass a Galguard block.

Before use, reject malformed/oversized manifests, unsupported mandatory schema fields, duplicate or colliding paths, invalid source references and excessive object counts. A read-only inspection/export mode may remain available for unsupported versions without enabling automatic recovery.

Accept only valid relative payload paths beneath locally approved roots. Reject traversal, absolute/drive/UNC/device paths where not explicitly authorized, alternate data stream syntax, reserved-name aliases and target collisions. Validate resolved containment at the time of use, not only by a string prefix test before writing. Handle parent-directory replacement races and hardlink aliases; do not modify a multiply-linked destination in place as a shortcut.

Do not follow untrusted symlinks/junctions out of an authorised tree. Legitimate Known Folder redirection and supported cloud reparse points require deliberate resolver handling, not a blanket ban on every reparse point. Windows exposes sharing and reparse-point controls through its file APIs; their correct combination requires dedicated adversarial tests. [S7]

Rules must remain bounded declarative data, not downloaded commands or executable regular-expression logic with uncontrolled cost. Use sandboxed/bounded parsers when examining unfamiliar formats. Published fingerprints contain metadata and evidence, not game binaries or private saves. Contributions should not upload original usernames, absolute personal paths or mutable save contents as supposedly public fingerprints.

The threat model includes untrusted cloud objects, third-party rules, malformed local files, interrupted operations and ordinary concurrent writers. It does not claim protection from a fully compromised host or administrator controlling both the process and its keys. Keep antivirus and OS protections active during testing; an identity match is not permission to disable them.

## 11. Portable metadata, authentication and optional encryption

The baseline should keep original save files usable by the owner. A readable manifest plus detached authentication can preserve portability. A private encrypted sidecar may hold justified private metadata, but omit sensitive data entirely when synchronisation does not need it. Do not duplicate an entire unencrypted tree and an entire chunk store by default.

Authenticate the manifest's exact defined serialization and bind its file paths, sizes, full digests, profile/release context, format version and causal references. Use the existing reviewed representation or a specified canonical format; RFC 8785 is one reference for deterministic JSON serialization, not a mandate to change an already interoperable format. Reject ambiguous duplicate keys before interpreting authenticated intent. [S10]

Use reviewed libraries for MAC/AEAD and key derivation. Retain a correct existing construction unless a reviewed migration justifies changing it. AES-GCM and XChaCha20-Poly1305 are examples, not interchangeable wire formats; algorithm-specific nonce limits and associated-data binding must be enforced. Do not invent proprietary encryption or derive an encryption key directly from a file fingerprint. [S9]

A readable payload leaks its contents, filenames and potentially game identity regardless of whether a sidecar is encrypted. A full-encryption mode is a separate owner-visible tradeoff, with a tested export/decryption path. Full encryption can still leak sizes, counts, timing and chunk patterns. The CCS 2025 CDC research shows why secret chunker parameters alone are not a sufficient confidentiality design. [S16]

DPAPI is appropriate for local protection of a cached key, not as the sole portable backup-key envelope: Microsoft documents that decryption is usually tied to the same computer and logon credentials. [S11] Cross-device enrollment, an independently recoverable key path, key rotation, device revocation and clean-reinstall recovery must be designed and tested before encrypted metadata becomes mandatory for restoration. Never require a publisher signing key on a client.

A valid MAC/signature establishes integrity, not freshness. Preserve authenticated causal heads and anti-rollback state where available; distinguish an explicitly requested old snapshot from a replay presented as latest. A clean machine with no trusted checkpoint or independent witness cannot prove that an adversarial cloud is showing the globally newest snapshot. Do not claim otherwise.

If authentication fails, do not auto-restore. Keep manually readable payloads available for a separately authorised, safety-checked import; never silently bless modified cloud data as a trusted snapshot. Missing keys or unsupported crypto should not cause deletion of recoverable bytes.

## 12. Research adoption register

This register records relevance, not a promise to use every technique. All reported gains belong to their authors' workloads and baselines. They are not Galdrive measurements, and unrelated papers' multipliers must not be compared as a leaderboard.

| Technique and evidence | Relevant use | Decision and promotion condition |
| --- | --- | --- |
| Prefix-compiled search plans, exact maps and incremental invalidation | Reduce directory visits and repeated work | Baseline design; measure against existing traversal before claiming a speedup. |
| Binary Fuse Filters, JEA 2022 [S12] | Compact negative filtering in a large, mostly static rule/fingerprint index | Optional experiment only if it avoids measurable expensive lookups. Exact records remain authoritative. |
| FastCDC, USENIX ATC 2016 [S13] | Established CDC comparison for larger changing files | Benchmark baseline, not a reason to chunk every save file. |
| VectorCDC, FAST 2025 and 2026 ACM TOS extension [S14] | Accelerate content boundary selection when chunking is a measured bottleneck | Isolated prototype; require real-save reuse, Windows portability and end-to-end improvement. |
| Vectorized SeqCDC, IEEE TPDS 2026 [S15] | Another vectorized boundary-selection candidate | Compare independently with file-level, fixed-block and CDC baselines; choose at most the justified implementation. |
| RepMaxCDC, 2026 author technical report/implementation [S17] | Tighter chunk-size behaviour and object-store integration experiments | Deferred. Do not describe the technical report as peer-reviewed or infer production suitability from another project's adoption. |
| Rateless IBLT, SIGCOMM 2024 [S18] | Reconcile very large sets between cooperating endpoints | Deferred until catalogue/object-set reconciliation is a demonstrated cost. A generic cloud drive does not execute the peer protocol for us. |
| BLAKE3 official implementations [S19] | Optional internal digest acceleration | Benchmark only; keep SHA-256 public fingerprint compatibility and do not assume hashing dominates I/O. |
| Breaking and Fixing Content-Defined Chunking, CCS 2025 [S16] | Threat-model and leakage review | Adopt the caution now; any deduplication/encryption combination needs its own reviewed leakage statement. |

### 12.1 Filtering is not discovery or verification

Binary Fuse Filters answer approximate membership questions. A negative answer only applies to the exact successfully built set/version; a positive result still needs an exact lookup. Publish the filter and exact index as one validated generation, with safe fallback on corruption, construction failure, unknown format or version mismatch. Do not calculate a whole-file hash merely to ask a filter if it is worth hashing that file. The paper's compact-index benefits do not eliminate filesystem enumeration. [S12]

### 12.2 Chunking is a storage optimisation, not an identity mechanism

The 2026 VectorCDC manuscript reports 8.35–26.2 times higher chunking throughput against its selected vector-accelerated comparisons. Its detailed conclusion identifies these comparisons as vector-accelerated hash-based algorithms. This is not an 8–26 times gain for complete sync or superiority over every contemporary chunker. [S14]

The 2026 SeqCDC manuscript reports roughly 10 times the throughput of its unaccelerated comparisons and 1.2–1.35 times its vector-accelerated comparisons. Earlier manuscripts differ; pin the exact paper and implementation version in any benchmark. [S15] Neither result establishes behaviour on Galgame save histories.

A hashless chunker avoids a rolling hash for boundary selection; it does not remove the need for full cryptographic integrity checks. Do not copy non-cryptographic or obsolete fingerprint choices from a benchmark harness into the restore trust path.

Begin with file-level incrementality. Add CDC only for a measured class of sufficiently large, repeatedly changing files. Compressed/encrypted or extensively rewritten save formats may provide poor block reuse; measure actual generations rather than assuming all saves differ locally. Keep thresholds empirical rather than adopting a paper's chunk size as a universal constant.

A chunked store changes portability, request count, retention and failure recovery. Specify block ordering, full-file digest verification, bounded decompression, algorithm parameters, old-reader behaviour and export reconstruction before rollout. If compression/encryption is used, define the order and authentication boundaries explicitly. Do not use unsafe cross-user convergent encryption or global deduplication merely to preserve savings.

CPU acceleration must have supported runtime dispatch and a tested portable fallback. Record compiler flags, CPU features, native dependency provenance and any relevant licence/patent review before shipping. A recent paper or available source repository is not, by itself, production acceptance.

## 13. Performance acceptance and experiment design

Measure stages separately: root resolution, directory enumeration, candidate matching, evidence reads, content hashing, optional chunking, compression/encryption, provider requests, transfer and restoration. Count directory visits, entries inspected, logical and physical bytes read when measurable, bytes uploaded, remote requests, retries, peak memory and foreground responsiveness.

Compare: current implementation; bounded metadata traversal; direct per-rule probing; compiled/shared-prefix probing; warm incremental indexing; and optional accelerators. Do not give a new method a warm cache while measuring the baseline cold. Record storage type, filesystem, OS, CPU, cache condition, rule corpus version, privacy settings and background software. Do not disable security software to obtain attractive numbers.

Use a labelled corpus spanning renamed directories, several releases of one work, shared engine files, orphan saves, installed games without saves, bundled progress, multiple physical installations, redirected Known Folders, non-ASCII paths and generic unrelated data. Include at least several real save generations for each format class under study. Synthetic directory populations can test scale, but not establish game-identification accuracy.

Report work-identification, release-applicability and save-profile precision/recall separately, plus abstention/unknown rates and false automatic actions. Keep a holdout split that prevents the same files or near-identical release copies from appearing in both rule construction and validation. Raw personal paths, game binaries and private saves stay local; publish aggregate/redacted receipts only with permission.

Proposed performance gate: promote an optional optimisation only after repeated, representative end-to-end measurements show a meaningful improvement, with approximately **25% lower median completion time** as an initial target, or a comparably material documented reduction in transfer/request cost without a meaningful latency regression. This is a planning threshold, not a universal guarantee. Record absolute savings too: a large percentage on a negligible step does not justify substantial complexity.

Use repeated runs, report distribution and variability, and disclose small sample sizes. Do not manufacture p95 confidence from a handful of observations. Safety, compatibility and old-backup recovery are mandatory regardless of speed; a performance win cannot compensate for an incorrect overwrite.

Discovery benchmarking is read-only in intent: no launches, game-file writes, archive extraction, registry edits, real cloud uploads or automated restores. OS access metadata or provider activity may still occur; account for those effects, especially placeholders. Restoration and failure-injection experiments run on disposable copies first. A real-account or native Windows result must be labelled untested until actually performed.

## 14. Delivery sequence and ownership

**Phase A — reconcile and measure.** Map actual existing code and owner directives to this guide. Add minimal instrumentation and a labelled, privacy-preserving benchmark harness. Identify the dominant cost before adding algorithms.

**Phase B — discovery foundation.** Add explicit states/provenance, logical-root resolution, compiled bounded rules, safe engine adapters, cache invalidation and ordinary-permission notifications. Use existing data and reviewed external candidates; do not wait for a complete global fingerprint corpus.

**Phase C — recovery correctness.** Integrate SaveSet mapping, coherent capture, lineage, complete cloud publication, destination safety and recovery logging with the existing R05-S/staged-restore path. Preserve current settings and history. This phase is a prerequisite for automatic restoration rollout.

**Phase D — optional acceleration.** Prototype only measured bottlenecks: USN, compact membership filters, digest acceleration or large-file CDC. Keep features gated and keep old-format readers/export paths available. Reject experiments that fail the benefit/risk gate without making their absence a release blocker.

**Phase E — controlled rollout.** Start with discovery-only/shadow decisions, then authorised backup, then safe empty-target recovery and subsequently lineage-proven updates. Compare proposed actions with expected outcomes before allowing writes. Include a feature rollback switch that stops new automatic actions without invalidating existing backups.

Workers may implement independent bounded tasks in parallel and run targeted, inexpensive checks. One integrator owns interface convergence and the final candidate. Treat broad tests as synchronization barriers: integrate required work first, fix failures with targeted retests, perform one holistic review of the stable integrated candidate, then run final regression/native smoke/build/package gates where relevant. Track pre-existing failures separately. This documentation change does not authorize those implementation or release actions.

## 15. Acceptance matrix

Every case needs an expected result and reproducible evidence. Passing an artificial fixture does not count as a real-account or real-device pass.

| Case | Required outcome |
| --- | --- |
| A01: Shared `save*.dat` / engine DLLs across unrelated works | Remain ambiguous without stronger identity evidence; zero automatic restore. |
| A02: Game removed, saves remain | Save-only state; authorised backup still possible, no install claim. |
| A03: Fresh verified install with no save directory | Resolve a reviewed creation-capable profile; recover only one compatible complete head. |
| A04: Newly extracted game contains someone else's bundled save | Treat as existing content, not empty; preserve before resolving direction. |
| A05: Same VNDB work, incompatible release or patch | Do not transfer compatibility by work ID alone. |
| A06: Two installs resolve to one physical save target | Detect overlap and coordinate one physical write scope. |
| A07: Library moved, drive letter reused, volume unplugged | Revalidate identity; missing/unavailable is not delete intent. |
| A08: Watch overflow, application downtime, USN gap or changed journal | Mark incomplete and reconcile; no stale-cache overwrite. |
| A09: Redirected Documents and unhydrated cloud placeholders | Use real logical roots; no unbounded hydration during fast discovery. |
| A10: Concurrent modification while capturing or restoring | Retry/defer safely; never publish mixed bytes or overwrite a changed preimage. |
| A11: Two offline devices diverge; clocks disagree | Retain both heads and prompt for progress, not clock order. |
| A12: Same content from another device or a self-generated restore event | No redundant payload/version loop; preserve causal metadata. |
| A13: Remote marker arrives before payload; pagination/network fails | Snapshot not restorable until complete; no destructive pruning. |
| A14: Crash after each restore state transition and between required roots | Resume/rollback from verified log; never claim whole-operation atomicity. |
| A15: New local play occurs after an interrupted restore | Do not blindly roll back over new progress; preserve recovery alternatives. |
| A16: Traversal, UNC/device/ADS syntax, aliases, hardlinks or parent-link race | No write outside the local authorised capability. |
| A17: Forged, replayed, truncated or unsupported metadata | Fail safe, retaining bytes and last usable backup; distinguish manual old-version restore. |
| A18: Quota reached, disk full, sharing violation, protected directory | Stop safely and explain; do not kill games, elevate silently or delete the only backup. |
| A19: Offline device returns after more than 10 generations | Retained causal summaries identify lineage or yield uncertainty, not silent overwrite. |
| A20: Corrupt/mismatched filter, unavailable SIMD, unsupported filesystem | Exact/portable bounded fallback with no reduction in decision quality. |
| A21: New hash/chunker/crypto version encounters an old backup/client | Explicit compatibility result and preserved export/recovery path; no silent reinterpretation. |
| A22: Clean reinstall without previous DPAPI state | Portable-mode manual files remain usable; encrypted mode succeeds only with the tested independent recovery mechanism. |
| A23: Galguard blocks content or identity verification succeeds on malicious files | Identity never clears the block or establishes safety. |
| A24: Real local, OneDrive and Google Drive adapters | Record actual validation separately from mocks and external-sync-folder completion. No claim of new backend support. |

Release-blocking findings include wrong-work or incompatible restore, path escape, loss of the last usable backup, undetected partial publication, broken old-backup recovery and secrets/private saves in published evidence. Zero observed failures is an acceptance result for the tested corpus, not a mathematical zero-risk claim.

## 16. Explicit non-goals and unresolved implementation choices

Do not make full-disk hashing, universal whole-disk indexing, nested-archive unpacking, a driver, an always-elevated daemon, LLM-inferred write paths, executable rule plugins, arbitrary binary-save merging or proprietary cryptography prerequisites for this work. Do not add official hosted storage, new cloud providers or whole-game backup under this guide.

Before implementation promotion, resolve concrete bindings to existing code: the actual profile schema and trust-generation contract; current process/lock integration; exact provider conditional-write capabilities; recovery-key UX; optional-source semantics; and any live-backup consistency adapter. These are integration tasks to verify, not justification to weaken safe defaults or force the owner to redesign already accepted behaviour.

The baseline is deliberately useful without research accelerators: bounded targeted discovery, clear evidence, portable hierarchy, file-level incrementality and recoverable writes. New technology earns adoption by outperforming this baseline under the actual workload.

## 17. Primary references and verification scope

Sources were checked on 2026-10-02. Bibliographic years describe the cited versions, not when an idea first appeared. External materials support platform behaviour and research findings; the requirements and thresholds in this guide are project design decisions, not claims that the sources implement this product.

- **S1 — Windows Known Folders.** Microsoft: https://learn.microsoft.com/en-us/windows/win32/shell/known-folders
- **S2 — Directory notifications and overflow handling.** Microsoft, `ReadDirectoryChangesW`: https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-readdirectorychangesw
- **S3 — USN journal identity and conventional privilege requirements.** Microsoft: https://learn.microsoft.com/en-us/windows/win32/fileio/using-the-change-journal-identifier
- **S4 — Cloud placeholders, hydration and reparse points.** Microsoft: https://learn.microsoft.com/en-us/windows/win32/cfapi/build-a-cloud-file-sync-engine
- **S5 — Ren'Py save-location configuration.** Official configuration documentation, especially `config.save_directory` and `config.savedir`: https://www.renpy.org/doc/html/config.html
- **S6 — Ludusavi manifest format and source project.** https://github.com/mtkennerly/ludusavi-manifest
- **S7 — Windows sharing modes and reparse-point controls.** Microsoft, `CreateFileW`: https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew
- **S8 — Causal version metadata example.** Syncthing Block Exchange Protocol v1: https://docs.syncthing.net/specs/bep-v1.html
- **S9 — Authenticated encryption, associated data and nonce requirements.** Libsodium: https://libsodium.gitbook.io/doc/secret-key_cryptography/aead
- **S10 — Deterministic JSON serialization reference.** RFC 8785, JSON Canonicalization Scheme: https://www.rfc-editor.org/rfc/rfc8785
- **S11 — DPAPI portability limitations.** Microsoft, `CryptProtectData`: https://learn.microsoft.com/en-us/windows/win32/api/dpapi/nf-dpapi-cryptprotectdata
- **S12 — Binary Fuse Filters: Fast and Smaller Than Xor Filters.** Thomas Mueller Graf and Daniel Lemire, Journal of Experimental Algorithmics 27 (2022), DOI 10.1145/3510449. Author manuscript and publication metadata: https://arxiv.org/abs/2201.01174
- **S13 — FastCDC: A Fast and Efficient Content-Defined Chunking Approach for Data Deduplication.** USENIX ATC 2016: https://www.usenix.org/conference/atc16/technical-sessions/presentation/xia
- **S14 — Accelerating Data Chunking in Deduplication Systems using Vector Instructions.** Sreeharsha Udayashankar, Abdelrahman Baba and Samer Al-Kiswany, ACM Transactions on Storage (2026), extending VectorCDC/FAST 2025. Author manuscript: https://cs.uwaterloo.ca/~alkiswan/papers/VectorCDC_TOS_2026.pdf ; author publication list: https://wasl.uwaterloo.ca/projects/deduplication/
- **S15 — Vectorized Sequence-Based Chunking for Data Deduplication.** Sreeharsha Udayashankar, Ali Assem Mahmoud and Samer Al-Kiswany, IEEE TPDS (2026). Author manuscript: https://cs.uwaterloo.ca/~alkiswan/papers/Vectorized_SeqCDC_TPDS_2026.pdf ; distinguish it from the earlier arXiv version: https://arxiv.org/abs/2505.21194
- **S16 — Breaking and Fixing Content-Defined Chunking.** Kien Tuong Truong, Simon-Philipp Merz, Matteo Scarlata, Felix Günther and Kenneth G. Paterson, CCS 2025. Research-group publication summary: https://research.ibm.com/publications/breaking-and-fixing-content-defined-chunking
- **S17 — Content-Defined Chunking with tight chunk size bounds / RepMaxCDC.** Ed Schouten, 2026 author technical report and implementation; not treated here as a peer-reviewed publication: https://github.com/buildbarn/go-cdc
- **S18 — Practical Rateless Set Reconciliation.** Lei Yang, Yossi Gilad and Mohammad Alizadeh, ACM SIGCOMM 2024, DOI 10.1145/3651890.3672219. Author manuscript: https://arxiv.org/abs/2402.02668 ; conference programme: https://conferences.sigcomm.org/sigcomm/2024/program/
- **S19 — BLAKE3 official implementations and test vectors.** https://github.com/BLAKE3-team/BLAKE3

**Review conclusion:** proceed with bounded discovery and safe-sync foundations; require measured benefit before optional algorithm adoption. No runtime, benchmark, restore, cloud-account, security certification or release-validation pass is asserted by this document.
