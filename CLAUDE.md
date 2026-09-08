# aigf — project state

Living notes for this repo. **Update this file as part of any change that
makes something here wrong** — a new backend, a changed default, a measured
number that moved. It is loaded automatically into every session, so it is
the mechanism that keeps context current; a stale entry here is worse than no
entry.

Every number below is measured or fetched, with the date and the command that
produced it, so it can be re-checked rather than trusted. Anything not
verified is labelled as such.

## Hardware budget

**NVIDIA RTX 4060 Laptop GPU, 8188 MiB total, driver 560.94** (WSL2).
Note that `nvidia-smi --query-compute-apps` cannot report per-process memory
under WSL2 (returns `[N/A]`), so components are measured by difference —
start/stop one and re-read the total.

Measured 2026-09-05 with `nvidia-smi --query-gpu=memory.used --format=csv`:

| Component | VRAM | How |
| --- | --- | --- |
| `rina` (qwen3 8.2B Q4_K_M) | **3659 MiB** | 4344 → 685 on `keep_alive: 0` |

STT measured 2026-09-05 from a 0 MiB card, on 1.9s / 3.3s / 12.2s clips:

| faster-whisper `small` | VRAM | load | avg latency | speed |
| --- | ---: | ---: | ---: | ---: |
| cuda, float16 | **685 MiB** | 1.4s | **0.33s** | 17.5x realtime |
| cpu, int8 | **0 MiB** | 2.7s | **1.30s** | 4.5x realtime |

So whisper on CPU is entirely usable — it costs about **+1.0s per turn** and frees
685 MiB. (Any script measuring this must call `voicepipe.cuda.ensure_cuda_libs()`
before importing the backend, or the CUDA path dies with
`Library libcublas.so.12 is not found`.)

TTS engines measured directly against a **0 MiB idle card** (bridge stopped,
Ollama unloaded), each started alone — see the table under "TTS backends"
below. Trust those over any by-difference figure: an earlier reading put
Turbo at 3250 MiB by subtraction, but measured alone from a clean baseline it
is **2805 MiB**.

**Chatterbox is the largest single consumer** — Turbo costs ~4x what whisper
does. It, not the LLM and not STT, is what makes the GPU tight.

### The LLM is already spilling to CPU

`GET /api/ps` while `rina` was loaded reported `size: 5971 MB` but
`size_vram: 3711 MB` — only ~62% of the model is on the GPU, the rest runs on
the CPU. `context_length` was 4096. So today's setup is already past the
card, and the LLM is the thing paying for it.

Moving TTS off the GPU frees 3250 MiB and leaves **~7503 MiB** for the LLM,
which is enough for an 8B Q4_K_M (5971 MB) to sit fully resident with ~1.5 GB
left for KV cache.

## Installed models

`GET /api/tags`, 2026-09-05: `rina:latest` (5.2 GB) and `qwen3:8b` (5.2 GB).
`rina` is qwen3 family, 8.2B, Q4_K_M — i.e. the Modelfile persona on qwen3:8b.

## TTS backends — measured

Measured 2026-09-05 from a 0 MiB idle card, 20 CPU cores, three reply-length
sentences per config, voice cloned from `voice_audition/refs/hinata_ref.wav`
(script: each backend started alone via its own class, VRAM read 3s after
load). RTF = audio seconds per wall-clock second; **above 1.0 is faster than
realtime**.

| config | VRAM | load | avg latency | avg RTF | clones a voice |
| --- | ---: | ---: | ---: | ---: | :---: |
| Chatterbox **Nano, cuda** | **1857 MiB** | 9.0s | **1.18s** | **2.54x** | yes |
| Chatterbox Turbo, cuda | 2805 MiB | 10.7s | 1.53s | 1.75x | yes |
| VITS, cpu | **0 MiB** | 5.1s | 1.35s | 1.84x | **no** |
| Chatterbox Nano, cpu | 93 MiB | 8.4s | 3.70s | 0.79x | yes |
| Chatterbox Turbo, cpu | 93 MiB | 12.1s | 5.65s | 0.43x | yes |

