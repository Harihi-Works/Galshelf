# Future Development Guides

[GalShelf home](../../README.md) · [Galdex](../../GALDEX.md) · [Galdrive](../../GALDRIVE.md) · [Galguard](../../GALGUARD.md)

This directory contains forward-looking engineering guidance. A reviewed plan is not evidence that its features are implemented, integrated, benchmarked or released. Existing owner decisions, production trust boundaries and verified compatibility contracts must be reconciled before implementation.

## Game discovery and save synchronisation

Read [Game Discovery and Safe Save Synchronisation](GAME_DISCOVERY_AND_SAVE_SYNC_FUTURE_GUIDE.md), reviewed on **2026-10-02**.

The guide covers bounded game/save discovery, source-relative backup hierarchy, installation and save-state separation, provenance, incremental indexing, safe automatic recovery, causal history, cloud publication, encryption/recovery boundaries and research adoption gates. It includes a second-review findings register, a phased implementation sequence, 24 acceptance scenarios and 19 primary references.

The baseline is useful without experimental algorithms. VectorCDC, SeqCDC, Binary Fuse Filters, BLAKE3 and optional USN acceleration require representative measurements where relevant; RepMaxCDC and Rateless IBLT remain deferred. No paper's benchmark is a Galdrive performance result.

Start with Sections 1–3 for scope and safety, Sections 4–11 for the design, Section 12 for research decisions, and Sections 13–15 for measurement, implementation ownership and acceptance.

## Execution boundary

Publishing these documents does not authorise application-code changes, production key operations, real-account testing, local-game modifications, merging implementation branches, building a release or deploying websites. Those actions require their own agreed scope. Keep one integration owner, use targeted worker validation and reserve broad final validation for the integrated, reviewed candidate.
