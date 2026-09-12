# ollama-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

The local model server's own API: `/api/generate` and `/api/chat` with
the sampling options and the residency the OpenAI-compatible endpoint
has no spelling for, `/api/embed`, `/api/tags`, `/api/show`, `/api/ps`,
and `/api/pull` with the per-layer progress a caller pumps.  Model
names and the Modelfile as values.

It is the second half of llm-client-nv rather than a second client: the
transport, the provider value and the tool-call vocabulary all come
from there, and what is here is what that package has no place for.

## Adding it, and checking it

```bash
novo pkg add ollama-nv        # into your novo.toml
novo pkg build                # type- and effect-check the package
novo test --isolate tests/ollpull_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: ollama-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use ollapi
use ollfault
use ollgen
use ollpull

// Pull a model, showing progress, then generate with it.
// `[io, net, time, async]` because the transport here is the standard
// library's HTTP client; over `LlmcTlsHttp` the same code is `[net]`.
fn ready(t: LlmcStdHttp, model: Str) -> Result<Str, OllFault> [io, net, time, async]
    var s = ollapi.pull_open(t, model, false)!

    while ollpull.succeeded(s.state) == false
        let step = ollapi.pull_next(t, s)!
        s = step.stream
        // The whole download, not the layer: see below.
        let total = ollpull.overall(s.state)
        println("${total.completed} / ${total.total}")

    let g = ollgen.with_keep_alive(ollgen.generate(model, "why is the sky blue"),
                                   OllKeepForever)
    let reply = ollapi.generate(t, g)!
    Ok(reply.text)
```

## The layer, and why

`host`, and six of the seven modules declare nothing.

| module | row | why |
| --- | --- | --- |
| `ollfault` | `[]` throughout | a fault is a value built from a status and a body |
| `ollnd` | `[]` throughout | the newline-delimited reader, feed and drain |
| `ollmodel` | `[]` throughout | a model reference, a Modelfile and the metadata bodies |
| `ollgen` | `[]` throughout | the generate and chat requests and replies |
| `ollembed` | `[]` throughout | the two embedding endpoints' shapes |
| `ollpull` | `[]` throughout | the pull's events and the progress arithmetic |
| every call in `ollapi` | `[e]` | effect-POLYMORPHIC: whatever the transport costs |

`ollapi` is the only module that reaches a server, and it declares **no
transport of its own**.  Every call is generic over llm-client-nv's
`LlmcTransport[e]`, so a call costs `[net]` over TLS, `[io, net, time,
async]` over the standard library's HTTP client, and nothing at all
over a recorded transcript — which is what `tests/ollapi_tests.nv`
drives, and what makes a whole `/api/pull` stream four lines of a
string in a test.

The headline row is `[net, time]`: the narrow transport's `[net]` plus
llm-client-nv's one clock read.  The wide transport is the same
`[io, net, time, async]` named there.

## The load-bearing interface

**`OllPullEvent`, and the progress a caller pumps.**  Two arguments,
both of them failures a simpler shape produces and nobody notices.

**A pull is minutes long and its errors are inside a 200.**  A 40 GB
model is twenty minutes on a good connection.  An API shaped
`pull(name) -> Result<Unit, Fault>` gives a program nothing to show a
person for those twenty minutes and no way to tell "still going" from
"hung" — and that is the small half.  The large half is that this
endpoint answers **200 and then writes**
`{"error": "pull model manifest: file does not exist"}` **as one of its
newline-delimited objects.**  A client that checked the status code
reports a completed download of a model that was never fetched, and the
next call fails with a 404 that looks like a different bug.
`ollpull.event_of` asks `ollnd.is_error` before anything else, and
`OllPullFailed` is a *variant of the event* rather than a return value,
so a caller draining the stream cannot walk past it.

**The progress numbers are per-layer and reset.**  A model is several
blobs, and `completed` and `total` describe the layer being downloaded,
not the model.  A client that drew one bar from those two numbers draws
a bar that reaches 100% and jumps back to zero four times — which is
the most common visible defect in clients written against this
endpoint.  `OllPullProgress` carries the digest so layers can be told
apart, `OllPullState` is the accumulator a caller threads, and
`ollpull.overall` is the whole-download figure the stream never sends.
The expected total *grows* as blobs are discovered, so a percentage is
"of what is known" and `overall` answers bytes.

There is no blocking `pull` in this package.  `pull_open` and
`pull_next` are the shape, and `/api/push` and `/api/create` stream the
same objects through the same pump.

## Newline-delimited JSON, not server-sent events

Ollama serves two APIs on one port and **they do not stream the same
way.**

- `/v1/chat/completions` streams SSE: `data: ` lines, blank-line frame
  boundaries, a `[DONE]` sentinel that is not JSON.  llm-client-nv's
  `llmcsse` reads that.
- `/api/chat`, `/api/generate` and `/api/pull` stream newline-delimited
  JSON: one complete object per line, no prefix, no blank line, no
  sentinel — the end is a `"done": true` **field** on the last object.
  That is `ollnd`.