(The 93 MiB on the CPU rows is torch's CUDA context, not model weights.)

Conclusions from this table:

- **Nano-on-GPU beats Turbo-on-GPU outright** — 948 MiB less VRAM *and*
  faster (1.18s vs 1.53s). There is no reason to run Turbo here. Nano is now
  the bridge's default via `--chatterbox-nano`.
- **Nano on CPU does not reach Resemble's claimed 3x realtime.** Measured
  **0.79x on 20 cores**, i.e. slower than realtime and a 3.7s wait per reply.
  Treat the vendor's "3x realtime on 8 CPU cores" as not reproducible here.
- **VITS is the only genuinely free-on-VRAM option**, but it is speaker-id
  based and **cannot clone a voice** — switching to it means giving up the
  Hinata voice.

### Chatterbox Nano — needs a git install, not a PyPI version

- `ResembleAI/chatterbox-nano` on Hugging Face (updated 2026-07-21) holds
  `t3_nano_v1.safetensors` beside the same `s3gen`/`ve` weights Turbo uses.
- 110M params vs Turbo's 350M. Loaded through the *same* class:
  `ChatterboxTurboTTS.from_pretrained(device, nano=True)`, which switches the
  checkpoint and uses `GPT2_small` instead of `GPT2_medium`.
- **PyPI's latest (0.1.7) does not have it** — its `from_pretrained(device)`
  takes no `nano` argument. Git main declares the same version `0.1.7` but
  its source differs, so a version bump will never pick this up.
- Installed here 2026-09-05 with
  `pip install --no-deps --force-reinstall git+https://github.com/resemble-ai/chatterbox.git`.
  `--no-deps` is safe and deliberate: every dependency git pins (torch 2.6.0,
  transformers 5.2.0, diffusers 0.29.0, gradio 6.8.0, …) already matched
  exactly, so only the package source changed. Turbo was regression-tested
  after the swap and still synthesizes.
- A PyPI backup of the pre-swap package is *not* kept in the repo; to revert,
  `pip install --force-reinstall chatterbox-tts==0.1.7`.

## "Hermes" means two different things — don't conflate them

Checked 2026-09-05 (I got this wrong once; the distinction is the whole point):

- **Hermes 3 / Hermes 4** — a *model* family (weights) from Nous Research,
  pullable in Ollama. Covered below.
- **Hermes Agent** — a *separate product*: an MIT-licensed, self-hosted
  autonomous agent runtime, also by Nous Research
  (`hermes-agent.nousresearch.com`, `github.com/NousResearch/hermes-agent`).
  It is an application, not a model and not a library.

Hermes Agent ships 60+ built-in tools (web search, browsing, image gen, TTS),
a three-layer memory system (working / episodic / semantic-skill) with FTS5
cross-session recall and LLM summarization, **skills** the agent writes for
itself as procedural memory (`skill_manage`, agentskills.io standard),
built-in cron scheduling, a bot mode for collaborating specialist bots, and
MCP client support. It runs on Linux/macOS/Windows/WSL2/Docker/serverless.

Crucially it is **model-agnostic** — Nous Portal, OpenRouter, OpenAI, or any
OpenAI-compatible custom endpoint, which includes local Ollama at
`http://localhost:11434/v1`. So "Hermes Agent" and "a Hermes model" are
independent choices, and it is a direct competitor to OpenClaw rather than to
the Claude Agent SDK.

### SUPERSEDED: "it needs ≥64K context, which this card cannot give a local 8B"

**The conclusion in this section is wrong and was corrected 2026-09-07** — see
"hermes3:8b RUNS AT 64K ENTIRELY ON THE GPU" below. The arithmetic here assumes
an **f16 KV cache** and a card shared with TTS. With `OLLAMA_KV_CACHE_TYPE=q4_0`
and TTS/STT on the CPU, `hermes3:8b` holds 64000 tokens with all 33 layers on
the GPU at 7137 MiB. The section is kept because the per-token KV reasoning is
still the right way to think about it; only the precision assumption changed.

### The original (f16-only) reasoning

Hermes Agent's own Ollama guide states a **minimum of 64,000 tokens** of
context for agent use with tools. Against the measured numbers above, for
Llama-3.1-8B (32 layers, 8 KV heads, 128 head-dim → ~128 KiB/token at fp16):

| KV precision | KV at 64K | + 4813 MiB weights | fits in 8188 MiB? |
| --- | ---: | ---: | :---: |
| fp16 | ~8192 MiB | ~13005 MiB | no |
| q8_0 | ~4096 MiB | ~8909 MiB | no |
| q4 | ~2048 MiB | ~6861 MiB | only with the GPU otherwise empty, and q4 KV degrades quality |

**So Hermes Agent driving a local Hermes 8B does not fit here** — not even
with whisper and TTS both pushed to the CPU. Running Hermes Agent against a
*hosted* model sidesteps this entirely and leaves the whole card to
whisper + TTS.

Measured directly 2026-09-05, asking Ollama for `qwen3:8b` at `num_ctx:
64000` (any 8B behaves the same; this is not Hermes-specific):

- Ollama capped context at its own max, **40960** — below the 64K asked for.
- Even so: **total 11064 MiB, only 4738 MiB on the GPU, 6326 MiB on the CPU —
  57% spilled**, with the card at 7398 MiB.
- 11064 MiB exceeds the whole 8188 MiB card, so it would spill even with the
  GPU completely empty.

The runtime itself needs no GPU — Hermes Agent's docs advertise running on a
$5 VPS. **The GPU question is entirely about where the *model* runs.** A
spilled model still works, just several times slower, which on a voice loop
means tens of seconds per reply.

### It CAN be driven programmatically — no fork, no custom channel

Resolved 2026-09-05 by reading the repo (`github.com/NousResearch/hermes-agent`,
HEAD 245e480). It is **Python**, not Node, and ships **three** protocols for
external programs, all driving the same `AIAgent` core
(`website/docs/developer-guide/programmatic-integration.md`):

| Protocol | Transport | Defined by |
| --- | --- | --- |
| ACP (Agent Client Protocol) | JSON-RPC over stdio | `acp_adapter/` |
| TUI gateway | JSON-RPC over stdio or WebSocket | `tui_gateway/server.py` |
| **API server** | **HTTP + SSE, OpenAI-compatible** | `gateway/platforms/api_server.py` |

The API server is the fit for this bridge. Enable in `~/.hermes/.env` with
`API_SERVER_ENABLED=true` and `API_SERVER_KEY=...`, run `hermes gateway`, and
it listens on `http://127.0.0.1:8642`.

- `POST /v1/chat/completions` — OpenAI-compatible and **stateless** (full
  conversation in `messages`), and per the docs the agent handles it "with its
  full toolset (terminal, file operations, web search, memory, skills)". That
  maps directly onto this project's `LLMBackend.ask(messages, think) -> str`.
- Streaming is SSE: standard `chat.completion.chunk` plus a Hermes-specific
  `hermes.tool.progress` event for tool-start visibility.
- `POST /v1/runs` + `GET /v1/runs/{id}/events` (SSE) for attach/detach without
  losing state, plus `/steer`, `/approval`, `/stop`.
- `POST /api/sessions/{id}/chat/stream` emits `assistant.delta`,
  `tool.started`, `tool.completed`, `run.completed`.

**Consequence: `voicepipe/backends/hermes_agent.py` would be a small
OpenAI-compatible client (~40 lines), and the Stick would be talking to a full
agent with no firmware change and no change to the wire protocol.** The
tool-progress events are also the missing progress seam for the Stick's
THINKING state and caption line.

One caveat from `website/docs/user-guide/security.md`: `api_server` counts as
an *unattended* surface (no human to answer an approval prompt), so
`unattended_mode` defaults to `deny` — dangerous commands are blocked
instantly rather than waiting on a timeout. The Stick is exactly such a
surface; decide this deliberately rather than setting `approve`.

## Hermes *model* availability

Checked against ollama.com 2026-09-05:

- Official Ollama library has **Hermes 3 in 3B / 8B / 70B / 405B**, plus older
  OpenHermes 7B, Nous Hermes 7B/13B, Nous Hermes 2 10.7B/34B.
- **No official Hermes 4 at 14B or smaller on Ollama.** Hermes 4 is 14B (Qwen3
  based) / 70B / 405B upstream, and the only 4.x on Ollama is a community
  `hermes-4.3-36B` (~21.8 GB Q4_K_M) — far past this card.
- Realistic local option here is **Hermes 3 8B**. Sizes from ollama.com:
  `hermes3:8b` 4.7 GB, `8b-llama3.1-q4_K_M` 4.9 GB, `8b-llama3.1-q3_K_M`
  4.0 GB, `hermes3:3b` 2.0 GB.
- Hermes *agent* workflows are documented as wanting **≥64K context**, well
  past the 4096 currently in use, and the KV cache is the binding constraint,
  not the weights.

### Does Hermes fit? (worked from the measured numbers)

Card is 8188 MiB. With the Nano bridge resident (whisper 685 + Nano 1857 =
**2542 MiB**, confirmed by reading the card after startup), **5646 MiB is
free**:

- `hermes3:8b` (4.7 GB ≈ 4813 MiB) fits fully resident, leaving ~830 MiB for
  KV cache. So **yes — Hermes runs entirely on the GPU, and the Hinata voice
  is kept.**
- But Llama-3.1-8B's KV cache is ~128 KiB/token at fp16 (32 layers × 8 KV
  heads × 128 head-dim × 2 for K+V × 2 bytes), so ~830 MiB buys only **~6K
  tokens** of context; roughly 12K with a q8_0 KV cache. **The 64K the Hermes
  agent docs ask for is not reachable on this card** while TTS is also on it.
- For comparison, today's `rina` (5971 MB) does **not** fit alongside TTS,
  which is exactly why `/api/ps` showed it spilling 38% to CPU.

Pushing STT and/or TTS to the CPU buys context, at ~1s of added latency each.
All three rows keep Hermes fully GPU-resident; only the context differs:

| layout | GPU used | free for LLM | KV after 4813 MiB of weights | added latency | keeps Hinata |
| --- | ---: | ---: | --- | ---: | :---: |
| whisper cuda + Nano cuda | 2542 MiB | 5646 MiB | ~6.5K fp16 / ~13K q8 | — | yes |
| whisper **cpu** + Nano cuda | 1857 MiB | 6331 MiB | ~12K fp16 / ~24K q8 | +1.0s | yes |
| whisper **cpu** + VITS **cpu** | ~0 MiB | 8188 MiB | ~26K fp16 / ~52K q8 | +1.1s | **no** |

Only the all-CPU row approaches the 64K the Hermes agent docs want, and it is
the row that gives up voice cloning. KV figures assume Llama-3.1-8B's
~128 KiB/token at fp16; halve the cost with a q8_0 KV cache.

## Hermes Agent vs OpenClaw (both checked 2026-09-05)

Both are self-hosted agent runtimes, both model-agnostic, both speak MCP,
both need a custom integration to be driven by this project's M5Stick
front-end. They differ in what they are *best* at:

| | Hermes Agent | OpenClaw |
| --- | --- | --- |
| Author | Nous Research | Peter Steinberger (ex-Moltbot) |
| Runtime | standalone app, "$5 VPS" class | Node.js (26 rec.; 22.22.3+/24.15+/25.9+) |
| Models | Nous Portal, OpenRouter, OpenAI, any OpenAI-compatible endpoint (incl. Ollama) | Anthropic, OpenAI, Gemini, Mistral, Cohere, Groq + Ollama, llama.cpp, LM Studio, vLLM, SGLang, LiteLLM |
| Built-in tools | 60+ (search, browse, image gen, TTS) | shell, browser, files |
| Memory | 3-layer working/episodic/semantic, FTS5 recall + LLM summarization, 8 external providers | sessions, context, memory, multi-agent routing |
| Skills | self-written procedural memory (`skill_manage`, agentskills.io) | not documented as a distinct feature |
| Scheduling | built-in cron | cron + webhooks |
| Channels | terminal, Telegram, Discord (`hermes gateway setup`) | Discord, Google Chat, iMessage, Matrix, Teams, Signal, Slack, Telegram, WhatsApp, Zalo |
| Stated context floor | **≥64K for tool use** | none stated (absence of docs, not evidence of tolerance) |

For this project: OpenClaw's strength is messaging-channel breadth, which is
irrelevant here — the Stick is the front-end. Hermes Agent's strength is
memory + skills + cron, which maps almost exactly onto the wishlist
(persistent memory, reminders, notes, search). Lean Hermes Agent on features;
the deciding risk for either is the missing custom-front-end entry point.

## OpenClaw (original notes)

