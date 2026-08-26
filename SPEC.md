# Human Presence Protocol, generation 1 (HPP/1)

**Status:** draft 2
**Repository:** this repository is the normative home of the specification.

## 1. Introduction

This section is informative.

The Human Presence Protocol (HPP) lets a local process ask two questions about the machine it runs on:

- Where is the human right now? At the local seat, connected over SSH, driving a remote-desktop session, or absent.
- Where did the asking process come from? Spawned inside an SSH session, from a desktop launcher, inside a terminal multiplexer, or from a service manager.

A per-user daemon answers these questions over a local socket. Clients use the answers to route their own behavior. The motivating example: a helper that "opens a file" launches a local GUI viewer when the user sits at the machine, but serves the file over a tunnel when the request arrives through SSH.

The protocol deliberately avoids a boolean "is the user present" model. The interesting states are the ones a single boolean cannot express, for example an active SSH session while someone also sits at the desk. The central abstraction is the surface: one channel through which the user could currently interact with the machine. The daemon reports a list of surfaces with per-surface signals, and each client applies its own thresholds.

### 1.1 Non-goals

This section is informative.

- **Not a security boundary.** Every process running as the owning user can forge or influence these signals. HPP data is advisory and serves behavioral routing, never authorization.
- **Not a system-wide service.** One daemon serves one user login session. Surfaces of other users are out of scope (REQ-SCOPE-1).
- **Not an advisor.** The daemon reports where the user might be and how reachable each channel is. What a client does with that — which window to open, which channel to write to — is the client's decision. The daemon computes one fixed, deterministic reachability view (section 5.5) and carries no per-application policy.

## 2. Conformance

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, NOT RECOMMENDED, MAY, and OPTIONAL in this document are to be interpreted as described in BCP 14 (RFC 2119, RFC 8174) when, and only when, they appear in all capitals, as shown here. This specification uses only MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY.

Two conformance targets exist:

- A **conforming daemon** satisfies every MUST and MUST NOT that names the daemon.
- A **conforming client** satisfies every MUST and MUST NOT that names the client.

Requirements carry stable identifiers in the form `REQ-<AREA>-<n>`. A deleted identifier retires and is never reused.

## 3. Glossary

Each entry lists banned synonyms. Terms not listed here carry their plain English meaning.

### daemon (noun)
The per-user background process that implements the server side of HPP.
Do not use: server, service, agent.

### client (noun)
A process that connects to the daemon through an endpoint.
Do not use: consumer, caller (see caller context for the related concept).

### owning user (noun)
The user account whose login session runs the daemon.
Do not use: operator, current user.

### surface (noun)
A channel through which the owning user could currently interact with the machine.
Do not use: channel, session (see session for the platform concept).

### session (noun)
A platform-native login session, for example a logind session on Linux or a WTS session on Windows.
Do not use: logon, login (as a noun).

### local seat (noun)
The physical console of the machine: its directly attached displays and input devices.
Do not use: console (as a standalone noun), physical seat.

### endpoint (noun)
An address where the daemon accepts connections, for example a socket path, a pipe name, or a TCP address.
Do not use: address, socket (when the transport is not a socket).

### transport binding (noun)
The set of rules that maps the wire protocol onto one transport type (section 7).
Do not use: transport layer.

### generation (noun)
The major version of the protocol, encoded in endpoint names (section 8).
Do not use: major version, protocol version.

### snapshot (noun)
The full list of surfaces, the active pin, and the capabilities, as observed at one instant.
Do not use: state dump.

### event (noun)
A notification that the daemon sends to a subscribed client after presence state changes.
Do not use: signal, message (as a specific term).

### pin (noun)
An override, set by the owning user, that replaces derived presence state in recommendations until the pin expires or is removed.
Do not use: override, force mode.

### caller context (noun)
The daemon's description of the process on the other end of one connection.
Do not use: peer info, client context.

### reachability (noun)
The daemon's computed, ordered view of which surfaces can reach the owning user now (sections 4.8 and 5.5).
Do not use: recommendation, presence state.

### client name (noun)
The identifier that the daemon configuration associates with one TCP token.
Do not use: token name, client id.

### token (noun)
A secret string that authenticates a TCP client and maps to exactly one client name.
Do not use: key, secret, credential.

## 4. Data model

