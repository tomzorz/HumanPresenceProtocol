# Human Presence Protocol

A local protocol for asking two questions no single tool answers today: **where is the human right now** (at the keyboard, over SSH, on a remote desktop, gone), and **where did the asking process come from** (an SSH session, a desktop launcher, a service manager). A per-user daemon answers over a local socket; clients use the answers to route their own behavior — open a GUI viewer when you're at the machine, serve a tunnel when you're not.

The full specification lives in [SPEC.md](SPEC.md). It covers the data model, the semantic rules that make independent implementations interchangeable, the JSON-RPC wire protocol, and the transport bindings (Unix domain sockets, Windows named pipes, and an optional token-authenticated TCP listener).

## Status

**HPP/1, draft 1.** The protocol is under active design and nothing is stable yet. Endpoint names carry the generation (`hpp1.sock`), so when generation 1 is declared stable, breaking changes will require a new generation rather than silently changing this one.

## Implementations

| Implementation | Language | Status |
|---|---|---|
| [Seance](https://github.com/tomzorz/seance) | C# / .NET | In development, reference implementation |

## Why a protocol and not just a program

The daemon is useful alone, but the interesting consumers are small: a shell alias, an editor plugin, a notification hook. Those get written in whatever language is at hand, which means the boundary has to be a boring wire format rather than a library. The spec exists so that `echo '{"jsonrpc":"2.0","id":1,"method":"presence.query"}' | nc -U $XDG_RUNTIME_DIR/hpp1.sock | jq` is a complete client, and so that a daemon written in Go or Rust answers exactly like the C# one.

## Non-goals

HPP is advisory, never authorization. Any process running as you can forge every signal it reports. It answers "where should output go", not "who is allowed". See section 9 of the spec before doing anything clever.
