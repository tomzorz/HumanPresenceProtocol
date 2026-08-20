# Before wider publicity: add JSON Schemas and golden transcripts

**Created:** 2026-08-20
**Status:** parked, gates announcing the spec as implementable

Decided during the initial design session (Tamás, 2026-08-20): the spec repo gets machine-readable conformance artifacts, but only after the spec survives review and Seance's Core exists to validate them against. Vectors written before any implementation tend to encode the spec's typos.

To add when unparked:

- `schemas/` — JSON Schema files for the wire objects (surface, confidence, pin, capabilities, caller context, reachability, event params, method results). Informative, not normative; the prose spec stays authoritative.
- `transcripts/` — golden NDJSON request/response transcripts (canonical exchanges: query, subscribe + event, pin lifecycle, TCP authenticate flow, error cases like authRequired). Implementations replay these as conformance tests.

Other publish-time reminders from the same session:

- Verify port 18512 against the IANA service-name registry (chosen as 0x4850 = "HP"; believed unsquatted, never verified).
- Update the README implementations table if Seance has moved/renamed by then.
