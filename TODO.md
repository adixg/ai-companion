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