This section defines the objects that appear on the wire. Field names are camelCase. Enumeration values are camelCase strings. Timestamps are ISO 8601 strings in UTC. Durations on the wire are integer milliseconds.

REQ-DM-1: The daemon MUST encode every object in this section as JSON with the field names and value sets defined here.

REQ-DM-2: The client MUST ignore object members that this specification does not define. This rule permits additive extension within a generation (section 8).

### 4.1 Tri-state values

A tri-state field takes exactly one of the values `"yes"`, `"no"`, or `"unknown"`.

REQ-TRI-1: If the platform cannot determine the value of a tri-state field, the daemon MUST report `"unknown"`.

REQ-TRI-2: The daemon MUST NOT substitute a default value for a signal that the platform cannot determine. This rule also applies to nullable fields: an unreadable idle time is `null`, not `0`.

The bug this prevents, stated informatively: macOS has no supported API that detects an active screen-sharing session. A daemon that reports "not remote" in that case misleads every client into opening windows that nobody can see. `"unknown"` moves the decision to the client, which knows its own tolerance for uncertainty.

### 4.2 Reliability

A reliability value takes exactly one of `"unreliable"`, `"indicative"`, or `"authoritative"`:

- `"authoritative"`: the platform is the source of truth for this signal.
- `"indicative"`: the signal is usually correct, and known failure modes exist.
- `"unreliable"`: the signal is present, and known cases report wrong values silently.

### 4.3 Surface

A surface object has the following members:

| Member | Type | Meaning |
|---|---|---|
| `kind` | string | Exactly one of `"localSeat"`, `"ssh"`, `"remoteDesktop"`, `"unknown"`. |
| `sessionId` | string | The platform-native session identifier. |
| `connected` | boolean | The session exists and is attached. |
| `onConsole` | tri-state | The session owns the local seat, so pixels this session draws are visible. |
| `locked` | tri-state | The screen lock of this session is engaged. |
| `idleMs` | integer or null | Milliseconds since input last arrived in this session. `null` when unreadable. |
| `remoteHost` | string or null | The remote peer, for `"ssh"` and `"remoteDesktop"` kinds. |
| `protocol` | string or null | The remote protocol, for example `"ssh"`, `"rdp"`, or `"vnc"`. |
| `displayServer` | string or null | `"wayland"`, `"x11"`, or `null` when not applicable. |
| `displayPowered` | tri-state | The display of this surface is powered. |
| `humanPresent` | tri-state | A human-presence sensor detects a person. Usually `"unknown"`. |
| `confidence` | object | Per-signal reliability (section 4.4). |
| `observedAt` | string | The UTC timestamp of the observation. |

REQ-SURF-1: The daemon MUST derive `kind` from current platform signals at each snapshot.

REQ-SURF-2: The daemon MUST NOT reuse a `kind` value from an earlier snapshot when current signals contradict it.

REQ-SURF-3: `locked`, `onConsole`, and `idleMs` describe independent facts. The daemon MUST NOT derive the value of one of these fields from another of these fields.

Section 5.2 explains why REQ-SURF-3 exists and how clients interpret the three fields together.

### 4.4 Confidence

A confidence object has the members `locked`, `onConsole`, `idle`, and `kind`. Each member is a reliability value (section 4.2).

REQ-CONF-1: The daemon MUST set each confidence member from the actual signal source in use, not from a per-platform constant. Example: a Linux daemon that reads a compositor-provided lock hint reports `"unreliable"`, and the same daemon reports `"authoritative"` for `onConsole` because logind is the source.

REQ-CONF-2: If the daemon reports a tri-state field as `"unknown"` because no source exists, the daemon MUST report the matching confidence member as `"unreliable"`.

### 4.5 Pin

A pin object has the following members:

| Member | Type | Meaning |
|---|---|---|
| `kind` | string | The pinned surface kind: exactly one of `"localSeat"`, `"ssh"`, `"remoteDesktop"`. |
| `pinnedAt` | string | The UTC timestamp at which the pin was set. |
| `expiresAt` | string or null | The UTC expiry timestamp. `null` means the pin lasts until removed. |

### 4.6 Capabilities

A capabilities object has the following boolean members: `canEnumerateSessions`, `canDetectRemoteDesktop`, `canReadIdle`, `canReadLockState`, and `hasHumanPresenceSensor`.

REQ-CAP-1: The daemon MUST NOT report a capability as `true` when the running platform code cannot deliver the signal.

