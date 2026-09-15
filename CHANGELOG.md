# Changelog

All notable changes to ollama-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `ollpull` — `/api/pull` as events a caller pumps: `OllPullEvent` with
  the in-band failure as a variant, `OllPullState` as the accumulator,
  and `overall`, the whole-download figure the stream never sends.
  `/api/push` and `/api/create` through the same pump.
- `ollnd` — the newline-delimited reader, feed and drain, framing on the
  raw newline between objects.
- `ollgen` — `/api/generate` and `/api/chat`: `OllKeepAlive`,
  `OllOptions` with the whole sampling set, `raw`, the `context` array,
  the split prefill and decode timings, and `check`.
- `ollmodel` — a model reference parsed rather than a string, the
  Modelfile as a value, `/api/tags`, `/api/show` with its capability
  list, and `/api/ps` with its VRAM figure.
- `ollembed` — both embedding endpoints behind one request value, with
  the dimension measured rather than read from metadata.
- `ollapi` — the client, every call effect-polymorphic over
  llm-client-nv's `LlmcTransport[e]`, plus `preload` and `unload`.
- `ollfault` — eleven variants, with `wants_pull` as the question no
  hosted API has.

### Known

- **`OllPullEvent` is the load-bearing interface.**  A pull is minutes
  long, and this endpoint reports its own failures inside a 200 — so a
  client that checked the status reports a completed download of
  nothing.  The failure is a variant of the event, which a drain loop
  cannot walk past.
- **The progress numbers are per-layer and reset**, so a bar drawn from
  them jumps back to zero once per blob.  `overall` is the whole
  download, and its expected total grows as blobs are discovered.
- **There is no blocking `pull`.**  `pull_open` and `pull_next` are the
  shape.
- **Newline-delimited JSON, not server-sent events.**  The same server
  streams `/v1` as SSE and `/api/*` as NDJSON, and the wrong reader on
  either buffers forever with no error.
- **This package declares no transport**: llm-client-nv's
  `LlmcTransport[e]`, so a program holding both clients configures one
  byte pipe.
- **Four things live only here** and each is a reason to run a model
  locally: `keep_alive`, the full `options` set, `raw`, and the split
  prefill/decode timings.  `num_ctx` is the one whose wrong value is
  silent — the server truncates from the front and says nothing.
- **Two management calls have no hosted counterpart**: `/api/ps`, whose
  VRAM figure is a tenfold speed difference invisible elsewhere, and
  `/api/show`, whose capability list answers "can this model be sent
  tools" before a 400 does.
- **A 404 here is repairable** and a hosted provider's is not, which is
  why `OllModelNotPulled` is its own variant and `wants_pull` is a
  published question.
- **Nothing normalises an embedding.**  The server does not say whether
  a model's vectors are unit vectors, and normalising on the way in
  would collapse `embsim.dot` and `embsim.cosine` for a caller who
  needed them apart.
- **Two dependencies**: llm-client-nv (`host`, a path dependency while
  the two are staged together) and prompt-nv (`core`).
- **No device claim.**  The package is `host`.
- **One toolchain defect filed** against what it took to write this:
  a wrong-arity call to a generic function aborts the compiler with an
  OCaml exception instead of reporting `E2002`.
