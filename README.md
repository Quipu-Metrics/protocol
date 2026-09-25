# Quipu Metrics Protocol

The wire protocol spoken between a Quipu Metrics client and the Quipu Metrics
backend at `quipumetrics.org`.

This repository contains no runnable code. It is the contract that every client
implementation and the backend must agree on.

## Contents

| Path | Purpose |
|------|---------|
| `SPEC.md` | The normative specification |
| `schema/submission.schema.json` | JSON Schema (draft 2020-12) for the request body |
| `examples/` | Valid example payloads |

## Who this is for

Anyone writing a Quipu Metrics client. The reference client is Java
(`Quipu-Metrics/client`), but the protocol is language agnostic. A client in
Rust, Go, PHP, or Lua that follows `SPEC.md` will work against the production
backend without any coordination with the maintainers.

## Versioning

Two numbers live here and they move independently.

| Number | What it is | Changes when |
|--------|------------|--------------|
| `v1` | The wire version, in the request path | Never, within a major version |
| `VERSION` | This repository's release | Any edit to the specification |

An implementer says "implements Quipu protocol v1, specification 0.1.0". The
wire version tells the backend how to read a request. The release tells a human
which wording was implemented.

Within a major wire version, the backend only ever adds optional fields. A
client written against `v1` keeps working until `v1` is retired, and retirement
is announced at least six months ahead.

Breaking changes get a new major wire version. Both versions run side by side
during the transition.

## License

The specification is released under CC0 1.0. Implement it freely, including in
closed source software, with no attribution requirement.
