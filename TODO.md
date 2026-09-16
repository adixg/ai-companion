# aigf — todo

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

## BLE transport migration (Phases 3-6)

Phases 1-2 passed — see `docs/ble-migration.md` for the full log. Remaining,
per `/home/aditya/.claude/plans/tranquil-drifting-stream.md`:

- **Phase 3 — GATT protocol design**: byte-envelope codec (frame types for
  START/STOP/HEARD/STATUS/REPLY/AUDIO_CHUNK/etc.), chunk reassembly across
  multiple physical BLE packets (confirmed necessary — a 1024-byte logical
  chunk needs ≥2 physical notifies at the MTU ceiling found in Phase 2),
  bonding + an app-layer shared-secret auth handshake, and time-sync (phone
  writes wall-clock epoch since NTP goes away without Wi-Fi).
- **Phase 4 — merge into `m5stick_bridge`**: rip out `WiFi.h`/
  `WebSocketsClient`, add NimBLE + a new `ble_transport.h`, rewire
  `main.cpp`'s connect/state-machine logic.
- **Phase 5 — real Android companion app**: today's `android_companion/` is
  a disposable throughput-test spike. The real app needs the GATT central
  logic plus a foreground service, the WebSocket bridge to `bridge_server.py`
  (replacing `tools/termux_relay.py`'s job), permissions, auto-reconnect, and
  a minimal settings UI.
- **Phase 6 — cutover**: validate against `tools/echo_server.py` first (no
  LLM involved), then a full conversation test, then a soak test (hours-long
  connection, reconnect after BT toggle/reboot/deep-sleep), then update
  `README.md`'s architecture diagram.

## Home-server deployment (k3s across the 1650 and the 4060)

Phase 1 (service split + k8s manifests + GPU-scheduler controller design)
done this session — see `docs/deployment-architecture.md` for the full plan
and `services/`, `deploy/kubernetes/`, `controller/gpu_scheduler/` for the
code. Nothing below has touched a live cluster yet.

- **Phase 2 — cluster bring-up**: install k3s on both nodes, label them
  (`gpu-tier=gtx1650`/`rtx4060`), apply `deploy/kubernetes/*.yaml`, validate
  `services/gateway`'s Phase-1 `/turn` WS endpoint against a test client.
  Then chart into `deploy/helm/` and wire `deploy/argocd/` for GitOps sync.
- **Phase 3 — observability**: `observability/prometheus/` +
  `observability/grafana/`.
- **Phase 4 — benchmarks**: `benchmarks/latency/` (split architecture vs.
  the `bridge_server.py` monolith — the service split adds network hops on
  a latency-sensitive path, so this needs a real measured comparison, not
  an assumption) and `benchmarks/gpu_allocation/` (how fast
  `controller/gpu_scheduler/` actually retargets `agent` on a node
  Ready/NotReady transition).
- **Phase 5 — gateway parity port**: port `bridge_server.py`'s real
  firmware wire protocol, speaker-verification gate, encouragement loop,
  and `announce` into `services/gateway/`, then cut the M5StickS3 over from
  `bridge_server.py` to the k3s-hosted gateway.