REQ-CAP-2: The client SHOULD read the capabilities before the client trusts a signal field, because a `false` capability predicts permanent `"unknown"` or `null` values for the matching fields.

### 4.7 Caller context

A caller context object describes the process on the other end of one connection:

| Member | Type | Meaning |
|---|---|---|
| `pid` | integer or null | The process id of the connected client. |
| `userId` | string or null | The platform-native user identifier: a decimal uid on Linux and macOS, a SID string on Windows. |
| `executablePath` | string or null | The executable path of the connected client. |
| `originKind` | string | The surface kind the client originated from, inferred from process ancestry. `"unknown"` when inference fails. |
| `originSessionId` | string or null | The session identifier the client belongs to, when resolvable. |
| `ancestry` | array | Objects of the form `{ "pid": integer, "name": string }`, ordered from the client toward the root. |
| `verifiedBy` | string | Exactly one of `"kernel"` or `"token"` (see below). |
| `clientName` | string or null | The client name of the authenticated token. `null` on kernel-verified transports. |

`verifiedBy` states how the daemon established the identity:

- `"kernel"`: the operating system reported the peer credentials of the connection. Used by the Unix socket binding and the named pipe binding.
- `"token"`: the client authenticated with a token (section 7.4). The daemon knows the client name and knows nothing kernel-verified about the process.

REQ-CTX-1: The daemon MUST resolve caller identity from transport peer credentials where the transport provides them.

REQ-CTX-2: The daemon MUST NOT resolve caller identity from environment variables. Environment variables are lost or wrong across `sudo`, `su`, and terminal multiplexer reattachment.

REQ-CTX-3: On token-verified connections, the daemon MUST set `pid`, `userId`, `executablePath`, and `originSessionId` to `null`, MUST set `ancestry` to an empty array, and MUST set `originKind` to `"unknown"`.

### 4.8 Reachability

A reachability object is the daemon's computed answer to one question: which surfaces can reach the owning user right now. The daemon states facts and their ordering; the client decides what to do with them.

| Member | Type | Meaning |
|---|---|---|
| `reachable` | array | Surface objects (section 4.3), ordered from most to least likely to reach the user (section 5.5). The array can be empty. |
| `blockedBy` | string or null | `null` when `reachable` is not empty. Otherwise exactly one of `"offConsole"`, `"locked"`, `"absent"`, `"pinnedUnavailable"` (section 5.5 defines the selection). |

The empty case exists because fast user switching produces a state the local-versus-remote question cannot answer: a live session that is off-console, with no remote channel either. Section 5.5 defines the client obligations.

## 5. Semantic requirements

Wire-level conformance alone does not make implementations interchangeable. Two daemons that parse the same JSON and answer the same situation differently defeat the purpose of a shared protocol. This section defines the behavior.

### 5.1 Truthfulness of unknowns

REQ-TRUTH-1: The daemon MUST prefer reporting `"unknown"` over guessing, for every tri-state field.

REQ-TRUTH-2: The daemon MAY infer a field value from indirect signals. If the daemon infers, the daemon MUST lower the matching confidence member to `"indicative"` or `"unreliable"`.

Example, stated informatively: detecting a VNC server attached to an existing local display by scanning listening ports is inference. Reporting the resulting surface with `confidence.kind = "unreliable"` is correct. Reporting it as authoritative is not.

### 5.2 Independence of locked, on-console, and idle

The three fields answer different questions:

| Field | Question |
|---|---|
| `locked` | Can the user interact right now? |
| `onConsole` | Would pixels drawn by this session ever be seen? |
| `idleMs` | How long since input arrived in this session? |

Fast user switching makes the distinction load-bearing. Several users can hold live sessions at the same time, and exactly one session owns the local seat. Two divergent combinations follow:

- Locked and on-console: a window opened now sits behind the lock screen and appears after unlock.
- Unlocked and off-console: a window opened now renders into a desktop nobody watches.

REQ-IND-1: The client MUST NOT treat `locked = "no"` alone as proof that the user can see this session.

REQ-IND-2: The client MUST read `idleMs` together with `onConsole`. An off-console session keeps counting idle time from the last input before the switch away, so a recently abandoned session looks active by idle time alone.

REQ-IND-3: Platforms differ on which of `locked` and `onConsole` is trustworthy, in both directions (Appendix A). The client SHOULD prefer the field whose confidence member is higher, because a fixed preference is wrong on at least one platform.