Checked 2026-09-05. A self-hosted **agent runtime and message router** (Node.js
gateway) by Peter Steinberger, renamed from Moltbot in January 2026. It
bridges chat apps (WhatsApp, Discord, Telegram, Signal, iMessage, Slack, …) to
AI coding agents, and can run shell commands, drive a browser and manage
files. State stays on your machine. It is **not a model** — it routes to one,
so "Claude hooked up with OpenClaw" is its intended shape rather than a hack.

Relevance here: this project already has its own front-end (the Stick, the
bridge, the wire protocol, the pixel-art UI), which is the part OpenClaw would
otherwise supply. Its capabilities are better borrowed as tools than adopted
as a runtime.

## Speaker verification (the gate)

Added 2026-09-05. Only the enrolled owner's voice gets a reply.

- **Model**: WeSpeaker ECAPA-TDNN-512, the official ONNX export from HF
  `Wespeaker/wespeaker-ecapa-tdnn512-LM` (`voxceleb_ECAPA512_LM.onnx`, 24 MB,
  192-d embeddings). Cached at `~/.cache/voicepipe/`, downloaded on first use.
- **Runs in-process in `chat`**, not in its own conda env like the TTS
  backends: it needs only `onnxruntime` and `kaldi-native-fbank` (neither
  pulls torch, both now installed there), and it sits in the path of every
  utterance, where a worker handshake would cost more than the inference.
- **`wespeaker` is not on PyPI** — only `wespeakerruntime`, which drags in
  torchaudio. Hence ONNX + kaldi-native-fbank directly.

### Two things the model is unforgiving about

1. **16 kHz or the embeddings stop discriminating.** Measured: computing fbank
   at a clip's native 24 kHz made same-speaker pairs score *below*
   cross-speaker pairs (0.596 vs 0.708 — worse than chance). After resampling
   to 16 kHz, a controlled pair (two VITS speaker ids, clean audio) gave
   **same speaker 0.665 / 0.767, different speaker 0.511 / 0.342 / 0.371**.
   The bridge already records at 16 kHz so the live path resamples nothing.
2. **Raw int16 sample scale**, not [-1, 1] floats.

### Enrol in the same mic *state* the gate will judge, not just the same room

The bug that broke the first enrollment, worth not repeating. The mic and
speaker share one I2S peripheral: playback runs `M5.Mic.end();
M5.Speaker.begin()`, and the next recording runs `M5.Speaker.end();
M5.Mic.begin()`. `M5.Mic.config()` was applied once in `setup()` and never
re-applied, so a recording made *after* a reply is not in the same state as
one made before any reply.

The first `enroll()` sent only a text `reply:` and no audio, so the firmware
never entered the speaker path and **all five samples were captured in a mic
state that never occurs in conversation**. Measured result: the owner scored
**0.748 on the first utterance (before she had spoken) and 0.17-0.60 on every
one after**, while a different person scored 0.069 — i.e. the model was fine
and the enrollment condition was wrong.

**The fix that mattered was the re-enrollment**, in `Session._say()`: it makes
enrollment speak its confirmations, so the mic is torn down and restarted
between samples exactly as in real use.

`main.cpp` also gained `applyMicConfig()`, called on every `M5.Mic.begin()`
rather than once at boot. **That one turned out to be a no-op** — see below.
It is flashed and harmless, but it is not what fixed anything.

The live voiceprint has 10 samples covering both states (1-5 captured before
any playback, 6-10 after), self-consistency 0.765-0.895. Because
`Voiceprint.score` is best-of, covering both conditions works without
discarding the earlier samples.

### The threshold is not calibrated for a real voice

`DEFAULT_THRESHOLD = 0.5` is a starting point. In the controlled synthetic run
a *different* speaker scored 0.511 and would have been let in.

Measured against real voices on this hardware: **owner 0.765-0.895, a
different person 0.069-0.075** — a wide gap. The bridge is therefore run with
`--speaker-threshold 0.6`, which leaves room for a tired or more distant voice
while still rejecting a stranger by a large margin. Re-check with
`tools/speaker_check.py` if the mic, room or firmware changes; every utterance
logs its score.

A rejected utterance gets one of `speaker.REJECTION_LINES` spoken at random
("You're not Aditya! Give me back to him.") rather than a silent caption, so
whoever tripped it hears why.

### Short utterances: `--short-utterances {ask,allow}`

Added 2026-09-07. A clip under `min_verify_seconds` can't be embedded
reliably — measured by truncating one known-good recording, the *same*
speaker scored **0.074 at 0.75s, 0.617 at 1.5s, 0.829 at 2.0s**. The
information isn't there yet, and no threshold separates anyone below ~2s.

- **`ask`** (default) returns `TOO_SHORT` and speaks a `TOO_SHORT_LINES` line
  asking for a longer one. It does not count toward the anger streak, because
  it is usually the owner being brief rather than a stranger.
- **`allow`** waves anything shorter straight through **unverified**. This is
  a real hole — it is precisely how a visitor's sub-2s utterances were being
  answered before the `TOO_SHORT` verdict existed — so it prints a warning at
  startup and logs `gate bypassed` on every use. Run with it deliberately.

Running with `allow` as of 2026-09-07, at the owner's request.

Also running with `--encourage --encourage-interval 28 32` (see below) — a
half-hour cadence, jittered rather than exactly 30 so it doesn't land on the
clock.

