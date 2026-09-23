# aicompanion — todo

Active punch list. See `CLAUDE.md` for current project state and links to
the detailed `docs/*.md` investigation logs behind each of these.

## Firmware / hardware features

- **ENV III HAT integration** — read and report ambient temperature (plus the
  HAT's humidity and pressure readings); define the Hat2-Bus wiring/I2C
  address and add the values to the device status protocol.
- **Battery analysis** — expose battery percentage, charge/discharge state,
  voltage, and trend through the firmware and companion app; use it for
  low-battery warnings and to diagnose power draw during audio sessions.
- **Logo / visual identity refresh** — choose the product logo and update the
  Android launcher icon, app name/branding, and any matching repository or
  dashboard assets.
- **Accurate README screen previews** — replace the current approximate clock
  and pomodoro SVGs with hardware screenshots or a source-faithful renderer
  generated from `clock_face.h` and `pomodoro_face.h`; keep the Rina sprite
  preview synchronized with the firmware visuals.
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

- **Companion control tools** — expose a deliberately small, authenticated
  control surface for: (1) switching the active LLM backend/model, with a
  spoken confirmation and safe fallback if the selected backend is unhealthy;
  and (2) reading and changing M5StickS3 display brightness and speaker
  volume, with bounded values and immediate device-side acknowledgement.
  The brightness/volume portion is hardware-dependent: implement and validate
  it only when the M5StickS3 is available, with the Android app and MCP tools
  sharing the same BLE command path.
- **Proactive push with a dedicated `say:` message type** — `Session.announce()`
  already works today (confirmed against the real Stick, no firmware change
  needed — see `docs/voice-pipeline.md`), but there's no way for the Stick to
  distinguish an agent-initiated proactive push from a normal answer. Add a
  dedicated wire message so the UI can show that distinction.
- **Tools as MCP servers**, not functions wired to one harness (Ollama today),
  so they survive a change of runtime or model without a rewrite.
- **Run live web-search MCP smoke tests on both Qwen routes** — run
  `tools/smoke_mcp.py` once with the RTX 4060/Qwen3-8B route active and once
  after failover to the GTX 1650/Qwen3.5-4B route; confirm both models call
  `search_web`, include a source URL, and return a grounded answer.
- **Spotify Connect MCP controls** — add OAuth/PKCE-backed tools for track
  search, device listing, play/pause/next, and volume control on the phone's
  Spotify client; require Spotify Premium and explicit confirmation for
  playback-changing actions. Keep this separate from streaming audio to the
  M5Stick.
- **Persistent notes MCP tools** — add restricted `read_notes` and
  confirmation-gated `append_note` tools backed by a persistent volume on the
  always-on Arch-SSD node; permit only the notes document, not arbitrary file
  paths, and include timestamps, size limits, and write tests.

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
  replacing `tools/termux_relay.py`'s job), a settings UI (host + port + shared
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
- **Phase 6 — cutover and reconnect hardening, device validation pending.**
  The `tools/echo_server.py` validation
  this bullet calls for is done with the *real* firmware on both ends (see
  above). **The real conversation test is now also done (2026-09-16)**: two
  full turns against the actual `bridge_server.py` (not the echo stand-in)
  confirmed end to end in both the server log (speaker gate accepted,
  correct STT, in-character LLM reply) and the Stick's own serial log
  (`listening... -> Processing -> heard -> Generating -> Done`, no drops).
  Full log: `docs/ble-migration.md`. **`README.md`'s architecture diagram is
  also updated (2026-09-16)** — the flowchart and layout tree now show
  `android_companion/`'s BLE-central-plus-WebSocket-client bridging the
  Stick to the server path. The app now fully closes stale GATT clients,
  retries scans/handshakes/WebSockets with bounded backoff, and synchronously
  tears down both transports on Stop so Start → Stop → Start does not reuse a
  half-stopped service. **Still remaining**: install the updated APK and run
  the real-device Start → Stop → Start, Bluetooth-toggle, reboot/deep-sleep,
  and hours-long soak tests.

## Home-server deployment (k3s across the 1650 and the 4060)

Phase 1 (service split + k8s manifests + GPU-scheduler controller design)
done — see `docs/deployment-architecture.md` for the full plan and
`services/`, `deploy/kubernetes/`, `controller/gpu_scheduler/` for the code.
The live cluster core is running; the repository and Android defaults now use
its gateway as the primary device path, see below.

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
  WSL2/Ubuntu 24.04), and **the whole chain is now proven working end to
  end, live (2026-09-16)**: `gateway`'s wire-protocol test round-tripped
  through `agent` → `ollama-rtx4060` (running on this laptop) → `qwen3:8b`
  → a real generated reply. Getting there took finding and fixing a real
  chain of WSL2-specific networking bugs, each one only surfacing once the
  previous was fixed:
  - `kopf` needs `patch` on nodes for its own bookkeeping, not just
    `get`/`list`/`watch` — without it, every node event 403'd before ever
    reaching the controller's actual logic, so `agent` never got retargeted
    at all.
  - `agent.yaml` and `controller/gpu_scheduler/deploy.yaml` needed a
    `nodeSelector` pinning them to the always-up node — neither does GPU
    work, but with no selector the scheduler put both on the 4060 node, so
    stopping that laptop killed the controller at the exact moment it
    needed to fail `agent` over.
  - k3s's agent-to-server "reverse tunnel" (what `kubectl logs`/`exec`
    depend on) and flannel's VXLAN backend **both** independently defaulted
    to each node's LAN IP instead of its Tailscale address — neither node is
    on the other's LAN, so cross-node pod traffic (including `agent` →
    `ollama-rtx4060`) silently went nowhere. Fixed with `--node-ip` and
    `--flannel-iface=tailscale0` on both the agent (this laptop) and the
    k3s **server** (`arch-ssd`) — confirmed via `agent` opening a real TCP
    connection to `ollama-rtx4060` across nodes, not just the env var
    flipping correctly.
  - `agent.yaml` hardcoded one `--model` regardless of which Ollama backend
    was active. Added `OLLAMA_MODEL` as a second env var (same pattern as
    `OLLAMA_HOST`) that the controller now patches alongside the host on
    every transition, so `qwen3.5:4b` (GTX 1650, 4GB) and `qwen3:8b` (RTX
    4060, 8GB) both get requested correctly.
  - `ollama-rtx4060.yaml` originally needed models copied in via `sudo
    rsync` (pod-to-internet egress is separately broken on this node,
    another WSL2 networking quirk not yet chased down) — that crashed the
    laptop when a ~20GB copy filled its actual ~25GB-free `C:` drive
    mid-transfer. Fixed properly: the pod now bind-mounts this node's
    existing native Ollama data dir directly, read-write, so it shares
    already-pulled models in place instead of duplicating anything, now or
    for any future model.

  `gateway.yaml` is applied and `Running` too, confirmed with the same live
  wire-protocol test (see Phase 5 below for the parity work this
  confirms). Pod-to-public-internet egress on the RTX 4060 node was also
  directly re-verified from an ephemeral node-pinned pod on 2026-09-18:
  DNS resolved `registry.ollama.ai` and HTTPS reached it (the bare registry
  URL's expected 404 made BusyBox `wget` exit nonzero). Still open: chart
  into `deploy/helm/` and wire `deploy/argocd/` for GitOps sync.
- **Phase 3 — observability**: deployed and `Running` in the cluster
  (Prometheus, Grafana, Tempo, kube-state-metrics, DCGM exporter on both
  GPU nodes; verified 2026-09-23). Dashboard polish is item 4 below.
- **Phase 4 — benchmarks**: `benchmarks/latency/` (split architecture vs.
  the `bridge_server.py` monolith — the service split adds network hops on
  a latency-sensitive path, so this needs a real measured comparison, not
  an assumption) and `benchmarks/gpu_allocation/` (how fast
  `controller/gpu_scheduler/` actually retargets `agent` on a node
  Ready/NotReady transition).
- **Phase 5 — gateway parity and repository cutover, done (2026-09-18).**
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
  regardless of which server handles live conversations. `gateway.yaml` is
  applied and `Running`, re-verified with the same wire-protocol test against
  the actual in-cluster pod. The gateway is exposed at NodePort `30800`, the
  Android app now defaults (and one-time migrates) to the always-on node's
  stable Tailscale name, and the manifest mounts the profile and voiceprint
  from a Kubernetes Secret. The manifest and Secret are applied (verified
  live 2026-09-23: `aicompanion-personal-data` exists and is mounted by the
  gateway pod, whose `/health` reports the speaker gate on). **Still open**:
  install the updated APK, and verify a real Stick conversation through the
  gateway plus the BLE soak cases above.

## Next repository improvements

3. **GitOps packaging** — turn `deploy/kubernetes/` into a Helm chart and
   wire `deploy/argocd/` for declarative image rollout and rollback.
4. **Observability dashboard polish** — Prometheus, Grafana, Tempo, and the
   service `/metrics` endpoints are now deployed. Replace the current
   functional four-panel dashboard with a useful operations view: summary
   stats for request rate/error rate and p50/p95 latency, millisecond/second
   units, sensible axes and thresholds, stable service colors, readable
   legends, service-health/target-up panels, and separate service and
   end-to-end voice-pipeline rows. Link slow panels to Tempo traces.
   Validate it with real M5Stick turns so gateway, STT, agent, and TTS spans
   all appear together; the agent benchmark alone does not cover that path.
5. **Measured split-architecture performance** — benchmark the k3s gateway
   against `bridge_server.py`, plus the controller's 4060-up/4060-down
   failover time, before treating the added network hops as free.
6. **Deployment hardening** — pin image digests/tags, add resource requests
   and limits plus liveness probes, and document a tested rollback procedure.
7. **Secrets cleanup** — rotate the placeholder BLE shared secret, keep the
   voiceprint/profile out of images, and document the Kubernetes Secret
   update procedure.
8. **LLM serving evaluation** — the runtime has already moved to llama.cpp
   behind the OpenAI-compatible seam (`docs/llama-cpp-migration.md`, live).
   Still open: the latency/VRAM measurement that doc requires before deleting
   the retained Ollama model data, and a vLLM comparison if it's still wanted.
9. **Android release quality** — add relay lifecycle/instrumented BLE tests,
    versioned signing, and a repeatable release APK path.
10. **CI integration coverage** — assemble the Android APK in CI and add a
    WebSocket gateway integration test alongside the existing unit tests.