### 5.3 Debounce

REQ-DEB-1: The daemon MUST debounce idle-driven changes before the daemon publishes an event for them. Per-client debouncing is the failure this prevents: without daemon-side debounce, every client flaps independently when the user jiggles the mouse.

REQ-DEB-2: The debounce window SHOULD be 5 seconds by default. The daemon MAY make the window configurable.

REQ-DEB-3: The daemon MUST publish each of the following changes without debounce:

- a lock state change,
- an on-console change,
- a session connect,
- a session disconnect,
- a pin change.

These changes are discrete and unambiguous. A user who switches back to their session expects deferred work to resume at once.

### 5.4 Pins

REQ-PIN-1: The daemon MUST support pins. Pins are not optional, because detection heuristics fail (Appendix A) and the pin is the escape hatch.

REQ-PIN-2: While a pin is active, the daemon MUST apply the pin filter of section 5.5 (step 2) when the daemon computes reachability.

REQ-PIN-3: A pin MUST NOT alter the reported surfaces. `presence.query` reports derived surfaces plus the pin object, and the client sees both.

REQ-PIN-7: A client that routes output from raw `presence.query` data, without `presence.reach`, MUST honor the active pin in the same way: only surfaces of the pinned kind are candidates. The pin is the owning user's correction for wrong heuristics, and a client that ignores the pin defeats that correction.

REQ-PIN-4: After the expiry timestamp of a pin passes, the daemon MUST stop applying the pin.

REQ-PIN-5: When a new pin is set while a pin is active, the daemon MUST replace the active pin with the new pin.

REQ-PIN-6: The daemon MAY drop the active pin when the daemon restarts.

REQ-PIN-8: The daemon MUST accept `presence.pin` and `presence.unpin` from every authenticated connection, including token-verified TCP connections. The pin-from-elsewhere case, for example `presence pin remote` sent from another machine over an overlay network, is a primary use of the TCP binding. Token revocation is the remedy for a misbehaving remote client.

Pin scope, stated informatively: a pin belongs to the daemon that holds it. On platforms with one daemon per user this means one pin per user per machine. On Windows, where one daemon runs per session (section 7.3), each session's daemon holds its own pin, and a pin set in the console session does not exist for the same user's RDP session daemon.

### 5.5 Reachability

`presence.reach` computes a view over the same facts that `presence.query` reports. The daemon adds no policy: the procedure below is fixed and deterministic, and it is the same for every client. A client that wants a different reading reads the raw surfaces instead.

REQ-RCH-1: The daemon MUST compute the reachability object with the following procedure. Two daemons that observe the same snapshot, the same pin, and the same caller context MUST produce the same reachability object.

1. Start with every surface of the current snapshot as a candidate.
2. If a pin is active, remove each candidate whose `kind` differs from the pinned kind. If this step empties the candidate list, return `reachable: []` with `blockedBy: "pinnedUnavailable"`.
3. Remove each candidate whose `connected` is `false`.
4. Remove each candidate of kind `"localSeat"` or `"remoteDesktop"` whose `onConsole` is `"no"` or whose `locked` is `"yes"`. A value of `"unknown"` removes nothing: exclusion needs a definite negative, in the spirit of REQ-TRI-1.
5. Order the remaining candidates:
   1. A candidate whose `sessionId` equals the caller's `originSessionId` comes first.
   2. If the caller's `originKind` is not `"unknown"`, a candidate whose `kind` equals the caller's `originKind` comes next. An `originKind` of `"unknown"` places no candidate in this group.
   3. Within each group, order the candidates by ascending `idleMs`, and place `null` values last.
   4. Break each remaining tie by ascending `sessionId` in byte order.
6. If no candidate remains, select `blockedBy` with the first match: `"offConsole"` if step 4 removed at least one candidate for `onConsole`, else `"locked"` if step 4 removed at least one candidate for `locked`, else `"absent"`.

The ordering encodes one opinion, stated informatively: a request that arrived over a channel gets answered over that channel first. When both the local seat and an SSH session are live and the caller came through SSH, the SSH surface sorts first, and the client still sees the local seat listed after it and chooses freely.

Step 5.2 excludes an `originKind` of `"unknown"`, stated informatively: an unclassified caller and an unclassified surface have nothing in common, and matching one to the other would sort the least understood surface above the local seat. Token-verified connections always report `originKind` as `"unknown"` (REQ-CTX-3), so this case is ordinary rather than rare.

