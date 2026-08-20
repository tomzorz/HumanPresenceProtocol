# Changelog

All notable changes to the specification are recorded here. Spec releases are tagged in git; the endpoint generation (`hpp1`) only changes on breaking revisions, per section 8 of the spec.

## HPP/1 draft 1 — 2026-08-20

Initial draft. Defines:

- The surface data model: tri-state signals, per-signal confidence, independence of `locked` / `onConsole` / `idleMs`.
- Caller context resolved from kernel peer credentials, with the token-verified degradation for TCP.
- Semantic requirements: truthfulness of unknowns, daemon-side debounce, pin semantics, the deterministic reachability procedure (`presence.reach`), own-user surface scope.
- Wire protocol: JSON-RPC 2.0, newline-delimited framing (no batches, by-name params), `presence.*` methods, subscribe returning the baseline snapshot, full-snapshot events with reset-only idle triggers.
- Transport bindings: Unix domain socket, Windows named pipe, optional token-authenticated TCP (default port 18512, loopback by default).
- Versioning by endpoint generation; no handshake.