**Rejected: a second SV model for short clips.** ECAPA-TDNN pools statistics
over the utterance, so short-duration degradation is inherent to the model
family, not specific to WeSpeaker — a second model sits on the same curve.
It would also need its own separate enrollment (embeddings across models
aren't comparable; see the mismatch guard above) and a lower threshold, which
rebuilds the same hole with two models resident. The cheaper fix, if the
convenience is wanted back without the hole, is a **recently-verified
window**: trust short clips for ~60s after an accepted utterance, reset on any
rejection. Not built yet.

**This is a filter, not authentication.** A recording of the owner passes,
because it is the owner's voice. It stops other people in the room being
answered; it must not gate anything that matters.

### Usage

    python bridge_server.py --enroll        # hold the button, say a sentence, x5
                                            # (she speaks between samples — that
                                            #  playback is part of the point)
    python bridge_server.py                 # gate is on automatically once enrolled
    python tools/speaker_check.py a.wav b.wav   # pick a threshold
    python bridge_server.py --no-speaker-check  # temporarily answer anyone

Enrollment goes through the Stick deliberately: the print must come from the
same mic, codec and room the gate will judge against. Samples live in
`memory/voiceprint.json`; delete it to start over. With no voiceprint the gate
is off and the bridge behaves exactly as it did before.

## Output volume — the amplifier was maxed, the signal wasn't

Measured 2026-09-07. `main.cpp` has had `M5.Speaker.setVolume(255)` (full
scale) since the first commit, so the *device* was already as loud as it goes.
The signal reaching it was not: a real VITS reply measured
**-3.5 dB peak / -18.1 dB mean**, i.e. the amplifier was at 100% driving audio
at roughly two thirds amplitude.

`resample_to_pcm16()` now runs ffmpeg's one-pass `speechnorm` filter
(`NORMALIZE_FILTER = "speechnorm=e=6.25:r=0.00001:l=1"`), on by default,
disabled with **`--no-normalize`**. Measured on a live reply, before and
after:

| | peak | mean |
| --- | ---: | ---: |
| raw | -4.5 dB | -15.0 dB |
| normalized | **-0.4 dB** | **-10.6 dB** |

Chosen over flat gain (`volume=+4dB`), which would clip the louder chunks, and
over `loudnorm`, which wants two passes. `speechnorm` also lifts quiet
syllables rather than scaling everything. It runs at ~153x realtime, so it
costs nothing next to the second already spent in TTS. Replies and
announcements both go through it, so an encouragement isn't quieter than an
answer.

Unchanged caveat: 255 is above M5Stack's own ≤191 guidance for battery
operation. Louder audio draws more, so if brownout reboots ever appear during
playback on battery, this is the first thing to turn down.

## Proactive announcements

`Session.announce(text)` speaks without anyone pressing the button, and
`--announce-socket` (default `/tmp/rina-announce.sock`) exposes it: one line
in, one announcement out. `tools/say.py` is the client.

**Confirmed working 2026-09-05 against the real Stick, with no firmware
change.** The wire protocol turns out not to care who started a turn —
`reply:` / audio / `end` just means "display and play", and `webSocketEvent`
never checks whether a question is outstanding. This is the foundation for
reminders, timers and alerts.

    python tools/say.py "the build finished"

### Announcements can no longer interleave with a turn

Added 2026-09-07, with the encouragement loop below. `reply:` / audio / `end`
is a *frame sequence*, not a message, so an announcement that started while a
turn was mid-flight would interleave its PCM with the reply's and the Stick
would play both as noise. `Session.speaking` is an `asyncio.Lock` held by
`handle_utterance()` and by `announce()`, so an unprompted line waits for the
turn to finish rather than corrupting it. This was latent for `tools/say.py`
too, not just for the new loop.

## Encouragement (`--encourage`)

Added 2026-09-07. She says something encouraging unprompted every so often:

    python bridge_server.py --encourage                       # every 15-20 min
    python bridge_server.py --encourage --encourage-interval 30 45

- Lines live in `voicepipe/encouragement.py` as a **static list**, not
  generated. Generating one would cost a full LLM turn every quarter hour,
  keep a model or agent warm for nothing, and can wander off-persona or fail
  while nobody is watching. The list is instant, needs no GPU, works offline.
- `line()` avoids anything used in the last `NO_REPEAT_WINDOW` (8) draws —
  hearing the same encouragement twice in half an hour reads as mechanical.
- The name comes from `speaker.OWNER` via a `{owner}` placeholder, so it is
  spelled in one place. Roughly a third of the lines use "-senpai"; every
  line doing so stops landing as affection.
- The interval is drawn fresh from `[MIN, MAX]` each time rather than fixed,
  so it doesn't read as a cron job. `bridge_server.encourage_loop()` builds
  on `announce()`, so **no firmware change** is needed.
- No Stick connected: the line is dropped and logged, never queued —
  encouragement that arrives an hour late is worse than none. Any exception is
  caught so one failure can't kill the task.

Verified 2026-09-07 through the real entrypoint (`--encourage-interval 0.05
0.08` on port 8766) — flags parse, the task starts, ticks fire, and the
no-Stick path logs rather than throwing.

## Memory

`memory/about-me.md` holds hand-written facts about Aditya. `voicepipe/
personas.py` reads it fresh at every start (no rebuild needed) and appends it
to the system prompt under a header telling the model to use it silently.
`--profile PATH` points elsewhere, `--no-profile` disables it.

Everything inside HTML comments is stripped, which is how the shipped template
costs nothing until it is filled in — the unfilled file contributes 27 chars.
This matters: the profile is spent on *every* turn, and the local model has
roughly 6K tokens total, so `load_profile()` warns above 4000 chars.

This is the static half of memory. The other halves — things she learns and
writes back, and searchable notes — need the agent loop and do not exist yet.
Keep them out of the profile: it is the always-resident slice, not a store.

## An 8B agent DOES fit — measured with the card empty and a quantized KV cache

Measured 2026-09-07, superseding the "does Hermes fit?" arithmetic above, which
assumed TTS was resident and an f16 KV cache. Two things changed:

1. **The card is now genuinely empty.** Running `--tts-backend vits
   --whisper-device cpu` leaves `nvidia-smi` reporting **0 MiB used of 8188**,
   confirmed directly. The whole card is available to the model.
2. **Ollama can quantize the KV cache**, which the earlier tables never applied.
   `OLLAMA_FLASH_ATTENTION=1` + `OLLAMA_KV_CACHE_TYPE=q8_0` (or `q4_0`), set as
   **server** environment variables — it is a global server setting, not a
   per-request option. Ollama 0.17.0 here.

`qwen3:8b` (Q4_K_M, 36 layers, 8 KV heads, 128 head-dim — so ~144 KiB/token at
f16), loaded alone on the empty card via `/api/generate` with `num_ctx` set,
then read back from `/api/ps`:

| KV type | num_ctx | total | on GPU | on CPU | fully resident? |
| --- | ---: | ---: | ---: | ---: | :---: |
| f16 | 8192 | 6255 MiB | 6255 | 0 | yes |
| f16 | **16384** | 7431 MiB | 7431 | 0 | **yes — f16 ceiling** |
| f16 | 32768 | 9855 MiB | 7304 | 2551 | no (74%) |
| f16 | 40960 | 11063 MiB | 7467 | 3596 | no (67%) |
| q8_0 | 16384 | 6415 MiB | 6415 | 0 | yes |
| q8_0 | 24576 | 7059 MiB | 7059 | 0 | yes |
| q8_0 | **28672** | 7393 MiB | 7393 | 0 | **yes — q8_0 ceiling** |
| q8_0 | 32768 | 7763 MiB | 6555 | 1208 | no (84%) |
| **q4_0** | **40960** | **6931 MiB** | **6931** | **0** | **yes** |

Read off that table:

- **f16 caps you at 16K.** This is the number every earlier estimate here was
  implicitly using, and it is the reason "Hermes doesn't fit" was the old
  conclusion.
- **q8_0 buys 28K**, at a perplexity cost benchmarked in the 0.002-0.05 range.
- **q4_0 fits qwen3's entire native 40960 window on the GPU** with ~1.25 GB of
  card to spare. 40960 is `qwen3.context_length`, i.e. the architectural
  maximum — there is nothing above it to reach without rope scaling.
- Measured KV cost per token: **147 KiB f16, ~80 KiB q8_0** (from the slope
  between two context sizes at fixed weights). q8_0 is ~55% of f16, not 50%,
  because of per-block scale factors.

So the binding constraint is no longer VRAM — it is **qwen3:8b's own 40960
ceiling** versus the 64K Hermes Agent's docs ask for. Reaching 64K needs a
model whose *native* window is larger; `hermes3:8b` is Llama-3.1-based
(131072 native) and is the obvious candidate.

**`qwen3:8b` declares tool calling.** `/api/show` reports `capabilities:
['completion', 'tools', 'thinking']`. The Ollama guide's claim that "only
`gemma4:31b` has reliable tool calling" is about the four models *that guide
lists*, not a general statement — do not read it as excluding qwen3.

### How to set the KV cache type without touching the system service

The measurements above were taken from a **second Ollama instance** on another
port, which leaves the systemd unit and the running bridge untouched:

    OLLAMA_HOST=127.0.0.1:11435 OLLAMA_FLASH_ATTENTION=1 \
        OLLAMA_KV_CACHE_TYPE=q4_0 ollama serve

Confirm it took effect by grepping the server log for `FLASH_ATTENTION` and
`q8_0`/`q4_0` — quantized KV **silently falls back to f16** on architectures
that don't support it, so setting the variable is not proof it applied.

To make it permanent for the system service instead, use a systemd drop-in at
`/etc/systemd/system/ollama.service.d/override.conf`. Note the service stores
models in **`/usr/share/ollama/.ollama/models`** (owned by the `ollama` user),
*not* `~/.ollama/models` — that second store exists here too and is a
different, stale copy. Checking the wrong one makes a running `ollama pull`
look like it is making no progress.

## Local Hermes: tools DO fire, about 3 times in 4 (corrected 2026-09-07)

`hermes3:8b-llama3.1-q4_K_M` was pulled and tested as the last local lever. It
**fits** — 64000 ctx, 33/33 layers, **7385 MiB** (248 MiB more than Q4_0) — and
it is clearly the better model: no more invented tools, `17*23` answered `391`,
plain questions answered plainly.

**But tools still never execute.** Measured over five identical runs of
"run the shell command `echo marker-N` and report its exact output":

    executed properly: 0/5

### The "0/10" figure was a BROKEN MEASUREMENT — retracted

Proxy-verified rate: **6 of 8 turns produced a real tool call** (later pinned down at 17/30 on the full toolset — see the table below). The earlier
`0/10` in this file was wrong. It came from detecting tool use by grepping the
CLI transcript for a `⚡` activity marker, **which `hermes chat -q` does not
print**. Tools were firing the whole time; the test could not see them.

The reliable way to measure this is a logging proxy between Hermes and Ollama
(`scratchpad/proxy.py` pattern): point `model.base_url` at it, forward to
11434, and record `tool_calls` per call. Ground truth of a real run:

    call 2: roles=['system','user']                     -> tool_calls=['terminal']
    call 4: roles=['system','user','assistant','tool']  -> tool_calls=[]

The `tool` role in call 4 is Hermes feeding the **executed** result back. The
loop works end to end: model requests, Hermes executes, result returns, model
summarises.

**Never measure tool use from CLI stdout.** Use the proxy.

### What actually fails, in the other 1-in-4

The model sometimes does not reach for a tool at all, and then either
fabricates or deflects. Both observed on a file it had no way to guess:

- Fabrication: invented hex contents for `secret.txt` plus a story about
  truncating them, when the real token was `8ffd9d6bb81b3b95`.
- Deflection: "I don't have permission to list or read most files here",
  which is false — `terminal` and `read_file` were both attached.

Confabulation is the dangerous half: nothing in the transcript marks a skipped
tool, so a wrong answer is indistinguishable from a right one.

**Two earlier explanations recorded here were wrong and are retracted:**

1. *"The model opens with `<SCRATCHPAD>` instead of `<tool_call>`."* The
   captured system prompt contains no such convention — the only "scratchpad"
   was the **current working directory** (`/tmp/.../scratchpad`), because the
   tests were run from there. A test-harness artifact.
2. *"Streaming is the trigger."* Contradicted on a larger sample.

The lesson: **n=1 on a stochastic model is not a finding**, and neither is a
pass/fail detector nobody validated.

### Where this leaves it

Hermes Agent is installed, configured, fully local, **fast** (6-10s warm) and
**functional**: it runs terminal commands, reads files, and feeds real results
back. The open problem is reliability, not capability — roughly one turn in
four skips the tool and answers from imagination.

### Measured: trimming tools helps, temperature does not (2026-09-07)

Three conditions, 30 proxy-verified turns each ("run `echo tok-N` and report
its exact output"), counting turns that produced a real `tool_calls` array:

| condition | tools | tool schemas | rate |
| --- | ---: | ---: | ---: |
| temp 0.2, full toolset | 17 | 25.3 KB | 6/10 (n=10) |
| temp **0.0**, full toolset | 17 | 25.3 KB | **17/30 = 57%** |
| temp 0.0, **trimmed** | **6** | **12.7 KB** | **26/30 = 87%** |

Fisher exact, trimmed vs full: **p = 0.020**. Temperature 0.0 vs 0.2 on the
full toolset: no difference (6/10 vs 6/10), so the earlier hope that sampling
was the lever is dead — it fixed the *raw-API* leak but not this.

Trimming also shrank the system prompt from 13.7 KB to **6.4 KB**, because the
skills index goes with it. Combined fixed payload roughly halved.

The trimmed set is the wishlist plus the one tool the test needs:

    hermes tools disable file skills todo clarify vision browser image_gen computer_use
    # leaves: web, terminal, memory, session_search, cronjob

**This is now the configured state.** Temperature is back at 0.2 rather than
0.0 — it measured identically and is the less degenerate setting.

87% is usable for a keyboard tool where a skipped tool is visible. It is still
not good enough to put behind the Stick unattended, because the remaining 13%
fabricates rather than refusing.

Untried levers, cheapest first:

### Current config state

`model.default = hermes3-64k-km`. Both `hermes3-64k` (Q4_0) and
`hermes3-64k-km` (Q4_K_M) exist locally, along with their base tags; remove
with `ollama rm` if the space is wanted back (~9.6 GB for the two bases).

## The blocker is now MODEL QUALITY, not VRAM — measured 2026-09-07

With the KV cache fixed (below), Hermes runs fast: a one-line reply went from
**2m 11s to 14s cold / 6-10s warm**, card at exactly the predicted **7137 MiB**,
33/33 layers resident. The remaining problem is that **`hermes3:8b` (Q4_0) is
not reliable at tool calling under Hermes' full prompt.**

Symptom: instead of emitting a structured tool call, it prints tool JSON as
prose, and invents tools that do not exist —

    {"arguments": {"a": 17, "b": 23}, "name": "multiplication"}
    .function: math_multiplier

and asked to "remember that my M5Stick is called Rina" it **said it would and
never called the `memory` tool** — i.e. the headline wishlist feature silently
does not work.

### What was ruled out, in order

1. **Not the server or the chat template.** Calling Ollama's
   `/v1/chat/completions` directly with one tool, both `hermes3-64k` and
   `qwen3:8b` returned a correct structured `tool_calls` array. So Ollama's
   tool parsing is fine and no `--jinja`-style server fix is needed.
2. **Not the tool count or schema bulk.** Synthetic sweep at 1 / 5 / 10 / 15 /
   20 tools (up to 10 KB of schema) — **every one correct** when the question
   maps to a tool. Tool count is not the trigger.
3. **Not the system prompt size.** Repeated with a ~13 KB system prompt: no
   change.
4. **Partly sampling temperature.** On the raw API the failure is intermittent:
   asked "what is 17 times 23" with 5 tools attached, **1 of 8 attempts leaked
   fake tool JSON at Ollama's default temperature, 0 of 8 at 0.2, 0 of 8 at
   0.0**. `temperature 0.2` is now baked into the Modelfile — Hermes does not
   send `temperature` for the primary model (only for summarization / MoA /
   mini-swe), so the Modelfile governs.
5. **But temperature is not sufficient.** Under Hermes' *real* prompt at 0.2 it
   still leaked. The trigger is the combination, and the model is the weak link.

**`num_gpu` in the Modelfile is what makes it fit; `temperature 0.2` is what
makes it behave — and it is still not enough.**

### Tool Search must be off for a small model

Before the trimming below, hermes3 called the `tool_call` bridge with **no
`name` argument** (`tool_call requires a 'name' argument`). Tool Search's
progressive-disclosure indirection is too much for an 8B. Set in
`~/.hermes/config.yaml`:

    tools:
      tool_search:
        enabled: "off"

**Quote the `off`.** YAML parses bare `off` as the boolean `False`, and
`model_tools.py:471` compares `ts_cfg.enabled != "off"` as a *string* — so the
unquoted form silently leaves Tool Search on.

### Toolsets trimmed

`agent.disabled_toolsets: [delegation, browser-use, code_execution, tts]`.
Measured with `hermes prompt-size`: tool schemas **36.5 KB / 19 tools → 24.4 KB
/ 15 tools**; the system prompt is a separate 13.7 KB. `tts` is redundant
because the bridge does its own speech. Re-enable with `hermes tools`.

### qwen3:8b is disqualified, and not on quality

`Failed to initialize agent: Model qwen3-agent has a context window of 40,960
tokens, which is below the minimum 64,000 required by Hermes Agent.` Hermes
**hard-enforces** the 64K floor at startup — it is not advisory. qwen3 caps at
40960 in Ollama, so the better-quantized (Q4_K_M) model cannot be used at all,
regardless of how well it behaves. `model.context_length` can override the
detected value but must itself be ≥64K, so it cannot be used to sneak qwen3 in.

**This is the real constraint on a local agent here:** the model must have a
≥64K native window *and* be good at tools *and* fit in 8 GB. `hermes3:8b`
satisfies the first and third and fails the second.

### Options if Q4_K_M does not fix it

- **A hosted model** via `fallback_providers` or as the default — Hermes is
  provider-agnostic, and this sidesteps both the 64K floor and the tool
  discipline in one move. The GPU then goes back to being free for TTS.
- **Keep the current bridge** (`--llm-backend ollama`, `rina`) for conversation
  and treat Hermes as a separate, non-voice tool — it works fine for
  file/terminal work where a wrong tool call is visible and correctable.
- Do **not** work around it by lowering the tool count further; the sweep above
  shows tool count was never the trigger.

## Hermes Agent is INSTALLED — 2026-09-07

`hermes --version` → **v0.21.0 (2026.8.31)**, upstream e9bccc90. Installed with

    bash install.sh --skip-setup --skip-computer-use

`--skip-setup` avoids the interactive wizard so the config could be written
directly. Code at `~/.hermes/hermes-agent/`, command at `~/.local/bin/hermes`,
data in `~/.hermes/`. Nothing needed root. (`ripgrep` is absent, so Hermes
falls back to grep for file search — `sudo apt install ripgrep` to fix.)

Configured with `hermes config set`:

    model.provider  = custom
    model.base_url  = http://localhost:11434/v1
    model.default   = hermes3-64k

`hermes3-64k` was built from a Modelfile over `hermes3:8b` carrying
`num_ctx 64000` **and** `num_gpu 99`. `num_gpu` has to live in the Modelfile:
Hermes speaks OpenAI-compatible `/v1`, which has no field for it.

In `~/.hermes/.env`: `HERMES_API_TIMEOUT=1800`, `API_SERVER_ENABLED=true`, and
a generated `API_SERVER_KEY` (the key for the bridge to authenticate with).

**Verified working**: `hermes chat -q "Reply with exactly: PLUMBING OK"`
returned `PLUMBING OK` from the local model, no cloud call.

### `agent.reasoning_effort` must be `none` for hermes3

The first attempt failed with `HTTP 400: "hermes3-64k" does not support
thinking`. The shipped config has `reasoning_effort: null`, which is *not* the
same as disabled — `hermes_constants.parse_reasoning_effort()` treats
`"none"`/`"false"`/`"disabled"` as off, and null as unset, so Hermes still
asked Ollama for thinking. `hermes3:8b` reports `capabilities: ['completion',
'tools']` with no `thinking`, and Ollama rejects the request outright. Setting
`agent.reasoning_effort: none` in `~/.hermes/config.yaml` fixes it.
(`hermes config set agent.reasoning_effort none` refuses the key as unknown and
suggests `reasoning_echo`; edit the YAML instead.)

### The fixed per-call payload is ~8,987 tokens

Reported by Hermes itself on the first turn (`Context: 2 msgs, ~8,987
tokens`) — system prompt plus every enabled tool's schema, sent on every call.
At the measured 1665 tok/s prefill that is **~5.4s** of prefill on turn one,
recovered on later turns by Ollama's prefix cache. `hermes prompt-size` breaks
it down and `hermes tools` / `hermes skills` trim it.

### STILL PENDING: the Ollama KV cache type (needs root)

Without it Hermes works but is **slow — 2m 11s for a one-line reply**, because
the systemd Ollama still runs an **f16** KV cache: at num_ctx 64000 that is
~8000 MiB of KV alone, so the model runs largely on the CPU. `/api/ps` reported
`size_vram` of **16312 MiB on an 8188 MiB card**, which is impossible and is
more evidence that `size_vram` is an estimate to be ignored; `nvidia-smi` read
7897 MiB.

The fix is a systemd drop-in at
`/etc/systemd/system/ollama.service.d/override.conf`:

    [Service]
    Environment="OLLAMA_FLASH_ATTENTION=1"
    Environment="OLLAMA_KV_CACHE_TYPE=q4_0"
    Environment="OLLAMA_KEEP_ALIVE=24h"

then `sudo systemctl daemon-reload && sudo systemctl restart ollama`. Expected
after that: 33/33 layers on the GPU at 7137 MiB and 48 tok/s, per the measured
table below.

### Hermes auto-loads `CLAUDE.md` from the working directory

It warned: `Context file CLAUDE.md TRUNCATED: 49904 chars exceeds limit of
20000`. So running `hermes` from this repo silently truncates this file at 20K
chars. Either trim it, raise `context_file_max_chars`, or accept that Hermes
sees only the first 40%. **This file is now large enough that it costs real
context in every tool that reads it** — worth splitting if it keeps growing.

## hermes3:8b RUNS AT 64K ENTIRELY ON THE GPU — measured 2026-09-07

This **overturns** the "Does Hermes fit?" section further up, which concluded
that a local Hermes 8B at 64K "does not fit here, not even with whisper and TTS
both pushed to the CPU." That conclusion assumed an f16 KV cache. It was wrong.

`hermes3:8b` pulled 2026-09-07: **Q4_0, 8.03B, 4.7 GB**, `capabilities:
['completion', 'tools']`, `llama.context_length = 131072`, 32 blocks, 8 KV
heads. Llama-3.1 family, so its native window is 128K — the thing qwen3:8b
cannot do at any KV precision, since qwen3 caps at 40960.

Measured on an empty card (`OLLAMA_FLASH_ATTENTION=1`,
`OLLAMA_KV_CACHE_TYPE=q4_0`), **forcing full offload with `num_gpu: 99`**, card
read with `nvidia-smi` and layer counts taken from the server log:

| num_ctx | layers on GPU | KV cache | card used | generates? |
| ---: | :---: | ---: | ---: | :---: |
| 40960 | **33/33** | 1440 MiB | 6145 MiB | yes |
| **64000** | **33/33** | 2250 MiB | **7137 MiB** | **yes** |
| 98304 | **33/33** | 3456 MiB | 7875 MiB | yes (only 313 MiB spare) |
| 131072 | 0/33 | 4608 MiB | 1373 MiB | yes, but **all on CPU** |

**64000 tokens, every layer on the GPU, 7137 of 8188 MiB, ~1 GB headroom.**
That is Hermes Agent's stated agentic minimum, met fully resident. 98304 also
fits but leaves too little margin to be safe.

KV at q4_0 measures **36 KiB/token** (1440 MiB / 40960 cells), exactly a
quarter of the 144 KiB f16 figure, and the log confirms it applied rather than
falling back: `K (q4_0): 1440.00 MiB, V (q4_0): 1440.00 MiB`.

### `num_gpu: 99` is load-bearing — Ollama's own estimate is too pessimistic

Without it, the scheduler holds layers back on a card that has room. At 40960
it offloaded only **28/33** and at 64000 only **16/33**, reporting `size_vram`
of 7221 and 7272 MiB — while `nvidia-smi` showed the card actually holding
**5495** and **4047 MiB**. So `/api/ps`'s `size_vram` is an *estimate*, it runs
high for this model, and Ollama then declines to offload against its own
inflated number.

**Trust `nvidia-smi`, not `size_vram`, and pass `num_gpu` explicitly.** With
`num_gpu: 99` every configuration above went to 33/33 and generated correctly.
(Every earlier table in this file used `size_vram`; those qwen3 rows are still
directionally right — they were fully resident, where the estimate and reality
agree — but the spilled rows overstate GPU use.)

### Speed, measured at num_ctx 64000, fully resident

| phase | measured |
| --- | --- |
| generation | **48.4 tok/s** |
| prefill | **1665 tok/s** (14017-token prompt in 8.42s) |
| cold model load | ~8s |

At 1665 tok/s, a Hermes system prompt plus tool schemas of ~10K tokens costs
about **6s on the first turn**. Ollama caches the prompt prefix, and Hermes
deliberately freezes its memory block into the system prompt at session start
to preserve exactly that cache, so later turns re-prefill only the new tokens.
For a voice loop this means one slow first reply, then ~1-3s per turn — which
is livable, and is the number to re-measure if it ever feels worse.

### The configuration that follows from all of this

- Ollama server env: `OLLAMA_FLASH_ATTENTION=1`, `OLLAMA_KV_CACHE_TYPE=q4_0`,
  `OLLAMA_KEEP_ALIVE=24h`.
- Model: a Modelfile over `hermes3:8b` with `PARAMETER num_ctx 64000` **and**
  `PARAMETER num_gpu 99`, built as e.g. `hermes3-64k`. Putting `num_gpu` in the
  Modelfile matters — Hermes talks OpenAI-compatible `/v1`, which has no field
  for it, so it cannot be passed per-request.
- The bridge keeps `--tts-backend vits --whisper-device cpu`; the card must be
  empty for this to fit.
- Trade-off to state plainly: `hermes3:8b` is **Q4_0**, a coarser quantization
  than `rina`/`qwen3:8b`'s Q4_K_M. `hermes3:8b-llama3.1-q4_K_M` (4.9 GB) is the
  better-quality tag and costs ~200 MiB more — untested here, but there is
  headroom for it at 64000.

## Hermes Agent — what the docs actually say (fetched 2026-09-07)

Read from the repo's raw markdown under `website/docs/`, not the rendered site
or the SEO blogspam that dominates a search for this.

- **Install** is one script: `curl -fsSL
  https://hermes-agent.nousresearch.com/install.sh | bash`. It provisions
  Python, Node.js, ripgrep and ffmpeg, clones the repo to
  `~/.hermes/hermes-agent/`, and symlinks `~/.local/bin/hermes`. Data lives in
  `~/.hermes/`.
- **Ollama is a "custom" provider**, configured in `~/.hermes/config.yaml`:

      model:
        default: "hermes3-64k"
        provider: "custom"
        base_url: "http://localhost:11434/v1"

- **Context is raised with a Modelfile**, not a config key:

      FROM hermes3:8b
      PARAMETER num_ctx 64000

  then `ollama create hermes3-64k -f Modelfile`. The guide states 64,000 as the
  minimum for agentic work.
- **`HERMES_API_TIMEOUT=1800`** in `~/.hermes/.env` for slow local models, and
  `OLLAMA_KEEP_ALIVE=24h` so an idle unload doesn't add a reload to the next
  prefill.
- **Prefill dominates the first turn locally.** Hermes sends the system prompt
  plus every enabled tool's JSON schema on every call. `hermes prompt-size`
  reports the byte breakdown; `hermes tools` disables unused toolsets and
  `hermes skills` removes skills. Hermes raises its own stream read timeout
  from 120s to 1800s for local endpoints.

### Tool Search is the feature that makes 64K unnecessary-ish

`website/docs/user-guide/features/tool-search.md`. When MCP or non-core plugin
tools are attached, their schemas are **replaced** in the model-visible tools
array by three bridge tools — `tool_search(queries)`, `tool_describe(names)`,
`tool_call(name, arguments)` — and schemas load on demand. Hermes' own core
tools (`terminal`, `read_file`, `write_file`, `patch`, `search_files`, `todo`,
`memory`, `browser_*`, `web_search`, `web_extract`, `clarify`,
`execute_code`, `delegate_task`, `session_search`) never defer.

This matters here because the 64K floor is driven by schema bulk, and Tool
Search is exactly the lever that shrinks it. It does not remove the floor, but
it means adding MCP servers no longer costs context linearly.

### The wishlist is mostly built in — MCP is for the rest

Mapping the four things wanted (persistent memory, reminders, notes, search)
onto what Hermes ships:

| Want | Hermes feature | Needs an MCP server? |
| --- | --- | --- |
| Persistent memory | `MEMORY.md` (2,200 chars) + `USER.md` (1,375 chars) in `~/.hermes/memories/`, injected into the system prompt at session start, managed by the agent's `memory` tool | no |
| Recall past conversations | Session Search — every session in SQLite at `~/.hermes/state.db` with **FTS5**, ~20ms queries, via the `session_search` tool | no |
| Reminders / timers | built-in cron, drivable in plain language through the `cronjob` tool | no |
| Web search | `web_search` / `web_extract`, **keyless out of the box** (DDGS, plus a rotating Exa/Parallel/Firecrawl/Keenable free ring); SearXNG self-hosted is the free keyed option | no |
| Notes | nothing purpose-built; memory is deliberately capped and is not a store | **yes** |

So MCP is not the way in — it is the extension point once the built-ins are
running. Config shape:

    mcp_servers:
      filesystem:
        command: "npx"
        args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/aditya/notes"]

Keys: `command` (stdio) or `url` (HTTP/SSE), `enabled`, `timeout` (default
300s), `trust` (`full` | `untrusted`), `tools.include` / `tools.exclude`, and
`${VAR}` interpolation from `~/.hermes/.env`. `hermes mcp` is an interactive
picker over a Nous-reviewed catalog; `/reload-mcp` reloads without a restart.
Hermes also reads Claude Code's `mcpServers` block via `hermes import-agent
claude-code`.

### Two cautions specific to this project

1. **`approvals.mode: smart` costs a second inference.** It calls an auxiliary
   LLM to judge whether a shell command is dangerous. On one 8 GB card with one
   model resident, that serializes behind the main generation and adds a full
   turn of latency to anything touching `terminal`.
2. **The Stick is an unattended surface.** `api_server` is grouped with webhooks
   under `approvals.unattended_mode`, which defaults to `deny` — a dangerous
   command is blocked instantly rather than hanging for the 300s approval
   timeout. Keep `deny`; `approve` would auto-approve every dangerous command
   reaching the bridge.

### Cron's no-agent mode is the right home for the encouragement loop, later

`hermes cron create "every 30m" --no-agent --script say.sh` runs a script on a
schedule with **zero LLM involvement**, delivering its stdout verbatim; empty
stdout is a silent tick. That is the same shape as `encourage_loop()`, so if
Hermes ever becomes the runtime, the loop moves there rather than being
reimplemented.

## Agent direction — where it stands

`voicepipe/backends/hermes_agent.py` exists and works (`--llm-backend
hermes-agent`). It is a plain OpenAI-compatible chat-completions client, so it
also drives OpenRouter, vLLM, llama.cpp, LM Studio and LiteLLM by pointing
`--hermes-url` elsewhere. Nothing in the bridge, the wire protocol or the
firmware changed to add it — which is the registry design paying off.

**Streaming/progress is done** (2026-09-05). `voicepipe/registry.py` defines
the event kinds `STATUS` / `DELTA` / `FINAL` plus `stream_reply(llm, messages,
think)`, which every caller uses: a backend may implement the optional
`ask_stream()`, and one that doesn't (or that raises NotImplementedError)
degrades to a single FINAL from `ask()`. So `ask()` is still the only method a
backend must have.

- `hermes_agent.ask_stream()` parses the SSE stream. Its frame handling is
  deliberately loose — the docs promise `event: hermes.tool.progress` on
  `/v1/chat/completions`, but that emission is not greppable in
  `gateway/platforms/api_server.py` (only the `/v1/runs` one, carrying
  `tool_name` and `delta`), so anything event-named like tool progress becomes
  a STATUS and anything unrecognised is skipped.
- `bridge_server.py` forwards each STATUS to the Stick as a new
  **`status:<text>`** wire message and ignores DELTA (partial text would only
  make the caption flicker).
- `main.cpp` has a `status:` branch that updates only the caption, leaving the
  state THINKING so the mauve particle field keeps running. **Unknown text
  messages were already ignored by the firmware, so sending `status:` is safe
  against a Stick that has not been reflashed — it simply isn't displayed
  until you flash.**
- `chat_loop.py` prints STATUS lines and streams DELTA text live into the
  terminal, and pushes STATUS into the orb's caption.

Still open, in order:

1. **Proactive push.** Re-reading `webSocketEvent` in `main.cpp`: the
   `reply:` / binary / `end` path does **not** check whether a turn is in
   progress, and the WS connection is persistent. So the server can send
   `reply:` + audio + `end` unprompted and the currently-flashed Stick should
   speak it — proactive announcements look achievable **server-side only**.
   Reasoned from the code, not yet tested. A dedicated `say:` message is still
   worth adding later so the Stick can distinguish an answer from an
   unprompted announcement.
2. **Tools as MCP servers**, not functions wired to one harness, so they
   survive a change of runtime or model.

## Firmware — flashed 2026-09-07

All three of the changes below are **on the Stick as of 2026-09-07**
(`pio run -t upload --upload-port /dev/ttyACM0`; latest flash 2638336 bytes
written and verified, 83.9% of 3145728, RAM 15.8%). Nothing is pending.

**Flashing from WSL2** — the Stick is not visible to WSL by default. It is
bus `1-4`, `303a:1001`, and has to be handed over from Windows first:

    /mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe \
        -NoProfile -Command "usbipd attach --wsl --busid 1-4"

which makes it `/dev/ttyACM0` (the user is already in `dialout`). Release it
back to Windows with `usbipd detach --busid 1-4`. Without this step
`pio run -t upload` fails with no port found, and there is no `/dev/ttyUSB*`
to look for — it enumerates as CDC-ACM, not as a USB-serial bridge.

1. **`applyMicConfig()` — reverted to a single boot-time call.** It was
   briefly re-applied on every `M5.Mic.begin()`; that re-assert **changed
   nothing and was removed** (flashed 2026-09-07, 2638320 bytes, hash
   verified). The call in `setup()` stays, because the board's defaults are
   not 16 kHz mono. Verified 2026-09-07 by reading `M5Unified/src/utility/Mic_Class.{hpp,cpp}`
   at the pinned version in `.pio/libdeps/`:

   - `mic_config_t config() const` returns a *copy* of the member `_cfg`, and
     `config(cfg)` just assigns it back. Re-applying the values already there
     is a plain self-assignment.
   - `Mic_Class::end()` stops the task, clears `_rec_info[]` and calls
     `_i2s_driver_uninstall(_cfg.i2s_port)`. **It never touches `_cfg`.**
     `_cfg` lives as long as the `M5.Mic` object, so it survives every
     `end()`/`begin()` cycle intact, and `begin()` reinstalls the I2S driver
     from those same retained values.
   - The mic and speaker hold *separate* config structs, so a speaker cycle
     cannot clobber the mic's.
   - `sample_rate` wasn't load-bearing anyway: the recording call passes it
     explicitly as `M5.Mic.record(micBuf, MIC_CHUNK_SAMPLES, SAMPLE_RATE)`.

   So the "different mic state after a reply" theory was wrong. The
   enrollment bug was fixed entirely by re-enrolling with playback between
   the samples, and the rejections logged before this flash turned out to be
   other people, not a mic-state problem — confirmed by the owner.
2. **The `status:` branch in `webSocketEvent`** — updates only the caption
   and leaves the state at THINKING, so the mauve particle field keeps
   running while an agent runs a tool. Unknown text messages were already
   ignored, so the server can send `status:` safely today; it just isn't
   displayed.
3. **The `mouth_closed` background fix** (sprite regeneration) — see below.

### `mouth_closed` was the only sprite with a near-black background

Diagnosed 2026-09-07, after a first attempt blamed the amplitude sampling and
changed nothing. The real cause was in the sprite sheet extraction.

The crop window is hung off her body at `WIDTH_ZOOM` (2.1), and
**`mouth_closed` is the first cell of its v2 row** — close enough to the
sheet's left edge that its window runs past it. Measured: `x0 = -8`, and it
is the *only* one of the 19 sprites that overhangs.

Two failures then compounded in `normalised_crop()`:

1. `sheet.crop((-8, ...))` — PIL pads out-of-bounds with **pure black**,
   giving an 8px black column at local x 0-7.
2. `bright[y0:y1, -8:217]` — numpy reads `-8` as *8 from the right edge*, so
   the slice is `1528:217`, i.e. **empty**. `mask.shape` was then `(120, 0)`.

The guard `if mask.shape == (crop.size[1], crop.size[0])` turned both into one
silent failure: shapes disagreed, so **the entire CANVAS_BG recolouring was
skipped** for that sprite and the raw v2 sheet background survived. Measured
before the fix:

| sprite | exact `#24273a` px | dark (≤45) px |
| --- | ---: | ---: |
| `mouth_closed` | 2860 | **16008** |
| `mouth_small` | 18764 | 0 |
| `listening` | 17922 | 0 |

`mouth_closed` is what `characterSpriteFor()` returns when `speakLevel <=
0.08` — i.e. **during every silence while she speaks**. The sprite is 220px
wide on a 240px screen, so its background sits behind the waveform bars, and
a near-black box appearing behind them on each pause is what read as the bars
"breaking".

The fix replaces the slice-and-guard with `window_mask()`, which builds a mask
the size of the window and treats anything off-sheet as background.

### …and it dragged the sheet's own border in with it

Second finding, same crop. With the background fixed, a **1px vertical stripe
at canvas x=31** was still there, running the full height of the waveform
band. `mouth_closed` was the only sprite with any non-background pixel in that
region at all — 45 of them, all in one column.

It is the **v2 sheet's printed left border at sheet x=13**, colour ≈`#6a7083`.
It is bright, so `CONTENT_THRESHOLD` keeps it and the crop preserves it as if
it were art. Harmless for every crop that sits inside the sheet; `mouth_closed`
is the only one that overhangs far enough to reach it.

Canvas x=31 falls in the 2px gap between the bars at x25-30 and x33-38 — the
3rd and 4th from the right of the left group — which is exactly where the
break was reported.

`frame_lines()` now removes any column or row that is bright across
≥`FRAME_LINE_FRACTION` (0.8) of the sheet. Measured borders: v2 x=13 at 0.82,
v3 x=20-21 at 0.86, idle x=16-17 at 0.81, plus each sheet's right-hand
counterpart and v3's horizontal row rules at 0.95-0.97. Real art tops out
around 0.2 on this measure — a cell is about a sixth of a sheet's height — so
the threshold has a wide margin.

Across both fixes, **only `mouth_closed.png` changed**; the other 42 PNGs are
byte-identical. Regression tests in `tests/test_make_face_sprites.py`.

**The earlier amplitude-sampling theory was wrong and has been reverted.** It
explained why *speaking* differs from *listening*, but not why the artifact
appeared as a dark region rather than a short bar, and the fix changed nothing
on the device. The 20ms-window-every-40ms sampling is still there, unchanged.

### Superseded: the speaking waveform gap

Diagnosed 2026-09-07 from a report that the bar column breaks while she
speaks. Both columns read the same `levelHist` ring buffer at mirrored x
(`canvas.width() - x - barW`), so the gap was on both sides — it is only
visible on whichever one you happen to watch.

The two states fed that buffer differently:

- **Listening** RMSes every `MIC_CHUNK_SAMPLES` (512 = 32ms) chunk as it
  arrives — contiguous, 100% of the audio measured.
- **Speaking** took a **20ms window once every 40ms**, positioned by wall
  clock. *Half the audio was never measured.* When that window landed in the
  gap between two words the bar read ~0 → height `4 + 0 = 4px`, next to
  neighbours up to 44px. That stub is the break.

Compounding it: a live mic always has a room noise floor, so `micLevel` never
bottoms out; synthesized speech contains **true digital silence**, so
`speakLevel` genuinely reaches zero.

**This was not the bug.** The description above is accurate — speaking really
does skip half the audio, and `speech_orb.py` really does smooth it away
(`_tick()`: `follow = 0.5 if rising else 0.12`) — but it is not what the eye
was seeing, and the firmware change made no visible difference. It was
reverted. Kept here so the same theory isn't re-derived from scratch.

## The event loop must never block during a turn

Fixed 2026-09-07, after a cold start dropped a turn with

    Rina: Hey there, what's up? 😊
    ! turn failed: received 1011 (internal error) keepalive ping timeout

The reply was generated and then thrown away, because the *connection* died
while it was being produced.

**Mechanism.** `websockets.serve()` is called with no `ping_interval` /
`ping_timeout`, so both default to 20s. Every heavy stage of a turn —
`gate.check`, `stt.transcribe`, `stream_reply`, `voice.synth` — was called
synchronously inside the coroutine, so the loop could not answer the Stick's
keepalive ping for the whole turn. Past 20s websockets declares the peer dead,
and the `await ws.send("end")` in the `finally` then raises.

**What made that turn slow** was the first `rina` load after a reboot: 5.2 GB
off a cold page cache, far slower than a warm ~10s load. Measured, for scale:
speaker gate **0.10s**, whisper transcribe **1.54s** (whisper and VITS load at
startup, so neither is inside a turn). So the model load was essentially the
whole thing. `OLLAMA_KEEP_ALIVE=24h` keeps it resident afterwards.

**Fix**: `iter_in_thread()` and `call_in_thread()` in `bridge_server.py`. The
blocking work runs in a worker thread; the loop stays free to answer pings —
and to actually send `status:` updates *while* the model works, instead of
queueing them behind it. `iter_in_thread`'s queue is **unbounded on purpose**:
a bounded one deadlocks if the consumer stops early, because the producer
blocks forever in `put()` with nothing draining it.

This was latent for any slow turn and would have been constant for an agent
backend running tools. Raising the ping timeout would have hidden it; the loop
being blocked is the actual defect.

### The regression test took three attempts — the first two were worthless

Both **passed with the bug deliberately reintroduced**:

1. Asserting "some heartbeats happened" — satisfied by ticks that land
   *before* the blocking stage starts.
2. Counting heartbeats in a window "during" the block — **a stalled event loop
   cannot observe its own stall**, so the block completes before any coroutine
   is scheduled to look.

What works is measuring it afterwards: the heartbeat records the gap between
its own consecutive ticks, and an inline stage shows up as one long gap
(`max(gaps) < BLOCK / 2`). Confirmed in both directions — fails at a 1.00s
stall with the blocking call restored, passes with the fix.

**Always confirm a regression test fails without the fix.** Two of three
plausible-looking tests here did not.

## Architecture reminders

- Adding a backend is one file in `voicepipe/backends/` — auto-discovered,
  declares its own CLI flags. No entrypoint edit.
- `LLMBackend.ask(messages, think) -> str` is the only method a backend must
  have. Callers go through `registry.stream_reply()`, which adds progress
  events when a backend offers `ask_stream()` and falls back to `ask()` when
  it doesn't — so a new backend never has to implement streaming.
- `speech_orb.py` and the firmware render the *same* sprites; both outputs
  come from one `tools/make_face_sprites.py` run, so they cannot drift.
- The M5StickS3 speaker is at `setVolume(255)` (max). M5Stack's own guidance
  is ≤191 on battery to avoid a brownout reboot; set deliberately, revert if
  reboots appear.