REQ-RCH-2: The client MUST handle an empty `reachable` array. A client that assumes a usable surface always exists does the wrong thing after the first user switch.

REQ-RCH-3: The daemon MUST return an empty reachability object as a JSON-RPC result, not as a JSON-RPC error. Unreachable is an answer, not a failure.

REQ-RCH-4: After an empty result, the client SHOULD subscribe and act after the next relevant event, because the blocked state usually ends with a transition back on-console.

When `reachable` is empty and the client is attached to a terminal, emitting an OSC sequence into its own terminal is the best client-side bet, because the terminal renders wherever the user's eyes are (Appendix B). The channel choice belongs to the client; the wire carries facts only.

### 5.6 Scope of reported surfaces

REQ-SCOPE-1: The daemon MUST report only surfaces that belong to the owning user.

Platform session APIs expose other users' sessions. Reporting them crosses the trust framing of a per-user advisory service and serves no client need this specification recognizes.

## 6. Wire protocol

### 6.1 Framing

REQ-RPC-1: The daemon and the client MUST exchange JSON-RPC 2.0 messages.

REQ-RPC-2: Each message MUST be one JSON object, encoded in UTF-8, terminated by exactly one line feed (byte `0x0A`).

REQ-RPC-3: A message MUST NOT contain an unescaped line feed. JSON string escaping already guarantees this for conforming encoders.

REQ-RPC-4: The daemon MUST accept an optional carriage return (byte `0x0D`) immediately before the line feed, because interactive tools on Windows emit CRLF.

This framing keeps the protocol shell-scriptable. Two lines into `nc -U` are a complete authenticated exchange, and `presence query --json | jq` works without a protocol library.

Three restrictions keep minimal implementations conforming:

REQ-RPC-5: The client MUST NOT send a JSON-RPC batch (a top-level array). The daemon MAY reject a batch with error code `-32600` without processing it.

REQ-RPC-6: A request id MUST be a JSON number or a JSON string.

REQ-RPC-7: When a request carries parameters, the client MUST pass them as a JSON object (by-name parameters). Positional parameter arrays are not used in this generation.

### 6.2 Methods

The daemon exposes the following methods. Parameters are passed as a JSON object (named parameters). Methods without parameters accept an omitted or empty `params`.

| Method | Params | Result |
|---|---|---|
| `presence.query` | none | `{ "surfaces": [...], "pin": object or null, "capabilities": object, "generatedAt": string, "serverInfo": object }` |
| `presence.context` | none | A caller context object (section 4.7). |
| `presence.reach` | none | A reachability object (section 4.8). |
| `presence.subscribe` | none | The same object shape as the `presence.query` result. Events follow as notifications (section 6.3). |
| `presence.pin` | `{ "kind": string, "durationMs": integer or null }` | The resulting pin object. |
| `presence.unpin` | none | `{}` (acknowledgment). |
| `presence.authenticate` | `{ "token": string }` | `{ "clientName": string }`. TCP binding only (section 7.4). |

REQ-MTH-1: The daemon MUST implement every method in the table above, with the exception that `presence.authenticate` is REQUIRED only when the daemon enables a TCP endpoint. Deviation from rule N1 noted: "REQUIRED" here abbreviates "MUST implement"; the sentence structure needs the participle.

REQ-MTH-2: `serverInfo` is an object of the form `{ "name": string, "version": string }` describing the daemon implementation. The daemon MUST treat `serverInfo` as informative. The client MUST NOT change protocol behavior based on `serverInfo`. Generation negotiation happens through endpoint names (section 8).

REQ-MTH-3: In `presence.pin`, a `durationMs` of `null`, or an omitted `durationMs`, means the pin lasts until removed.

REQ-MTH-4: The daemon MUST reject a `presence.pin` call whose `kind` is not one of `"localSeat"`, `"ssh"`, `"remoteDesktop"` with JSON-RPC error code `-32602` (invalid params).

REQ-MTH-5: The result of `presence.subscribe` MUST be the current snapshot, in the same object shape as the `presence.query` result. This closes the race between a query and a following subscribe: no transition can fall between the two, because the subscription and its baseline state are one exchange.

### 6.3 Events

REQ-EVT-1: After the daemon answers `presence.subscribe`, the daemon MUST send a JSON-RPC notification with method `presence.event` to that connection after each published state change (section 5.3 defines which changes are published and when).

