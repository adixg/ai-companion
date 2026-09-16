# aicompanion — todo

Active punch list. See `CLAUDE.md` for current project state and links to
the detailed `docs/*.md` investigation logs behind each of these.

## Firmware / hardware features

- **Wake word activation** — replace hold-to-talk with a wake word, so
  talking to her doesn't need a button press at all.
- **IMU gesture sensor** — activate listening when the wrist is raised
  (M5StickS3 has an onboard IMU already; this is a firmware gesture-detection
  feature, not new hardware).
- **Haptic feedback** — vibration motor feedback for state transitions
  (listening started, reply ready, etc.) — needs a hardware capability check
  first: confirm whether the M5StickS3 has a vibration motor at all before
  scoping this as firmware-only vs. requiring an add-on module.
- **Voice isolation** — separate the owner's voice from background noise/other
  speakers before STT, rather than only gating whole-utterance acceptance
  after the fact (today's speaker-verification gate accepts or rejects an
  entire utterance; it doesn't clean up an accepted one).
- **HAT SPK2 speaker module — not plug-and-play compatible, on hold.**
  Checked against M5Stack's own docs: the StickS3's I2S audio pins (G18
  MCLK/G14 DOUT/G17 BCLK/G15 LRCK, to its onboard ES8311 codec) are internal
  and **not exposed on the Hat2-Bus connector** — that connector only carries
  GPIO5/4/Boot/6/1/7/8/43/44/2/3 plus power/ground. HAT SPK2 was designed for
  the original M5StickC's HAT connector, which *does* expose fixed I2S pins
  (GPIO 25/26/0 on that board's ESP32 chip) — pins that don't exist in any
  comparable form on the StickS3's silicon. So even though the module may
  physically plug into the StickS3's Hat2-Bus, the electrical signals it
  expects aren't present there. Two paths forward, neither a quick config
  flag: (1) hand-wire it — open the HAT SPK2 board, trace which physical pad
  carries BCK/WS/DATA, and bit-bang a second I2S peripheral (I2S_NUM_1) on
  spare Hat2-Bus GPIOs via a custom `M5.Speaker.config()`; or (2) skip the
  HAT connector, treat it as an external amp/speaker wired directly to spare
  GPIOs instead of relying on its connector-based design. Owner chose to
  hold off on this for now.

## Agent / backend

- **Proactive push with a dedicated `say:` message type** — `Session.announce()`
  already works today (confirmed against the real Stick, no firmware change
  needed — see `docs/voice-pipeline.md`), but there's no way for the Stick to
  distinguish an *agent-initiated* proactive push (e.g. Hermes finishing a
  background tool run) from a normal answer. Add a dedicated wire message so
  the UI can show that distinction.
- **Tools as MCP servers**, not functions wired to one harness (Ollama today),
  so they survive a change of runtime or model without a rewrite.

## BLE transport migration (Phase 6 remaining)

Phases 1-5 all passed — see `docs/ble-migration.md` for the full log.
**Phase 4 (merge into `m5stick_bridge`) is DONE (2026-09-16)**:
`WiFi.h`/`WebSocketsClient` are gone from the real firmware, replaced by
`ble_transport.h` (adapted from the Phase 3 spike's proven bonding/AUTH/
TIME_SYNC/codec) and `ble_envelope.h`. Builds at 71.3% flash — better than
Phase 1's 78.0% projection. Confirmed on the actual `m5stick_bridge`
firmware now flashed and running (not a spike): connect → bond → AUTH →
TIME_SYNC → normal `loop()` all working on the Stick's serial log, and
`WS connected to bridge_server.py` on the phone, independently confirmed
via `ss` showing the live TCP connection.

- **Phase 3 (GATT protocol design)**: byte-envelope codec, chunk
  reassembly, bonding, the app-layer shared-secret AUTH handshake, and
  TIME_SYNC are all implemented and confirmed round-tripping together on
  real hardware (2026-09-16) — including three real bugs found and fixed
  along the way (an Android bond-broadcast receiver needing
  `RECEIVER_EXPORTED`, a `WRITE_ENC` characteristic permission Android's
  stack reproducibly refused regardless of retries, and a stale phone-side
  bond record after repeated reflashing).
- **Phase 5 — real Android companion app, DONE (2026-09-16).**
  `android_companion/` evolved from the Phase 3 spike into a real app:
  `RelayService` (foreground service, survives the Activity closing) owns
  the BLE central connection plus an OkHttp WebSocket bridge to
  `bridge_server.py`'s actual protocol (translating BLE frames to/from
  `start`/`stop`/`reset`/`heard:`/`status:`/`reply:`/binary audio/`end`,
  replacing `tools/termux_relay.py`'s job), a settings UI (host + shared
  secret, persisted), a "Forget device" shortcut, and a battery-optimization
  exemption request. Confirmed on real hardware: full connect → bond → AUTH
  → WebSocket-to-`tools/echo_server.py` sequence succeeds end to end,
  independently confirmed via `ss` showing the live TCP connection, not just
  the app's own log. Two real bugs found and fixed along the way: Android
  blocks cleartext `ws://` by default since API 28 (`bridge_server.py` only
  ever speaks plain `ws://`, so cleartext is deliberately allowed — see
  `AndroidManifest.xml`'s comment), and `targetSdk` 36 enforces edge-to-edge
  layout unconditionally, so `WindowCompat.setDecorFitsSystemWindows` is a
  no-op now — fixed with a real `ViewCompat` window-insets listener instead.
- **Phase 6 — cutover, in progress.** The `tools/echo_server.py` validation
  this bullet calls for is done with the *real* firmware on both ends (see
  above). **The real conversation test is now also done (2026-09-16)**: two
  full turns against the actual `bridge_server.py` (not the echo stand-in)
  confirmed end to end in both the server log (speaker gate accepted,
  correct STT, in-character LLM reply) and the Stick's own serial log
  (`listening... -> Processing -> heard -> Generating -> Done`, no drops).
  Full log: `docs/ble-migration.md`. **`README.md`'s architecture diagram is
  also updated (2026-09-16)** — the flowchart and layout tree now show
  `android_companion/`'s BLE-central-plus-WebSocket-client bridging the
  Stick to `bridge_server.py`, in place of the old Wi-Fi-hotspot +
  `tools/termux_relay.py` path. **Still remaining**: a soak test (hours-long
  connection, reconnect after BT toggle/reboot/deep-sleep).

## Home-server deployment (k3s across the 1650 and the 4060)

Phase 1 (service split + k8s manifests + GPU-scheduler controller design)
done — see `docs/deployment-architecture.md` for the full plan and
`services/`, `deploy/kubernetes/`, `controller/gpu_scheduler/` for the code.
Phase 2 is now underway on the home server, see below.

- **Phase 2 — cluster bring-up**: home server (`arch-ssd`, GTX 1650) done —
  k3s installed and labeled, GPU runtime chain verified (containerd nvidia
  runtime + RuntimeClass + device plugin + time-slicing, see
  `docs/deployment-architecture.md`), `ollama-gtx1650`, `stt`, `tts`, and
  `agent` all applied and confirmed `Running` with real GPU access.
  **`tts`'s `/synth` 500 is now fixed (2026-09-16)** — both TTS backends
  hardcoded a dev-machine conda path that didn't exist in the container;
  `PYTHON` now falls back to `sys.executable`, both CLI scripts are `COPY`'d
  into the image, and `vits`'s image now clones its ~1.5GB git-lfs HF Space
  during build. Re-verified live: `/synth` returns a real, valid synthesized
  wav (1.35s, 16-bit mono 22050Hz).

  The RTX 4060 laptop has **joined as a second node** (`laptop-2vc40919`,
  WSL2/Ubuntu 24.04), and `controller/gpu_scheduler/` is deployed there —
  needed an RBAC fix (`kopf` needs `patch` on nodes for its own bookkeeping,
  not just `get`/`list`/`watch`) and `agent.yaml` needed a `nodeSelector`
  pinning it to the always-up node (it was landing on the 4060 node by
  default, so stopping that laptop killed `agent` at the exact moment the
  controller needed to fail it over). **Known bug, not yet fixed**: the
  laptop node is unhealthy — it flaps `Ready`/`NotReady`, its kubelet API
  intermittently 502s, and `ollama-rtx4060` can't start
  (`UnexpectedAdmissionError: no healthy devices present` for
  `nvidia.com/gpu`) — GPU passthrough into containerd under WSL2 specifically
  hasn't been made to work yet, confirmed broken rather than just
  unverified now. Still open: `gateway.yaml` hasn't been applied to the
  cluster yet (the gateway's code has real firmware parity, see Phase 5
  below, but nothing has applied the manifest). Then chart into
  `deploy/helm/` and wire `deploy/argocd/` for GitOps sync.
- **Phase 3 — observability**: `observability/prometheus/` +
  `observability/grafana/`.
- **Phase 4 — benchmarks**: `benchmarks/latency/` (split architecture vs.
  the `bridge_server.py` monolith — the service split adds network hops on
  a latency-sensitive path, so this needs a real measured comparison, not
  an assumption) and `benchmarks/gpu_allocation/` (how fast
  `controller/gpu_scheduler/` actually retargets `agent` on a node
  Ready/NotReady transition).
- **Phase 5 — gateway parity port, mostly done (2026-09-16).**
  `services/gateway/app.py` now speaks the real firmware wire protocol
  (`start`/`stop`/`reset`, `heard:`/`status:`/`reply:`/binary audio/`end`,
  ported via a small `_AsgiWebSocketAdapter` so the exact same dispatch
  logic bridge_server.py uses works against FastAPI's WebSocket), the
  speaker-verification gate (runs in-process here too -- CPU-only, in the
  hot path of every utterance, not worth a network round trip), `announce`
  + its Unix socket, and `encourage_loop` -- all ported with the same
  behavior and reasoning as bridge_server.py's `Session`, see that module's
  updated docstring. `voicepipe/wire_audio.py` is new: the ffmpeg
  resample/normalize step both servers now share, so it can't drift between
  them. Verified two ways: the mocked unit-test suite
  (`tests/test_services_gateway.py`, full happy path plus every failure
  path) and a live smoke test against the real running `stt`/`agent`/`tts`
  pods (real WebSocket client, real faster-whisper call, correct
  `reply:(didn't catch that)`/`end` for a non-speech tone). `--enroll` mode
  was deliberately **not** ported -- enrollment only touches the voiceprint
  file on disk, so bridge_server.py's existing `--enroll` still works
  regardless of which server handles live conversations. **Still open**:
  applying `gateway.yaml` to the cluster at all (nothing has yet), then
  cutting the actual M5StickS3 over from `bridge_server.py` to the
  k3s-hosted gateway once BLE Phase 6's soak test is finished too.