A client that pointed the wrong reader at the wrong endpoint gets
nothing and no error: each buffers forever waiting for a boundary the
other's framing never produces.  `ollapi` never gives a caller the
chance, because it only ever speaks `/api/*`.

`ollnd` also frames on the **raw** newline between objects.  A generated
token is very often a newline, and `{"response":"\n"}` is one line
whose JSON encoding contains a backslash and an `n`; a reader that split
on decoded text would cut the object in half.

## Why this package exists, given `/v1`

The same server answers the OpenAI-compatible endpoint, and
llm-client-nv already speaks it —
`llmcreq.compatible("http://127.0.0.1:11434", "")` is a working client
and needs nothing from here.  **If a program wants hosted models and
this server behind one code path, that is the path.**  Four things live
only here, and each is a reason somebody runs a model locally:

- **`keep_alive` — how long the model stays resident.**  Loading a 7B
  model off disk takes seconds; the default eviction is five minutes.
  A service answering one request a minute pays the load on most of
  them and cannot say otherwise through `/v1`, which has no field for
  it.  `ollapi.preload` and `ollapi.unload` are the two ends of it, and
  `preload` sends an empty prompt so a warm-up costs no tokens.
- **`options` — the whole sampling set.**  `num_ctx`, `num_predict`,
  `repeat_penalty`, `mirostat`, `num_gpu`, `num_thread`, `seed`.  `/v1`
  maps a handful and drops the rest, so a caller tuning a local model
  through it is tuning nothing.  `num_ctx` is the one whose wrong value
  is silent: a prompt longer than it is truncated **from the front**
  with no field in the reply saying so, so a long conversation quietly
  loses its system message.  `ollgen.check` refuses a `num_ctx` above
  the model's own, which `ollmodel.context_length` reports.
- **`raw` — bypass the chat template.**  A caller that rendered its own
  prompt — which is exactly what prompt-nv's `promptchat.render_chat`
  produces — must be able to send it unwrapped, or the server applies a
  second template over the first and the model sees two system headers.
- **The timings.**  `prompt_eval_count` and `prompt_eval_duration`,
  `eval_count` and `eval_duration`, `load_duration`.  Prefill and
  decode are different operations with different bottlenecks, and one
  tokens-per-second figure over both hides which of them a change
  helped.  `/v1`'s `usage` has nowhere to put any of it.

And two management calls with no hosted counterpart at all, because no
hosted API has a machine the caller is responsible for: `/api/ps`, which
says what is loaded and **how much of it is on the GPU** — a tenfold
speed difference invisible in every other answer the server gives — and
`/api/show`, whose `capabilities` list answers "can this model be sent
tools" before a 400 does, or worse, before a model that ignores them
answers in prose.

## Where novollm and this meet

orbit/novollm is the **in-process** engine: a GPT decoder with a paged
KV cache, weights read from a safetensors checkpoint, a block pool and
a block table, all inside the program's own address space.  This
package is the **out-of-process** one: another program holds the
weights and the cache, and what crosses is HTTP.

They meet at two places, and both are the same idea:

- **The KV cache.**  novollm's `kv` pool *is* the conversation state;
  `/api/generate`'s `context` array is that state's token-id form
  crossing a process boundary.  The server keeps nothing between calls,
  so a program continuing a conversation carries the array — which is
  the same design decision novollm made by holding the pool, arrived at
  from the other side.
- **Residency.**  A loaded model in novollm is a value the program
  holds; here it is `keep_alive` and `/api/ps`.  A program that wants
  the second to behave like the first asks for `OllKeepForever`.

The choice between them is not about speed.  It is about who owns the
machine: novollm is what a program embeds when it must not depend on
another process, and this is what it uses when the machine already has
one serving every program on it.

## What this package does not do

- **No transport.**  llm-client-nv's `LlmcTransport[e]`, on purpose:
  two clients that each invented a byte pipe would be two pipes a
  program holding both has to configure separately.
- **No `/v1` dialect.**  That is llm-client-nv's, and pointing it at
  this server is one line.
- **No model registry knowledge.**  No table of model names, sizes or
  context windows: such a table is wrong within a month, and
  `ollapi.show` asks the server instead.
- **No normalisation of embeddings.**  A model's vectors may or may not
  be unit vectors and the server does not say; normalising on the way
  in would make `embsim.dot` and `embsim.cosine` the same function for
  a caller who needed to know they were not.  `embnorm.l2_normalize` is
  the call, where the arithmetic is.
- **No clock.**  `modified_at` and `expires_at` are the strings the
  server wrote.  calendar-nv is not a dependency this earns.

## Dependencies

Two:

- **llm-client-nv** (`host`) — the transport trait, the provider value
  and `LlmcToolCall`, which this server's tool-call shape already is.
  A **path** dependency while both are staged together; the published
  release names `llm-client-nv = "^0.0.1"` by range.
- **prompt-nv** (`core`) — the same `PromptConvo` llm-client-nv renders
  into the hosted dialects, rendered here into the local server's own.
  Both packages taking it is what lets a program move a conversation
  between a local model and a hosted one without converting anything.

## Licence

Apache-2.0.