REQ-EVT-2: An event notification carries these params: `{ "reason": string, "surfaces": [...], "pin": object or null, "generatedAt": string }`. The `reason` is exactly one of `"surfaceConnected"`, `"surfaceDisconnected"`, `"lockChanged"`, `"consoleChanged"`, `"idleChanged"`, `"pinChanged"`.

REQ-EVT-3: Each event MUST carry the full current surface list, not a delta. Full snapshots make lost or reordered notifications harmless: the client always re-evaluates from complete state.

REQ-EVT-4: A subscription MUST end when its connection closes. No unsubscribe method exists in this generation; a client that wants to stop closes the connection, or opens a dedicated connection for its subscription.

REQ-EVT-5: The daemon MUST send an `"idleChanged"` event only after input arrives in a session that received no input for at least the debounce window (section 5.3). Rising idle time MUST NOT generate events.

The asymmetry in REQ-EVT-5 is deliberate. Idle time is a counter that rises every second, so "the value changed" cannot be a trigger. The discrete moment worth pushing is the user coming back. For the other direction, the client computes the current idle time of a surface locally, as `idleMs` plus the time elapsed since `generatedAt`, and runs its own timer for "the user has been away for X". Thresholds stay client-owned, which section 1 promises.

### 6.4 Errors

The daemon uses standard JSON-RPC 2.0 error codes for malformed requests, unknown methods, and invalid parameters. This specification defines two additional codes in the implementation-defined server error range:

| Code | Name | Meaning |
|---|---|---|
| `-32000` | `authRequired` | The connection is a TCP connection and is not authenticated. |
| `-32001` | `authFailed` | The presented token matches no configured token. |

REQ-ERR-1: On an unauthenticated TCP connection, the daemon MUST reject every request except `presence.authenticate` with code `-32000`.

REQ-ERR-2: After a failed `presence.authenticate`, the daemon MUST return code `-32001` and MUST keep the connection unauthenticated. The daemon MAY close the connection after repeated failures.

## 7. Transport bindings

### 7.1 Endpoint discovery

REQ-DISC-1: The client MUST check the environment variable `HPP_ENDPOINT` before the client uses a platform default endpoint.

REQ-DISC-2: `HPP_ENDPOINT` holds exactly one endpoint string in one of these forms:

