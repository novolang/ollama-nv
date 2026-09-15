# ollama-nv

[Ollama](https://ollama.com) is a program that runs large language models on
your own machine and serves them over HTTP on port 11434. It has two APIs:
an OpenAI-compatible one under `/v1`, and its own under `/api`, documented
in the project's
[API reference](https://github.com/ollama/ollama/blob/main/docs/api.md).
This package speaks the second one in novo-lang. It is built on
[llm-client-nv](https://novo-lang.org/packages/llm-client-nv), whose
transport and provider value it uses, and on
[prompt-nv](https://novo-lang.org/packages/prompt-nv), whose conversation
type it sends.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What the API is

A **model** is named `family:tag`, such as `llama3.2:3b`, and the tag
defaults to `latest`. Models are **pulled** from a registry onto the
machine, and a pulled model is **resident** only while it is loaded into
memory.

Three endpoints generate text. `/api/generate` takes a single prompt.
`/api/chat` takes a conversation and can be given tool definitions.
`/api/embed` turns text into vectors. Five endpoints manage the machine:
`/api/tags` lists what is pulled, `/api/show` describes one model,
`/api/ps` says what is loaded right now, and `/api/pull`, `/api/push` and
`/api/create` move models.

**`keep_alive`** is how long a model stays in memory after a request. The
server's default is five minutes. Loading a seven-billion-parameter model
off disk takes seconds, so a service answering one request a minute pays
that load on most of them.

**`options`** is the full sampling and runtime set: the context window, the
reply cap, temperature, the repetition penalties, mirostat, the seed, how
many layers go on the GPU, and how many threads to use.

The streaming endpoints send **newline-delimited JSON**: one complete JSON
object per line, no prefix, no blank line and no sentinel. The last object
carries `"done": true`. This is not the same as server-sent events, which
is what the `/v1` endpoints on the same port send.

A **pull** streams status objects and per-layer progress objects. Each
progress object names a **digest**, the identifier of one blob of the
model, with how many of that blob's bytes have arrived and how many there
are. A model is several blobs.

| Quantity | Value |
| --- | --- |
| Default base URL | `http://127.0.0.1:11434` |
| Default tag | `latest` |
| Default residency after a request | 5 minutes |
| `keep_alive` for "stay loaded" | −1 |
| `num_predict` for "fill the context" | −2 |
| Endpoints this package names | 13 |
| Unset integer options | −1 |

The four things only the `/api` endpoints offer:

| | Why it matters |
| --- | --- |
| `keep_alive` | `/v1` has no field for residency, so a program cannot keep a model loaded. |
| the full `options` set | `/v1` maps a handful and drops the rest, so tuning a local model through it tunes nothing. |
| `raw` | Sends a prompt without the model's chat template, for a caller that rendered its own. |
| the timings | Prefill and decode are separate counts and durations. `/v1`'s usage has nowhere to put them. |

## Install

```
novo pkg add ollama-nv
```

## Example

```novo
use ollapi
use ollgen
use ollpull
use llmchttp

// Download a model, printing progress, then generate with it. The
// transport is llm-client-nv's, so the effects here are the standard
// library's HTTP client's. Over `LlmcTlsHttp` the same code is `[net]`.
fn ready(t: LlmcStdHttp, model: Str) -> Result<Str, OllFault> [io, net, time, async]
    // `/api/pull` answers immediately and writes objects for minutes.
    var s = ollapi.pull_open(t, model, false)!

    while ollpull.succeeded(s.state) == false
        let step = ollapi.pull_next(t, s)!
        s = step.stream
        // The whole download, not the layer in flight. The stream
        // itself only reports per-layer numbers.
        let total = ollpull.overall(s.state)
        println("${total.completed} of ${total.total} bytes")

    // Keep the model resident, so the next request does not pay the
    // seconds it takes to load the weights off disk.
    let g = ollgen.with_keep_alive(ollgen.generate(model, "why is the sky blue"),
                                   OllKeepForever)
    let reply = ollapi.generate(t, g)!
    Ok(reply.text)

fn main() [io, net, time, async]
    let p = ollapi.local_provider()
    match llmchttp.std_http(p, 600000)
        Err(f) => println(f.message())
        Ok(t)  =>
            match ready(t, "llama3.2")
                Ok(text) => println(text)
                Err(f)   => println(f.message())
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: ollama-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `ollmodel` | A model reference and its parts, the tag and show and ps replies, and the Modelfile as a value. |
| `ollgen` | The generate and chat requests, the sampling options, the residency, the reply, its timings and the streaming accumulator. |
| `ollembed` | The two embedding endpoints' requests and the reply, with the dimension and shape checks. |
| `ollpull` | A pull's events, the accumulator a caller threads, and the whole-download progress the stream never sends. |
| `ollnd` | The newline-delimited reader: feed bytes, take lines, and the three questions asked of each line. |
| `ollapi` | Every endpoint, each generic over a transport, plus the local provider value and the "is it running" check. |
| `ollfault` | The eleven ways a call fails, whether each is worth retrying, and whether a pull would repair it. |

Six of the seven modules declare no effects. Every call in `ollapi` is
generic over llm-client-nv's transport trait and costs whatever the
transport costs: `[net]` over TLS, `[io, net, time, async]` over the
standard library's HTTP client, and nothing at all over a recorded
transcript.

## How to choose an entry point

**`ollapi.generate` takes one prompt and `ollapi.chat` takes a
conversation.** Take chat when the model has a chat template and the
program has turns. Take generate when the program has one string, when it
wants `raw`, or when it is continuing a conversation through the `context`
array.

**`generate_stream`, `chat_stream` and `stream_next` are the streaming
form.** They hold a state value the caller threads through its own loop.

**`pull_open` and `pull_next` are the only way to pull.** There is no
blocking `pull` in this package. `push_open` and `create_open` are the same
pump over `/api/push` and `/api/create`.

**`ollapi.preload` and `ollapi.unload` are the two ends of residency.**
`preload` sends an empty prompt with a `keep_alive`, so a warm-up costs no
tokens.

**`ollembed.embed` takes one string and `embed_many` takes a list.** The
reply is the same shape either way, and `ollembed.first` is the single
vector out of it.

**`ollnd` is usable on its own**, for a caller reading one of these streams
through a transport this package does not know.

## The rules a user needs

1. **An error can arrive inside a 200.** `/api/pull` answers 200 and then
   writes `{"error": "..."}` as one of its newline-delimited objects. A
   client that checked only the status reports a completed download of a
   model that was never fetched. `ollnd.is_error` is asked of every line
   before anything else, and `OllPullFailed` is a variant of the event, so
   a caller draining the stream cannot walk past it.
2. **A pull's progress numbers are per layer and they reset.**
   `completed` and `total` describe the blob being downloaded, not the
   model. A bar drawn straight from them reaches the end and jumps back to
   zero once per blob. `ollpull.overall` is the whole-download figure, and
   the stream never sends it.
3. **The expected total grows during a pull**, because blobs are
   discovered as it goes. A percentage is therefore "of what is known so
   far". `ollpull.fraction` answers zero until something is known, and
   showing bytes avoids the question.
4. **A pull spends real time in `verifying sha256 digest`**, during which
   no byte counter moves. `OllPullState.status` is the words, and
   `ollpull.describe` is one line a terminal can print.
5. **The `/api` endpoints stream newline-delimited JSON and the `/v1`
   endpoints on the same port stream server-sent events.** Pointing the
   wrong reader at either gets nothing and no error: each waits for a
   boundary the other's framing never produces. `ollnd` is the reader for
   these endpoints and llm-client-nv's `llmcsse` is the reader for those.
6. **A stream is framed on the raw newline between objects.** A generated
   token is very often a newline, and an object carrying one has a
   backslash and an `n` in its text rather than a line break. A reader that
   split on decoded text would cut an object in half.
7. **A stream that ends without its `"done": true` object is truncated.**
   `OllStreamTruncated` carries how many objects arrived, and what arrived
   reads like a result.
8. **`num_ctx` is the option whose wrong value is silent.** A prompt longer
   than the context window is truncated by the server from the front, with
   no field in the reply saying so, so a long conversation quietly loses
   its system message. `ollmodel.context_length` reports the model's own
   ceiling and `ollgen.check` refuses a request above it.
9. **Send `raw` when you rendered the prompt yourself.** Otherwise the
   server applies the model's chat template over a prompt that already has
   one, and the model sees two system headers.
   `promptchat.render_chat` in prompt-nv is what produces such a prompt.
10. **A reply that hit `num_predict` is truncated.** `OllDoneLength` is
    that state, `ollgen.is_complete` is the question, and nothing in the
    text marks it.
11. **Prefill and decode are separate measurements.** `OllTimings` carries
    the prompt token count and duration, the eval token count and
    duration, and the load duration. `ollgen.eval_rate` and `prompt_rate`
    are the two rates. One figure over both hides which half a change
    helped.
12. **`/api/ps` says how much of a model is on the GPU.**
    `ollmodel.is_on_gpu` is the question, and the answer is a tenfold speed
    difference that no other reply from this server reveals.
13. **Ask `/api/show` whether a model supports tools.** `ollmodel.can`
    reads the capability list. A model that does not support them may
    answer in prose describing the call it would have made.
14. **There is no rate limit and no quota.** The resource that runs out is
    memory, and a model larger than the machine's RAM or VRAM fails at load
    time, minutes into a call. `OllModelLoadFailed` carries the server's
    message.
15. **`OllModelNotPulled` is not retryable.** The repair exists, and it is
    a different call. `ollfault.wants_pull` is the question a program asks
    before making it.
16. **The server keeps no conversation between calls.**
    `/api/generate`'s `context` array is the state, and a program
    continuing a conversation carries it. `ollgen.with_context` is where it
    goes.

## What is not included

- **A transport.** Every call is generic over llm-client-nv's transport
  trait. Two clients that each invented a byte pipe would be two pipes a
  program holding both has to configure separately.
- **The `/v1` dialect.** llm-client-nv speaks it, and pointing it at this
  server is one line:
  `llmcreq.compatible("http://127.0.0.1:11434", "")`. Take that path when a
  program wants hosted models and this server behind one code path.
- **A blocking pull.** See rule 2 and the entry-point section.
- **Knowledge of any model.** There is no table of names, sizes or context
  windows, because such a table is wrong within a month. `ollapi.show` asks
  the server.
- **Normalisation of embeddings.** A model's vectors may or may not be unit
  length and the server does not say. Normalising on the way in would make
  the dot product and the cosine the same number for a caller who needed to
  know they were not.
  [embeddings-nv](https://novo-lang.org/packages/embeddings-nv) is where
  that arithmetic lives.
- **A clock.** `modified_at` and `expires_at` are the strings the server
  wrote.
- **A microcontroller build.** The package is `host`: its subject is
  talking to a server.

## Related packages

- [llm-client-nv](https://novo-lang.org/packages/llm-client-nv) is the
  client for the hosted chat APIs. It owns the transport trait, the
  provider value and the tool-call type this package reuses, and it speaks
  the `/v1` endpoints this server also serves.
  `ollgen.to_llmc_reply` converts a reply into its shape.
- [prompt-nv](https://novo-lang.org/packages/prompt-nv) owns the
  conversation. Both packages take it, which is what lets a program move a
  conversation between a local model and a hosted one without converting
  anything.
- [gguf-nv](https://novo-lang.org/packages/gguf-nv) reads the model files
  this server distributes. This package never opens one.
- [embeddings-nv](https://novo-lang.org/packages/embeddings-nv) is the
  arithmetic over the vectors `/api/embed` answers.
- `std.llm` in the standard library runs inference through the toolchain's
  own path, with the model named as `provider:model`. It is the one-line
  ask; this package is the local server's full surface.
- `std.hf` in the standard library fetches model files from the HuggingFace
  Hub, which is the other way a model arrives on a machine.

## Tests

```bash
novo test tests/ollgen_tests.nv     # 21 tests: requests, options, replies, timings
novo test tests/ollapi_tests.nv     # 20 tests: the endpoints over a recorded transport
novo test tests/ollpull_tests.nv    # 12 tests: the pull's events and its progress
```

The request and reply shapes are the Ollama API reference's own. Because
every call in `ollapi` is generic over the transport, the suite drives a
recorded transcript that costs no effects, so a whole `/api/pull` stream is
four lines of a string in a test.

The suite asserts the failures this package exists to prevent: an error
object inside a 200, a progress bar drawn from per-layer numbers, a stream
framed on a decoded newline, and a stream that ends without its `done`
object.

The tests compile today and fail at run, each on the
`not implemented: ollama-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a time
as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `ollmodel.OLL_DEFAULT_BASE_URL`, `.OLL_DEFAULT_TAG` | yes (they are constants) |
| `ollmodel.name_of`, `.canonical`, `.short`, `.is_reference` | no |
| `ollmodel.details_of`, `.tag_of`, `.decode_tags`, `.decode_ps`, `.is_on_gpu` | no |
| `ollmodel.decode_show`, `.can`, `.context_length` | no |
| `ollmodel.modelfile_of`, `.from`, `.with_instruction`, `.render`, `.base_of` | no |
| `ollgen.keep_alive_value`, `.options`, and the five `with_*` option calls | no |
| `ollgen.encode_options`, `.encode_generate`, `.encode_chat` | no |
| `ollgen.generate`, `.with_options`, `.with_keep_alive`, `.with_context`, `.raw` | no |
| `ollgen.chat`, `.chat_with_options`, `.chat_with_tools` | no |
| `ollgen.is_complete`, `.no_timings`, `.eval_rate`, `.prompt_rate` | no |
| `ollgen.empty_reply`, `.decode_reply`, `.chunk_of` | no |
| `ollgen.stream`, `.apply`, `.partial`, `.finish`, `.check` | no |
| `ollgen.to_llmc_calls`, `.to_llmc_reply` | no |
| `ollembed.embed`, `.embed_many`, `.with_keep_alive`, `.path_of`, `.encode` | no |
| `ollembed.decode`, `.dimension`, `.is_well_formed`, `.first` | no |
| `ollpull.event_of`, `.is_failure`, `.fault_of` | no |
| `ollpull.state`, `.advance`, `.overall`, `.fraction`, `.describe`, `.succeeded` | no |
| `ollpull.encode_pull`, `.encode_push`, `.encode_create`, `.events_of` | no |
| `ollnd.reader`, `.reader_with`, `.feed`, `.take`, `.pending_len`, `.lines_of` | no |
| `ollnd.parse`, `.is_error`, `.error_of`, `.is_done` | no |
| `ollapi.local_provider`, `.provider_at`, `.path_of`, `.streams`, `.is_not_running` | no |
| `ollapi.generate`, `.chat`, `.generate_stream`, `.chat_stream`, `.stream_next` | no |
| `ollapi.embed`, `.tags`, `.show`, `.ps`, `.version`, `.delete`, `.copy` | no |
| `ollapi.pull_open`, `.pull_next`, `.push_open`, `.create_open` | no |
| `ollapi.preload`, `.unload` | no |
| `ollfault.is_retryable`, `.wants_pull`, `.model_of`, `.of_status`, `OllFault.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
