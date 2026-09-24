# aicompanion — project state

Living notes for this repo. **Update this file (or the relevant `docs/*.md`)
as part of any change that makes something here wrong** — a new backend, a
changed default, a measured number that moved. It is loaded automatically
into every session, so it is the mechanism that keeps context current; a
stale entry here is worse than no entry.

This file is a **lean index** — current state and pointers only. The full
investigation logs (measurements, dead ends, retracted theories, exact
commands) live in `docs/`, split by topic, because this file is loaded in
full at the start of *every* Claude Code session regardless of what the
session is actually about — a 92KB chronological log of old debugging arcs
was pure context cost on every single session, not something worth paying
just to keep it all in one file. See `TODO.md` for the active punch list.

Every number below is measured or fetched, with the date and the command that
produced it, so it can be re-checked rather than trusted. Anything not
verified is labelled as such.

## Hardware budget

**NVIDIA RTX 4060 Laptop GPU, 8188 MiB total** (WSL2). **Default TTS backend
is VITS-Umamusume** (`--tts-backend vits`, speaker id 10 = Grass Wonder),
0 MiB VRAM, runs on CPU, ~1.35s latency — changed 2026-09-16 from Chatterbox
at the owner's request. Chatterbox (voice cloning from a reference clip,
`--tts-backend chatterbox`) is still available and is the larger GPU
consumer of the two (Turbo ~4x whisper's VRAM); when using it,
Nano-on-CUDA (`--chatterbox-nano`, 1857 MiB, 1.18s latency) beats Turbo
outright on both size and speed, so prefer it over Turbo.

Full measured tables (STT/TTS latency and VRAM by backend, the Chatterbox
Nano git-install requirement, installed Ollama models): **`docs/hardware-budget.md`**.

## Voice pipeline (speaker gate, output volume, announcements, memory)

- **Speaker verification** gates replies to the enrolled owner's voice
  (WeSpeaker ECAPA-TDNN-512 ONNX, `--speaker-threshold 0.6`). Owner scores
  0.765-0.895 when the voiceprint was enrolled, a stranger 0.069-0.075 — but
  **through the Stick over BLE the owner has scored only 0.611, 0.661 (2026-09-16)
  and 0.610 (2026-09-24, a 2.91s clip)**, i.e. a margin of ~0.01 over the
  threshold, so a slightly worse take would lock the owner out. A synthetic
  voice scores 0.084. Re-enrolling through the Stick or a lower threshold (0.5
  still leaves a wide gap to strangers) would widen it; not yet done. `--short-utterances
  {ask,allow}` handles clips under ~2s, which can't be embedded reliably;
  currently run with `allow` at the owner's request (a real, deliberate hole
  for sub-2s clips).
  A failed check (model missing, embedding crashed) now **refuses** the
  utterance with a neutral line instead of letting it through
  (`--speaker-on-error {reject,allow}`, default `reject`): it used to fail open,
  and in the cluster the gateway silently accepted every voice for lack of
  `curl` (fixed and verified live 2026-09-23: the synthetic voice is now refused at
  score 0.084).
  Every verdict is logged (`speaker verdict=… score=… threshold=… audio=…s`)
  and exported to Prometheus (`aicompanion_gateway_speaker_*`), including
  `unverified` for utterances answered without being scored; `tools/obs_tui.py`
  has a `speaker` panel for it.
- **Output is normalized** (ffmpeg `speechnorm`, on by default) — the
  amplifier was already maxed (`setVolume(255)`) but the signal reaching it
  wasn't, at -18.1 dB mean before normalizing. `255` is above M5Stack's own
  ≤191 battery guidance; watch for brownouts.
- **Proactive announcements** (`Session.announce`, `tools/say.py`) and a
  static **encouragement loop** (`--encourage`) both work with **no firmware
  change** — the wire protocol never checks who started a turn.
- **Memory** is currently just `memory/about-me.md`, read fresh every start.
  Persistent/searchable memory needs the agent loop above and doesn't exist
  yet.

Full detail (enrollment mic-state bug, threshold calibration, volume
normalization numbers, announcement locking): **`docs/voice-pipeline.md`**.

## Firmware (`firmware/m5stick_bridge/`)

