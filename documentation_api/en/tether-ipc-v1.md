# Tether IPC v1

This protocol connects ChromaLux to local `cls-tether-*` processes. It is an
internal interprocess contract for signed daemons, not a public network API.

Reference sources:

- `tether_ipc.h`;
- `tether_control_channel.h`;
- `TetherSession.cpp`;
- `LiveViewConsumer.cpp`;
- `cls-tether-mock`.

## Architecture

1. The host creates a local Unix socket.
2. It launches the daemon and passes the path through `CLS_TETHER_SOCK`.
3. The daemon connects to the socket.
4. The control channel uses one compact JSON object per line, delimited by
   `\n`.
5. Live View images pass through a POSIX shared-memory ring.

A `TetherSession` controls one daemon and one open camera. The protocol version
is `1`.

## JSON envelopes

Request:

```json
{"id":1,"kind":"request","method":"tether.hello","args":{}}
```

Successful response:

```json
{"id":1,"kind":"response","method":"tether.hello","result":{}}
```

Event:

```json
{"id":7,"kind":"event","method":"tether.camera_added","args":{"camera":{}}}
```

A response contains either `result` or `error`, never both. Identifiers match
requests with responses; events are asynchronous.

The common reader used by provider daemons limits a control line to 1 MiB. The
host and mock do not enforce the same limit everywhere in 0.6.5: an integration
must remain strictly below 1 MiB per message.

## Methods

| Method | Main input | Result |
|---|---|---|
| `tether.hello` | host/protocol versions | daemon identity, versions, capabilities |
| `tether.list_cameras` | empty object | `cameras` array |
| `tether.open` | `camera_id`, `capture_dir` | session, source, shm, optional capabilities |
| `tether.close` | empty object | empty result |
| `tether.trigger_capture` | `capture_id` | capture, source, and timestamp |
| `tether.get_setting` | `key` | setting snapshot |
| `tether.set_setting` | `key`, `value` | value actually applied |
| `tether.list_settings` | empty object | settings inventory |
| `tether.live_view_start` | `max_fps`, `format` | format and source |
| `tether.live_view_stop` | empty object | empty result |
| `tether.af_trigger` | empty object | `af_status` |
| `tether.focus_step` | normalized direction and size | status and native steps |
| `tether.health` | empty object | provider-specific diagnostics |
| `tether.diagnostic_config` | empty object | read-only gPhoto2 state |

`diagnostic_config` is not a surface for changing settings.

## Events

- `tether.capture_transfer_started`
- `tether.capture_file_ready`
- `tether.capture_complete`
- `tether.setting_changed`
- `tether.camera_added`
- `tether.camera_removed`
- `tether.disconnected`

A RAW+JPEG pair shares the same `capture_id`. The host can therefore group
files from the same transaction until `capture_complete`.

## Cameras and settings

Recognized camera fields include:

```text
id, provider, make, model, transport, serial, firmware,
usb_location, support_level, support_reason, available
```

A setting snapshot has the following form:

```json
{
  "key": "iso",
  "value": 400,
  "display_name": "ISO",
  "choices": [100, 200, 400],
  "display": ["100", "200", "400"],
  "min": 100,
  "max": 12800,
  "step": 100,
  "read_only": false
}
```

The UI uses the normalized keys `exposure_mode`, `exposure_comp`,
`shutter_us`, `iso`, `aperture`, `wb_mode`, `wb_kelvin`, `battery`, and
`meter_ev`, among others. A provider may add namespaced keys.

## Capabilities

The response to `tether.hello` may advertise:

- `capture`
- `live_view`
- `settings`
- `shutter_button`
- `ae_lock`
- `af_trigger`
- `focus_step`

The host must make each optional command conditional on the corresponding
capability. An older daemon may respond with the `1001 unknown_method` error.

## Errors

The `error` object contains at least a code and a message.

| Code | Constant | Meaning |
|---:|---|---|
| 5 | `kClsTetherErrInternal` | internal input/output error |
| 6 | `kClsTetherErrNoCamera` | camera not present |
| 13 | `kClsTetherErrPermission` | permission denied |
| 16 | `kClsTetherErrBusy` | resource busy |
| 22 | `kClsTetherErrInvalidArg` | invalid argument |
| 95 | `kClsTetherErrUnsupported` | operation not supported |
| 110 | `kClsTetherErrTimeout` | timeout |
| 1000 | `kClsTetherErrProtocol` | protocol error |
| 1001 | `kClsTetherErrUnknownMethod` | unknown method |
| 1002 | `kClsTetherErrSessionNotFound` | session not found |
| 1003 | `kClsTetherErrHandshakeMismatch` | incompatible versions |
| >= 2000 | provider | SDK- or hardware-specific error |

Disconnect reasons on the wire are `usb_unplug`, `sdk_crash`, `daemon_exit`,
and `host_close`. The host may also produce local reasons such as
`socket_eof`.

## Live View ring

The shared file uses the machine's native byte order.

| Item | Value |
|---|---:|
| magic | `0x434C5354` (`CLST`) |
| version | 1 |
| ring header | 64 bytes |
| number of slots | 4 |
| slot size | 4,194,304 bytes |
| total size | 16,777,280 bytes |
| frame header | 40 bytes |
| maximum payload | 4,194,264 bytes |
| JPEG format | 1 |
| RGBA32F format | 2 |
| end of stream | bit 0 of `flags` |

The daemon writes a slot's header and payload, then publishes `head_seq` with
release semantics. The consumer reads with acquire semantics, selects the most
recent frame, and may drop intermediate frames.

`ClsTetherFrameHeader` provides `seqno`, the microsecond timestamp, format,
dimensions, useful size, and flags.

## Integration limits

- The local socket has no application-level authentication.
- A session accepts only one client.
- The 0.6.5 providers do not all enforce version checks with the same rigor.
- Optional methods are not uniformly available.
- This protocol is reserved for the host and its signed daemons; exposing it
  over a network without a new security layer is outside the contract.