- `unix:<absolute path>` — a Unix domain socket path, for example `unix:/run/user/1000/hpp1.sock`.
- `pipe:<name>` — a Windows named pipe name without the `\\.\pipe\` prefix, for example `pipe:hpp1-2`.
- `tcp:<host>:<port>` — a TCP endpoint, for example `tcp:127.0.0.1:18512`. An IPv6 host is bracketed: `tcp:[::1]:18512`.

REQ-DISC-3: If `HPP_ENDPOINT` is set and the connection fails, the client MUST report the failure. The client MUST NOT silently fall back to a platform default, because the variable expresses explicit intent, for example a test harness or a relay.

The endpoint name encodes the generation (section 8): `hpp1` is generation 1.

### 7.2 Unix domain socket binding (Linux, macOS)

REQ-UDS-1: The daemon MUST listen on `$XDG_RUNTIME_DIR/hpp1.sock` when `XDG_RUNTIME_DIR` is set.

REQ-UDS-2: When `XDG_RUNTIME_DIR` is not set, the daemon MUST listen on `/tmp/hpp1-<uid>.sock`, where `<uid>` is the decimal uid of the owning user.

REQ-UDS-3: The daemon MUST set the socket file mode to `0600`.

REQ-UDS-4: The daemon MUST resolve caller credentials from the socket peer: `SO_PEERCRED` on Linux, `LOCAL_PEERCRED` on macOS. These connections report `verifiedBy = "kernel"`.

REQ-UDS-5: The daemon SHOULD remove a stale socket file left by a previous daemon instance before binding, because a stale file otherwise blocks startup. The daemon MUST NOT remove a socket that another live process holds open.

### 7.3 Named pipe binding (Windows)

REQ-PIPE-1: The daemon MUST listen on the named pipe `\\.\pipe\hpp1-<sessionId>`, where `<sessionId>` is the decimal Windows session id of the daemon's own session.

REQ-PIPE-2: The daemon MUST restrict the pipe security descriptor to the owning user.

REQ-PIPE-3: The daemon MUST resolve the caller process id with `GetNamedPipeClientProcessId`. These connections report `verifiedBy = "kernel"`.

A client that does not know its own session id can query it from the operating system; the session id of a process is cheap to read on every Windows API surface. This binding is per-session by design: one daemon per interactive session (section 1.1).

### 7.4 TCP binding (optional)

The TCP binding exists for clients that native IPC cannot reach, for example a process inside a WSL distribution asking the Windows host daemon. Native transports get per-user isolation from the operating system; TCP gets nothing from the operating system, and the token requirement restores the boundary.

REQ-TCP-1: The daemon MAY provide a TCP endpoint. The daemon MUST NOT enable a TCP endpoint by default.

REQ-TCP-2: When the TCP endpoint is enabled, the daemon MUST require token authentication on every TCP connection, including loopback connections.

REQ-TCP-3: The first request on a TCP connection MUST be `presence.authenticate`. Section 6.4 defines the rejection behavior for every other case.

REQ-TCP-4: The daemon configuration maps each token to exactly one client name. The daemon MUST accept a token only when the token exactly matches one configured token.

REQ-TCP-5: The daemon SHOULD bind the TCP endpoint to a loopback address by default, because the wire is plaintext (section 9). The daemon MAY bind to other addresses when configured to do so.

REQ-TCP-6: The default TCP port is 18512. The daemon MAY make the port configurable. (18512 is `0x4850`, the ASCII bytes `HP`.)

REQ-TCP-7: Authenticated TCP connections report `verifiedBy = "token"` and carry the client name (section 4.7, REQ-CTX-3).

## 8. Versioning

The generation is the only cross-generation compatibility signal, and the generation lives in the endpoint name: `hpp1.sock`, `\\.\pipe\hpp1-<sessionId>`. There is no version handshake and no version field.

REQ-VER-1: A change that breaks a conforming generation-1 client MUST NOT ship under a generation-1 endpoint name. Such a change defines generation 2, with endpoint names containing `hpp2`.

REQ-VER-2: Within a generation, this specification MAY add optional object members, event reasons, and methods. REQ-DM-2 (clients ignore unknown members) makes these additions safe.

REQ-VER-3: A generation-2 daemon MAY serve generation 1 on the generation-1 endpoint at the same time. Clients discover generations by endpoint presence: probe the highest known generation first, then fall back.

This document versions itself as "HPP/1, draft N" until the editor declares generation 1 stable. After that declaration, REQ-VER-1 governs every change.

## 9. Security considerations

**HPP is not a security boundary.** Every process running as the owning user can connect to the daemon, read every signal, set pins, and forge the underlying platform signals themselves. Clients MUST NOT use HPP data for authorization decisions. The protocol answers "where should output go", never "who is allowed".

Within that framing, the transports differ in what they guarantee:

- The Unix socket binding and the named pipe binding restrict connections to the owning user through file mode and security descriptor (REQ-UDS-3, REQ-PIPE-2), and identify callers through kernel-reported credentials.
- The TCP binding restricts connections through tokens (REQ-TCP-2). The wire is plaintext: an on-path observer reads presence data and can capture tokens. Deployments that cross machine boundaries SHOULD run the TCP binding over an encrypted overlay network, because this specification defines no transport encryption in generation 1.

Presence data is behavioral metadata about a person. Even advisory, it reveals when someone is at their desk. Operators who expose a TCP endpoint beyond loopback take on that disclosure and scope tokens accordingly.

Caller context on kernel-verified transports reports facts the kernel asserts (pid, uid). Ancestry walking and origin inference read process tables that the owning user can manipulate; caller context is therefore as advisory as everything else.

## Appendix A. Platform signal landscape (informative)

This appendix is informative. It records the known reliability characteristics that motivated the confidence model (section 4.4). Implementers of new daemons and clients benefit from the same map.

Each platform has one trustworthy field and one flaky one, and the platforms disagree on which is which:

| Platform | `locked` | `onConsole` |
|---|---|---|
| Windows | Strong. Lock-on-switch is the default, but group policy can change it. | Strong (WTS connect state). |
| Linux | Weak. `LockedHint` is compositor-reported and many compositors never set it. | Strong. logind `Active` is authoritative. |
| macOS | Weak. A fast-user-switch away does not reliably set the lock flag. | Strong (`kCGSSessionOnConsoleKey`). |

Known blind spots for `kind` detection:

- Windows: `WTSClientProtocolType` and `SM_REMOTESESSION` detect RDP only. TeamViewer, AnyDesk, VNC, and RustDesk drive the physical console session and look local. Process-name detection is possible and is inference (REQ-TRUTH-2).
- Linux: xrdp and Xvnc appear as separate remote sessions. A VNC server attached to an existing local display (for example `x11vnc`) does not. Port-scan detection is inference.
- macOS: no supported API detects an active Screen Sharing session. Absence of the helper process means `"unknown"`, not `"no"` (REQ-TRI-1).

Idle granularity varies: Windows reports milliseconds, X11 reports milliseconds, Wayland compositors report a coarse boolean idle hint through logind, and macOS reports seconds. `idleMs` therefore carries platform-dependent precision, and `confidence.idle` reflects it.

## Appendix B. Prior art (informative)

- **VS Code Remote**: the `code` CLI on a remote host talks to a socket forwarded back over SSH, so a server-side "open file" surfaces in the local window. Solves one slice of this problem for one application.
- **lemonade**: a small Go daemon providing open, copy, and paste RPC over SSH port forwarding. Closest in spirit; single-consumer, not subscribable.
- **OSC escape sequences**: OSC 52 (clipboard), OSC 8 (hyperlinks), OSC 9 and OSC 777 (notifications). The terminal emulator acts as the presence oracle, because it renders wherever the user's eyes are, and the sequence follows the connection chain automatically. This is why section 5.5 points tty-attached clients at OSC when reachability comes up empty: when classification fails, the terminal that delivered the request is the best guess for where the user is, and the client itself owns that terminal.

The gap none of these fill, and the reason HPP exists: a general, multi-client, subscribable presence service with caller-origin context.

## Appendix C. Example exchange (informative)

A client connects over the Unix socket, queries, and subscribes. Lines marked `→` flow client to daemon, lines marked `←` flow daemon to client. Line breaks inside JSON are for readability only; on the wire each message is one line.

```
→ {"jsonrpc":"2.0","id":1,"method":"presence.query"}
← {"jsonrpc":"2.0","id":1,"result":{
     "surfaces":[
       {"kind":"localSeat","sessionId":"2","connected":true,
        "onConsole":"yes","locked":"no","idleMs":4210,
        "remoteHost":null,"protocol":null,"displayServer":"wayland",
        "displayPowered":"yes","humanPresent":"unknown",
        "confidence":{"locked":"unreliable","onConsole":"authoritative",
                      "idle":"indicative","kind":"authoritative"},
        "observedAt":"2026-08-17T14:03:07Z"},
       {"kind":"ssh","sessionId":"7","connected":true,
        "onConsole":"no","locked":"unknown","idleMs":180424,
        "remoteHost":"10.0.0.12","protocol":"ssh","displayServer":null,
        "displayPowered":"unknown","humanPresent":"unknown",
        "confidence":{"locked":"unreliable","onConsole":"authoritative",
                      "idle":"indicative","kind":"authoritative"},
        "observedAt":"2026-08-17T14:03:07Z"}
     ],
     "pin":null,
     "capabilities":{"canEnumerateSessions":true,"canDetectRemoteDesktop":true,
                     "canReadIdle":true,"canReadLockState":false,
                     "hasHumanPresenceSensor":false},
     "generatedAt":"2026-08-17T14:03:07Z",
     "serverInfo":{"name":"seance","version":"0.1.0"}}}
→ {"jsonrpc":"2.0","id":2,"method":"presence.subscribe"}
← {"jsonrpc":"2.0","id":2,"result":{ ...the current snapshot, same shape as the query result... }}
← {"jsonrpc":"2.0","method":"presence.event","params":{
     "reason":"surfaceDisconnected",
     "surfaces":[ ...the full current list... ],
     "pin":null,
     "generatedAt":"2026-08-17T14:09:31Z"}}
```

A TCP client authenticates first:

```
→ {"jsonrpc":"2.0","id":1,"method":"presence.authenticate","params":{"token":"..."}}
← {"jsonrpc":"2.0","id":1,"result":{"clientName":"wsl"}}
→ {"jsonrpc":"2.0","id":2,"method":"presence.reach"}
← {"jsonrpc":"2.0","id":2,"result":{"reachable":[],"blockedBy":"offConsole"}}
```