Three screens, cycled by **BtnA tap** (hold still talks): Rina's face, a
Catppuccin pixel-art **clock** (NTP-synced, no battery-backed RTC on this
board so time is unset after a cold boot until Wi-Fi lands), and a
**pomodoro timer** (50 min focus / 10 min break, BtnB click starts/pauses,
BtnB double-click resets). Powering the device fully off is the **physical
power button** (double-click it — confirmed PMIC-level power-off per
M5Stack's docs); BtnB no longer has a software deep-sleep substitute, that
was removed 2026-09-15 once the real power button's behavior was confirmed.

The event loop must never block during a turn (fixed 2026-09-07) —
`iter_in_thread()`/`call_in_thread()` in `bridge_server.py` keep the
WebSocket ping alive while STT/LLM/TTS run in a worker thread. Same
principle applied on the firmware side (fixed 2026-09-16): `connectNetwork()`
used to block in `setup()` until BLE connected, so with the phone's
companion app not running the Stick sat stuck on the connecting screen with
no way to reach the clock or pomodoro screens — neither of which needs BLE
at all. It now only kicks off advertising and returns; `loop()` (where
BtnA's screen-cycling lives) picks up the actual connect completion on its
own next iteration. Confirmed live: screen cycling now works immediately at
boot with no phone connected.

**Flashing from WSL2**: the Stick isn't visible to WSL by default — hand it
over from Windows first:

    /mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe \
        -NoProfile -Command "usbipd attach --wsl --busid <id>"

(check the current busid with `usbipd list` — it moves between ports),
which enumerates as `/dev/ttyACM0` (or higher if another device is already
attached). Release with `usbipd detach --busid <id>`.

Full detail (the mic-config no-op investigation, the `mouth_closed` sprite
background bug and its two compounding causes, clock/pomodoro layout and
button-handling decisions, the BtnB deep-sleep removal): **`docs/firmware-notes.md`**.

## BLE transport migration (reconnect/soak validation remaining)

Motivation: replace the phone's persistent Wi-Fi hotspot (battery/data
drain) with BLE. ESP32-S3 is BLE-only (no Classic BT), so this is a
from-scratch GATT protocol + a new Android companion app. Full plan:
`/home/aditya/.claude/plans/tranquil-drifting-stream.md`.

**Phase 1 (flash budget) and Phase 2 (throughput) both PASSED.** BLE-only
end state projects to 78.0% flash (22% headroom); measured sustained
throughput is **396-408 kbps**, counted on the Android side, comfortably
clearing the ~256 kbps target. Getting there took finding and fixing five
real bugs (an Android `BluetoothGatt` operation-queue race, a deprecated
write API, NimBLE ATT resource exhaustion, a silent value-length cap, and
zombie BLE connections from `am force-stop`) — full chain in the doc below,
worth reading before touching this code again so none of it gets
re-discovered from scratch.

**Phase 3 (GATT protocol design) is CLOSED (2026-09-16).** The byte-envelope
codec (`ble_envelope.h` / `BleEnvelopeCodec.kt`), bonding, the app-layer
shared-secret AUTH handshake, and TIME_SYNC all round-trip correctly
together on real hardware — see `docs/ble-migration.md` for the full log,
including three real bugs found and fixed (an Android bond-broadcast
receiver needing `RECEIVER_EXPORTED` instead of the generally-recommended
`RECEIVER_NOT_EXPORTED`; `WRITE_ENC` on the RX characteristic reproducibly
rejected by Android's own stack regardless of retries, fixed by dropping to
plain `WRITE` since the AUTH frame is the real access-control gate anyway;
and a stale phone-side bond record after repeated reflashing, needing a
manual unpair).

**Phase 5 (real Android companion app) is DONE (2026-09-16).**
`android_companion/` evolved from spike into a real app: `RelayService` is
a foreground service owning both the BLE central connection and an OkHttp
WebSocket bridge to the gateway protocol (translating BLE
frames to/from `start`/`stop`/`reset`/`heard:`/`status:`/`reply:`/binary
audio/`end`), plus a settings UI (host + port + secret, persisted), replacing
`tools/termux_relay.py`'s job. Two real bugs found and fixed: Android
blocks cleartext `ws://` by default since API 28 (deliberately allowed, see
`AndroidManifest.xml`'s comment), and `targetSdk` 36 enforces edge-to-edge
layout unconditionally, making the old `WindowCompat.setDecorFitsSystemWindows`
opt-out a no-op — fixed with a real window-insets listener instead.

**Phase 4 (merged into the real `firmware/m5stick_bridge/`) is also DONE
(2026-09-16).** `WiFi.h`/`WebSocketsClient` are gone from the real
firmware; `ble_transport.h` (adapted from the Phase 3 spike's proven
bonding/AUTH/TIME_SYNC/codec) and `ble_envelope.h` replace them, and the old
`webSocketEvent()` became `handleBleFrame()` with unchanged logic. Builds at
**71.3% flash** — better than Phase 1's 78.0% projection. **Confirmed on
the actual `m5stick_bridge` firmware now flashed and running** (not a
spike): connect → bond → AUTH → TIME_SYNC (`settimeofday` called with a
real epoch) → normal `loop()` running correctly, on the Stick's own serial
log; **`WS connected to bridge_server.py`** on the phone (via
`tools/echo_server.py` as the stand-in), independently confirmed via `ss`
showing the live TCP connection on both ends of this whole chain.

**Phase 6's conversation test is DONE (2026-09-16).** Two full turns ran
against the real `bridge_server.py` (not the echo stand-in) over BLE,
confirmed independently in both the server log (speaker gate accepted at
0.611/0.661 vs. the 0.6 threshold, correct STT, in-character LLM reply) and
the Stick's own serial log (`listening... -> Processing -> heard ->
Generating -> Done`, no drops). `README.md` now shows
`android_companion/` bridging the Stick's BLE connection to the k3s gateway,
with `bridge_server.py` retained as the standalone fallback. The relay now
closes stale GATT clients,
times out stalled scan/handshake stages, reconnects BLE/WebSocket with bounded
backoff, and synchronously tears down on Stop to make Start → Stop → Start
reliable. This APK (`versionName` 1.0) is installed on the phone
(owner-confirmed 2026-09-23). **Start → Stop → Start and a Bluetooth toggle
both reconnect correctly on the real device** (owner-verified 2026-09-23).
**Still remaining**: reboot/deep-sleep reconnect and the hours-long soak.

Full log (measured flash-budget tables, the full bug-by-bug debugging arc,
Android tooling setup in WSL2): **`docs/ble-migration.md`**. Remaining
phase (6): **`TODO.md`**.

## Home-server deployment (k3s primary path; device verification pending)

Motivation: this project's purpose is explicitly employability (portfolio
piece for recruiters), which is why this track favors real infra (k3s, a
custom controller) over the leaner option a pure personal-use deployment
would pick. Target hardware is two real, heterogeneous GPU nodes: the home
server's GTX 1650 (4GB, always on) and this laptop's RTX 4060 (intermittent).

`bridge_server.py` has been split along `voicepipe/registry.py`'s existing
STT/LLM/TTS/SV boundaries into HTTP services (`services/stt`, `services/agent`,
`services/tts`, `services/gateway`), deployed via k3s manifests
(`deploy/kubernetes/`). `controller/gpu_scheduler/` is a custom Kubernetes
controller that retargets the `agent` service's LLM endpoint (`LLM_HOST` +
`LLM_MODEL`) to whichever GPU node is currently up — the differentiated piece
of this track. LLM serving there is llama.cpp (`llama-server`, CUDA), migrated
from Ollama on 2026-09-18 — see `docs/llama-cpp-migration.md`.

**The service split and cluster core are live on both nodes.** `arch-ssd`
(GTX 1650): k3s
live, the containerd→nvidia-container-runtime→RuntimeClass→device-plugin GPU
chain verified end to end (including GPU time-slicing, since the node has
one physical GPU shared by two pods), `llama-cpp-gtx1650`/`stt`/`tts`/`agent`
all `Running` with confirmed CUDA access. `tts`'s `/synth` 500 (both TTS
backends assumed a dev machine's conda envs) is **fixed and re-verified
live** (2026-09-16) — see `TODO.md`.

The RTX 4060 laptop (WSL2/Ubuntu) has **joined as a second node, and the
whole chain is proven working end to end on it, live (2026-09-16, on the
Ollama-era stack; the live `agent` now points at `llama-cpp-rtx4060` with
`qwen3-8b`)**: a real request round-tripped `gateway` → `agent` → the 4060
model on this laptop → a real generated reply. Getting here meant finding and
fixing a chain of WSL2-specific networking bugs — kopf needing `patch` on
nodes for its own bookkeeping, `agent`/the controller needing a
`nodeSelector` to stay off the intermittent node, and (the deep one) both
k3s's reverse-tunnel and flannel's VXLAN backend defaulting to each node's
LAN IP instead of its Tailscale address, silently blackholing all
cross-node pod traffic until `--node-ip`/`--flannel-iface=tailscale0` were
set on **both** nodes. `agent.yaml` carries a paired `LLM_MODEL` env var the
controller patches alongside `LLM_HOST` (originally `OLLAMA_MODEL`/
`OLLAMA_HOST`), so each node requests the model sized for its own card
(`qwen3.5-4b` on the 1650, `qwen3-8b` on the 4060). Full bug-by-bug log:
`TODO.md`. Each llama.cpp server keeps its own native-Linux GGUF cache
hostPath. Disk caution learned the hard way: a 20GB model copy crashed this
laptop's `C:` drive mid-transfer on 2026-09-16 (only ~25GB was really free,
not the ~880GB `df -h` reported from inside WSL2 — check real free space
from Windows PowerShell). Conda and the npm/pip/HF caches have since moved
to `D:` (`/mnt/d/wsl-cache`), and `C:` now has ~91GB free.

**`services/gateway` now has full wire-protocol parity with
`bridge_server.py`** (2026-09-16) — the real `start`/`stop`/`reset` +
`heard:`/`status:`/`reply:`/binary-audio/`end` protocol, the in-process
speaker-verification gate, `announce`, and `encourage_loop` are all ported,
verified against both a mocked test suite and a live smoke test against the
real running `stt`/`agent`/`tts` pods. **`gateway.yaml` is applied and
`Running`** too (2026-09-16), re-verified with the same live wire-protocol
test against the actual in-cluster pod. As of 2026-09-18 the Android app
defaults and one-time migrates to the stable gateway endpoint
`arch-ssd.tail38f762.ts.net:30800`, and the gateway manifest mounts the
profile/voiceprint from a Secret. Applying those latest changes and testing
one real Stick conversation remain before the runtime cutover is verified.

Full design (service-boundary reasoning, the k3s-vs-alternatives tradeoff,
node/service placement table, the added-latency cost of splitting a process
into networked services): **`docs/deployment-architecture.md`**. Remaining
deployment work: **`TODO.md`**.

## Architecture reminders

- Adding an LLM backend is one file in `voicepipe/backends/` — auto-discovered,
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
