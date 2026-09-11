# Unofficial Arturia AudioFuse Control Center HTTP API

This document describes the HTTP API shipped with Arturia AudioFuse Control
Center (AFCC) 2.4.0.347. It is based on static analysis of `httpfuse.dll`,
read-only runtime enumeration, and a small number of confirmed write tests
with an AudioFuse Studio.

> Arturia publishes an official developer documentation set for this API
> (v1.0.0) at
> https://dl.arturia.net/products/audiofuse-studio/extra/Fuse_HTTP_API_1_0_0.zip
> (mirrored locally in [`user documentation/`](./user%20documentation/)).
> That set is canonical for behavior; this file adds statically-recovered
> detail — the full route/method matrix and raw binary evidence — that the
> official docs don't include.

Device capability varies by model, input/output index, power state, and
possibly firmware.

The API requires the **latest firmware** on the device and the **latest
AFCC** on the host — it ships **disabled by default**; the user must
explicitly turn it on once via **Preferences → Http Api → Server: On**.
That setting persists across host reboots once set. If discovery finds
nothing, the most common cause is simply that the toggle was never flipped.

The current API version is `v1` (the `/api/v1` prefix). Non-breaking
additions (new endpoints, new fields) may appear under `/api/v1` without a
version bump — a client should ignore unknown fields rather than treat them
as errors. A future breaking change would move to `/api/v2`.

## Base URL

```text
http://localhost:64347/api/v1
```

The port above (`64347`) was observed in one session against a tested Studio
— it is **not stable**. The listener belongs to
`AudioFuseControlCenterAgent.exe`, the AFCC background agent — the API keeps
running even when the AFCC UI window is closed, since it's hosted by the
agent process, not the UI. The port can be dynamically re-allocated if the
requested one is taken, is not exposed in any user-editable config file, and
is not guaranteed stable across agent restarts. Production clients must
resolve it via Bonjour/DNS-SD rather than hardcoding a port (see
[Discovery](#discovery) below); the hardcoded value here is only for the
one-off manual `curl` examples throughout this document.

API version discovery:

```http
GET /api/v1/version
```

```json
{"version":"1.0.0"}
```

## Discovery

The service is advertised over Bonjour/DNS-SD as `_audiofusehttp._tcp.` in
domain `local.`, TCP transport. TXT records are not used in this release —
don't depend on them. The advertised record's `addresses` array **may
include the host's LAN IP**; the HTTP server itself binds only to
`127.0.0.1`, so always connect to `http://127.0.0.1:<port>` — connecting to
the LAN address fails. If a resolved record's addresses don't match any
local interface, treat it as another host's advertisement on the same LAN
and ignore it (the reference plugin filters this way; the sample scripts in
`user documentation/samples/` implement the same filter).

```bash
# macOS
dns-sd -B _audiofusehttp._tcp.
dns-sd -L "AudioFuse Http Api" _audiofusehttp._tcp. local

# Linux (Avahi)
avahi-browse -r _audiofusehttp._tcp.
```

Python (`zeroconf`) and Node (`bonjour-service`) libraries work the same
way — browse for the service type, take the port, ignore the host. See
[`03-discovery-and-connection.md`](./user%20documentation/03-discovery-and-connection.md)
for full listings in both.

A robust client cycles through **DISCOVERING → CONNECTING → READY →
RECONNECTING**: discover the port, confirm with `GET /version` (and
`GET /devices`), then issue normal requests; on a Bonjour `down` event or an
SSE error, close everything and go back to DISCOVERING after a 1–2s backoff.
Doing the version/devices check up front (rather than inferring server
health from the first parameter request's `403`) gives a much clearer error
message to surface to the user ("AFCC is running but no AudioFuse is
connected" vs. an opaque failure).

If Bonjour is unavailable (blocked mDNS on a hardened network), fall back to
asking the user to paste the port shown in Control Center's preferences —
this is a last-resort escape hatch, not the primary path.

## General conventions

Responses use JSON, except for the Server-Sent Events endpoint and empty error
responses. A leaf GET wraps its value in an object named after the final path
component:

```http
GET /api/v1/monitoring/mute
```

```json
{"mute":false}
```

Ordinary setters use POST with a partial JSON object sent to the parent
resource:

```http
POST /api/v1/monitoring
Content-Type: application/json

{"mute":false}
```

```json
{"mute":false}
```

For an indexed resource:

```http
POST /api/v1/input/analog/1
Content-Type: application/json

{"inst":false}
```

Aggregate GET responses are convenient, but a field in an aggregate does not
always have the same meaning as the similarly named leaf. For example,
`/monitoring` uses `volume` for the volume-knob assignment, whereas
`/monitoring/volume` returns the numeric level currently controlled by that
knob. Prefer leaf GETs when requesting a control's current value.

Per the official spec, a successful `PUT`/`POST` returns `200 OK` with an
**empty body** (or `200 Request ignored` with a short note if the write was
a no-op). The examples in this document that show a setter echoing the
field back (e.g. `POST /monitoring {"mute":false}` → `{"mute":false}`) are
what was actually observed at runtime against the tested Studio — a real
discrepancy from the documented empty-body contract, not a typo. Don't rely
on the echoed value in a client; always treat a bare `200` as success and
re-`GET` if you need the confirmed value.

Values are typed per the general JSON conventions: booleans are lowercase
and unquoted (`true`/`false`), integers have no decimal point, floats used
for level/gain controls are always in dB, strings are used for enum keys
(never labels), and `{"endpoints": [...]}`-style arrays and `{"1": {...}}`-
style keyed objects are used for subscriptions and indexed collections
respectively. A type mismatch (e.g. `"true"` instead of `true`) is rejected
with `400 Invalid Value Type`.

Most parent endpoints accept **any combination** of their children in a
single `POST`; `/preset`'s `name`/`slot`/`saved` combination rules (see
[Presets](#presets)) and a few per-channel link constraints are the
documented exceptions, rejected with `403 Invalid Request` because the
combination would be semantically ambiguous or unsafe.

## Setting values

There is no `/set` endpoint. For ordinary properties, send a partial object to
the leaf's parent resource:

```text
GET  /api/v1/monitoring/mute
POST /api/v1/monitoring       {"mute": true}

GET  /api/v1/input/analog/1/inst
POST /api/v1/input/analog/1   {"inst": false}
```

`Content-Type: application/json` is required whenever the request has a
body (optional but recommended otherwise); if present, it must be exactly
`application/json` or the server rejects it with `400 Invalid Content Type`.

**PUT vs POST**: `PUT` sets exactly one field — either on a leaf endpoint
(`PUT /monitoring/mute {"mute": true}`) or on a parent endpoint with a single
child (`PUT /monitoring {"mute": true}`, equivalent to the leaf form). `POST`
sets multiple fields on a parent endpoint in one round trip
(`POST /monitoring {"mute": false, "volume": -3.5, "dim": false}`). Both have
the same effect when only one field is being set; use `PUT` for simple
single-field changes (the URL is self-describing and easy to log) and `POST`
for atomic-feeling multi-parameter updates like scene/snapshot recall. Either
method rejects a write to a read-only endpoint with
`405 Read-Only Resource` — this covers every `*_options` endpoint, plus
`current_source`, `sync`, `is_at_reference_level`, `preset/names`,
`version`, `devices`, and `update/endpoints`.

For PowerShell with `curl.exe`, put the JSON body in single quotes:

```powershell
curl.exe -H "Content-Type: application/json" -d '{"mute":true}' http://localhost:64347/api/v1/monitoring
```

For safety, read the current value first when experimenting. Clock, phantom
power, routing, reamp, and volume changes can interrupt audio or create unsafe
signal levels.

### Setter routes and bodies

Enum values must use a key returned by the corresponding `*_options` GET, not
its human-readable label.

| Value being set | POST route | JSON body |
| --- | --- | --- |
| Monitor volume | `/monitoring` | `{"volume": -60.0}` |
| Monitor mute | `/monitoring` | `{"mute": true}` |
| Monitor dim | `/monitoring` | `{"dim": true}` |
| Monitor mono | `/monitoring` | `{"mono": true}` |
| Monitor source | `/monitoring` | `{"source": "main_mix"}` |
| Speaker set A/B | `/monitoring` | `{"ab_speaker_set": false}` |
| Headphone mono | `/monitoring/phones/:index` | `{"mono": true}` |
| Headphone source | `/monitoring/phones/:index` | `{"source": "cue_mix_1"}` |
| Phones 2 speaker set A/B | `/monitoring/phones/2` | `{"ab_speaker_set": false}` |
| Preferred clock source | `/clock` | `{"preferred_source": "internal"}` |
| Sample rate | `/clock` | `{"sample_rate": 44100}` |
| Analog input phantom power | `/input/analog/:index` | `{"48v": false}` |
| Analog input gain | `/input/analog/:index` | `{"gain": 0.0}` |
| Instrument mode | `/input/analog/:index` | `{"inst": false}` |
| Input channel link | `/input/analog/:index` | `{"link": false}` |
| Input connector mode | `/input/analog/:index` | `{"mode": "line_trs"}` |
| Input pad | `/input/analog/:index` | `{"pad": "off"}` |
| Input phase inversion | `/input/analog/:index` | `{"phase_invert": false}` |
| Rear input selection | `/input/analog/:index` | `{"rear": false}` |
| Analog input source | `/input/analog/:index` | `{"source": "<option-key>"}` |
| Analog output volume | `/output/analog/:index` | `{"volume": 0.0}` |
| Analog output link | `/output/analog/:index` | `{"link": false}` |
| AUX output volume | `/output/aux/:side` | `{"volume": 0.0}` |
| AUX output link | `/output/aux/:side` | `{"link": true}` |
| AUX output source | `/output/aux/:side` | `{"source": "main_left"}` |
| AUX reamp | `/output/aux/:side` | `{"reamp": false}` |
| ADAT output source | `/output/adat/:index` | `{"source": "usb"}` |
| S/PDIF output source | `/output/spdif` | `{"source": "usb"}` |
| Loopback source | `/output/loopback` | `{"source": "disabled"}` |

Replace `:index` with a decimal channel number and `:side` with `l` or `r`.
Support is capability-dependent: a syntactically valid setter may return 403
when that control is unavailable on the connected device.

The parent-POST behavior is runtime-confirmed for `monitoring.mute`,
`monitoring.volume`, `clock.preferred_source`, and `input.analog[1].inst`.
The remaining rows follow setter registrations recovered statically but have not
all been exercised on hardware. Option lists and state-only fields such as
`current_source`, `sync`, and `*_options` are read-only.

The plugin also registers specialized POST routes directly on several numeric
leaves. They are included in the static method matrix below. On the tested
Studio, posting an unchanged object value to `/monitoring/volume`,
`/input/analog/5/gain`, and `/output/aux/l/volume` returned
`404 Not Available`; their intended body format or device scope remains
unresolved.

### Complete command examples

```powershell
# Select Cue Mix 1 for headphones 1
curl.exe -H "Content-Type: application/json" -d '{"source":"cue_mix_1"}' http://localhost:64347/api/v1/monitoring/phones/1

# Set input 5 gain to its currently observed value
curl.exe -H "Content-Type: application/json" -d '{"gain":0.0}' http://localhost:64347/api/v1/input/analog/5

# Route ADAT output 1 from USB
curl.exe -H "Content-Type: application/json" -d '{"source":"usb"}' http://localhost:64347/api/v1/output/adat/1

# Disable loopback
curl.exe -H "Content-Type: application/json" -d '{"source":"disabled"}' http://localhost:64347/api/v1/output/loopback
```

### Status codes

| Status | Meaning | Retry? |
| --- | --- | --- |
| `200 OK` | Successful read/write or OPTIONS request. | n/a |
| `200 Request ignored` | Write accepted but was a no-op (value already matched, or documented no-op like `PUT /preset {"saved": false}`). Treat as success; suppress a "value changed" toast if you have one. | n/a |
| `400 Invalid Content Type` | The `Content-Type` header was present but not `application/json`. | No — client bug. |
| `400 Empty Request Body` | `PUT`/`POST` sent with no body on an endpoint that requires one. | No. |
| `400 Request Body Parsing Failed` | Body was not valid JSON (unquoted keys, trailing commas, etc). | No. |
| `400 Invalid Value` | A value was outside the legal range (e.g. `slot: 99`, an unsupported `sample_rate`). | No — validate against `*_options` first. |
| `400 Invalid Value Type` | The JSON type didn't match the endpoint's type (e.g. `"true"` instead of `true`). | No. |
| `403 Device Not Found` | No AudioFuse connected at all, or `targeted-device` names a serial that no longer matches. | Yes — refresh `/devices` first, 500ms then exponential to ~5s. |
| `403 Invalid Device` | The connected device's model doesn't support the API at all (anything but 16Rig/Studio). | No — nothing to retry, surface to user. |
| `403 Invalid Request` | Syntactically valid but semantically rejected: forbidden field combination (e.g. `/preset` `name`+`slot`), a channel that can't be split/joined, or a device-state conflict (e.g. 48V with nothing plugged in). | No — fix the request shape. |
| `404 Not Found` | No route pattern matched the URL. | No. |
| `404 Not Available` | A route pattern matched, but the property/resource isn't exposed by this device or index. | No. |
| `405 Read-Only Resource` | A write was sent to a read-only endpoint. Covers every `*_options` endpoint plus `current_source`, `sync`, `is_at_reference_level`, `preset/names`, `version`, `devices`, and `update/endpoints`. | No. |
| `429 Too many requests` | The firmware rate-limits writes to give the device time to physically apply each change (relay switching, PLL re-lock) before the next arrives. | Yes, but slow down — coalesce rapid writes (e.g. a scrubbing slider), ≥200ms between writes. |
| `500 Unexpected Error` | Handler failed unexpectedly — usually a hardware/driver fault (`/clock/preferred_source` losing track of the device is the most common trigger) rather than a client bug. Also observed for `GET /preset` on Studio and a non-numeric string captured as an `:index`. | At most once or twice (1s, then 3s), then surface and suggest reconnecting the device or restarting AFCC. |

Some responses recorded in this document for unsupported params on the
tested Studio used `403 Device Not Found` where the spec calls for
`404 Not Available` — see the table above and treat `404` as the documented
behavior for "endpoint exists but unsupported on this device/index," and
`403 Device Not Found` strictly for "no device connected at all."

The 429 numbers above are conservative; the sample scripts in
`user documentation/samples/curl/_lib.sh` report, measured on a 16Rig, that
a **preset recall alone keeps the API busy for ~8s**, while sample-rate
changes and speaker-set switches are faster but still often well over 1s —
size your retry/backoff budget accordingly (the samples default to 500ms
between retries, up to 20 attempts, giving ~10s of headroom).

`HEAD` is handled generically, but the server closes the response with the GET
`Content-Length` and no body; some clients report this as a short transfer.

### Security model

The API is **localhost-only with no authentication**: it binds exclusively
to `127.0.0.1` and is unreachable from other machines, but any process on
the host that can reach `127.0.0.1` can read and write the full device
state — there is no token, password, or origin check. CORS is fully open by
design (not merely an artifact of the observed headers below), so any web
page open in any browser on the same machine can call the API too. This is
intentional for the current release: the API is meant as a local developer
control surface, not an internet-facing service.

### OPTIONS and CORS

Writable resource routes respond to OPTIONS with:

```text
Access-Control-Allow-Headers: content-type
Access-Control-Allow-Methods: GET, POST, OPTIONS
```

Some aggregate resources (`/monitoring`, `/clock`, and `/preset`) instead report:

```text
Access-Control-Allow-Methods: GET, PUT, POST, OPTIONS
```

The `GET, POST, OPTIONS` list marks single-child leaves that only accept the
multi-field `POST` form; the `GET, PUT, POST, OPTIONS` list marks parent/leaf
resources that also accept the single-field `PUT` form (see PUT vs POST
above). `/version` and `/devices` do not have an OPTIONS route and return
404.

An OPTIONS success proves that a route *pattern* matched; it does not prove that
the concrete URL identifies a valid resource. For example, both
`/output/analog/volume` and `/output/analog/nonsense` match
`/output/analog/:index`, so OPTIONS returns 200 and advertises GET, PUT, POST,
and OPTIONS. A subsequent GET returns `500 Unexpected Error` because `volume`
or `nonsense` is not a valid numeric index. The actual volume leaf has the form
`/output/analog/:index/volume`.

### Complete statically registered method matrix

This matrix comes from all 82 calls to the route-registration function in
`httpfuse.dll`. Routes are included even when unavailable on the tested Studio,
because another AudioFuse model or configuration may implement them.

| Route pattern | Registered methods |
| --- | --- |
| `/devices` | GET |
| `/version` | GET |
| `/update` | POST, OPTIONS |
| `/monitoring` | GET, PUT, POST, OPTIONS |
| `/monitoring/:param` | GET |
| `/monitoring/volume` | POST, OPTIONS |
| `/monitoring/phones` | GET, POST, OPTIONS |
| `/monitoring/phones/:index` | GET, PUT, POST, OPTIONS |
| `/monitoring/phones/:index/:param` | GET |
| `/monitoring/phones/:index/volume` | POST |
| `/clock` | GET, PUT, POST, OPTIONS |
| `/clock/:param` | GET |
| `/preset` | GET, PUT, POST, OPTIONS |
| `/preset/:param` | GET |
| `/output` | GET, POST, OPTIONS |
| `/output/analog` | GET, POST, OPTIONS |
| `/output/analog/:index` | GET, PUT, POST, OPTIONS |
| `/output/analog/:index/:param` | GET |
| `/output/analog/:index/volume` | POST |
| `/output/aux` | GET, POST, OPTIONS |
| `/output/aux/:side` | GET, PUT, POST, OPTIONS |
| `/output/aux/:side/:param` | GET |
| `/output/aux/:side/volume` | POST |
| `/output/adat` | GET, POST, OPTIONS |
| `/output/adat/:index` | GET, PUT, POST, OPTIONS |
| `/output/adat/:index/:param` | GET |
| `/output/spdif` | GET, PUT, POST, OPTIONS |
| `/output/spdif/:param` | GET |
| `/output/loopback` | GET, PUT, POST, OPTIONS |
| `/output/loopback/:param` | GET |
| `/input` | GET, POST, OPTIONS |
| `/input/analog` | GET, POST, OPTIONS |
| `/input/analog/:index` | GET, PUT, POST, OPTIONS |
| `/input/analog/:index/:param` | GET |
| `/input/analog/:index/gain` | POST, OPTIONS |

The registration operation identifiers `0`, `1`, `2`, and `4` map to GET, PUT,
POST, and OPTIONS. Registration does not prove device support: a handler can
still return `403 Device Not Found`, `404 Not Available`, or
`500 Unexpected Error`.

## Service endpoints

### Devices

```http
GET /api/v1/devices
```

```json
{"devices":["<device-serial-number>"]}
```

Each string in `devices` is the serial number of a connected AudioFuse device.
The serial number is not exposed as a REST subresource:
`GET /devices/<device-serial-number>` returned 404.

### Update submission and events

```text
POST /api/v1/update              {"endpoints": ["/monitoring/mute", "/monitoring/volume"]}
GET  /api/v1/update/endpoints
GET  /api/v1/events
Accept: text/event-stream
```

`/events` only pushes updates for endpoints in an active subscription set —
a stream opened without first subscribing stays open but never emits
anything beyond the heartbeat. Build the subscription with `POST /update`
before opening `/events`; `GET /update/endpoints` lists what's currently
subscribed. The subscription is stateful and tied to the **AFCC process,
not the TCP connection**: if the SSE connection drops and you reconnect,
your subscription is still in force as long as you kept refreshing it — you
don't strictly need to re-POST on reconnect, though doing so defensively is
cheap and covers the case where AFCC itself was restarted while you were
disconnected. It expires after ~50s of inactivity, so refresh it (re-POST
the same or a superset list) every 30–45s.

Once subscribed, a change produces:

```text
event: update
data: {"payload":[{"key":"/monitoring/mute","value":true}]}
```

`payload` is always an array of `{key, value}` pairs — never a single bare
object, even for one change — and `key` is the canonical path without the
`/api/v1` prefix. With nothing subscribed, the stream only shows the
heartbeat (a detail the official docs don't mention at all — this heartbeat
framing is purely a runtime observation from this project, not part of the
published spec):

```text
event: update
: heartbeat
```

Treat unknown `key`s or a future event type other than `update` as
forward-compatible noise to ignore, not an error — the server may add
lifecycle/housekeeping event types in later versions.

**Common integration pitfalls**, most concretely: browser `EventSource`
fires the default `onmessage` handler only for *unnamed* SSE events, but
this API sends everything as a **named** `update` event — a client must use
`addEventListener("update", ...)`, not `onmessage`, or it will silently see
nothing. Beyond that: forgetting to subscribe before opening the stream
(symptom: silence, not an error), letting the subscription lapse past ~50s,
and reconnecting in a tight loop instead of backing off ≥1s (2s is the
reference plugin's choice) are the other common mistakes.

Keep **one shared SSE connection per process**, not one per UI component —
maintain a listener registry keyed by endpoint path so multiple parts of a
client can subscribe to the same key without opening duplicate connections.
Multiple SSE connections from the same client multiply the server's
bookkeeping for no benefit.

## Clock

### Aggregate

```http
GET /api/v1/clock
```

```json
{"clock":{"current_source":"spdif","preferred_source":"internal","sample_rate":44100,"sample_rate_options":{"keys":["44100","48000","88200","96000","176400","192000"],"labels":["44.1 kHz","48.0 kHz","88.2 kHz","96.0 kHz","176.4 kHz","192.0 kHz"]},"source_options":{"keys":["internal","spdif","adat","word"],"labels":["Internal","S/PDIF","ADAT","WORD"]},"sync":1}}
```

### Leaves

| GET route | Type | AudioFuse Studio example |
| --- | --- | --- |
| `/clock/current_source` | string, read-only state | `{"current_source":"spdif"}` |
| `/clock/preferred_source` | string enum | `{"preferred_source":"internal"}` |
| `/clock/sample_rate` | integer enum | `{"sample_rate":44100}` |
| `/clock/sample_rate_options` | keyed option object | See aggregate example. |
| `/clock/source_options` | keyed option object | See aggregate example. |
| `/clock/sync` | integer/boolean-like, read-only state | `{"sync":1}` |

Static route registration and prior runtime tests establish POST partial-object
setters on `/clock`. Changing sample rate or clock source can disrupt audio and
was not tested in this pass.

`/clock/sample_rate`, `/clock/current_source`, `/clock/preferred_source`,
and `/clock/sync` are all SSE-subscribable; `sample_rate_options` and
`source_options` are static enumerations and are not. When changing
`preferred_source`, watch `current_source` and `sync` over SSE rather than
assuming an immediate lock — the device may take a moment, and may fall
back to `internal` if the requested source isn't actually present.

**Schema note**: the official reference's example for `*_options` uses the
field name `values` and native JSON numbers for sample rates
(`{"keys": [44100, 48000, ...], "values": [...]}`), while the runtime
response actually observed against the tested Studio uses `labels` and
string-typed keys (`{"keys": ["44100", "48000", ...], "labels": [...]}`, as
shown above). Both the field name and the number-vs-string typing differ —
write parsing code that tolerates either shape, and always read
`sample_rate_options`/`source_options` at runtime rather than hardcoding
values, since valid rates/sources can vary by firmware and product variant.

## Monitoring

### Aggregate

```http
GET /api/v1/monitoring
```

Observed AudioFuse Studio response:

```json
{"monitoring":{"ab_speaker_set":false,"dim":false,"mono":false,"mute":false,"phones":{"1":{"mono":false,"source":"main_mix","source_options":["main_mix"]},"2":{"ab_speaker_set":"main_mix","mono":"main_mix","source":"main_mix","source_options":["main_mix"]}},"source":"main_mix","source_options":["main_mix"],"volume":"main_mix"}}
```

In this aggregate, `volume: "main_mix"` indicates that the physical volume
knob is assigned to Main Mix; it is not the numeric monitor level. Read
`GET /monitoring/volume` for that level. Other aggregate fields can likewise
describe assignments or expose a reduced representation, so use the leaf
routes below when the exact control value or complete option list is needed.

### Core leaves

| GET route | Type | Studio result/example |
| --- | --- | --- |
| `/monitoring/volume` | float, dB | `{"volume":-76.0}` |
| `/monitoring/mute` | boolean | `{"mute":false}` |
| `/monitoring/dim` | boolean | `{"dim":false}` |
| `/monitoring/mono` | boolean | `{"mono":false}` |
| `/monitoring/source` | string enum | `{"source":"main_mix"}` |
| `/monitoring/source_options` | keyed option object | `main_mix`, `cue_mix_1`, `cue_mix_2` |
| `/monitoring/ab_speaker_set` | boolean | `{"ab_speaker_set":false}` |

Example source options:

```json
{"source_options":{"keys":["main_mix","cue_mix_1","cue_mix_2"],"labels":["Main Mix","Cue Mix 1","Cue Mix 2"]}}
```

`ab_speaker_set` is a plain binary A/B toggle, not a general speaker-set
selector — the API doesn't support more than two speaker sets natively. For
a "Mains → Mids → NS10s → Atmos array" style rig with more than two sets,
drive the actual routing from your client or the DAW and use
`ab_speaker_set` only as a sub-toggle within whichever pair is active.

The following registered monitoring leaves are **16Rig-only** and returned
403 on the tested Studio (correct behavior — they simply don't exist on
that device):

| Endpoint | Purpose |
| --- | --- |
| `/monitoring/reference_level` | The reference level (dB) used by the reference-recall pattern below. RW. |
| `/monitoring/is_at_reference_level` | `true` when the current volume equals `reference_level`. RO — the device sets this, clients can't. |
| `/monitoring/bass_management` | Enables bass-management for the immersive bus. |
| `/monitoring/lip_sync` | A/V sync compensation. |
| `/monitoring/lfe_10db` | LFE channel +10dB boost. |

`reference_level` + `is_at_reference_level` implement a specific workflow:
set the reference level once at calibration time, then a single "A/B
against reference" button just writes that same value to `/monitoring/volume`,
and a UI indicator subscribed to `is_at_reference_level` shows whether
you're currently sitting at it:

```bash
# Calibrate once
curl -X PUT http://127.0.0.1:56894/api/v1/monitoring/reference_level \
     -H 'Content-Type: application/json' -d '{"reference_level": -18.0}'
# A/B button: jump to reference
curl -X PUT http://127.0.0.1:56894/api/v1/monitoring/volume \
     -H 'Content-Type: application/json' -d '{"volume": -18.0}'
```

Static analysis also confirms this immersive solo/mute subgroup, **16Rig-only**:

| Endpoint | Purpose |
| --- | --- |
| `/monitoring/solo_fronts` / `mute_fronts` | Solo/mute the front bed. |
| `/monitoring/solo_left_right` / `mute_left_right` | Solo/mute L/R only. |
| `/monitoring/solo_center` / `mute_center` | Solo/mute the centre channel. |
| `/monitoring/solo_surrounds` / `mute_surrounds` | Solo/mute surround channels. |
| `/monitoring/solo_heights` / `mute_heights` | Solo/mute height channels. |
| `/monitoring/solo_lfe` / `mute_lfe` | Solo/mute the LFE channel. |

All twelve leaves returned `403 Device Not Found` on the tested Studio (per
the 403-vs-404 caveat above, `404 Not Available` is the documented status
for this case). These fields are mutually independent on the API — if a UI
wants to enforce "only one solo active at a time," it must clear the others
itself in the same `POST /monitoring` body; the device won't do it for you.

### Headphones

Collection aggregate:

```http
GET /api/v1/monitoring/phones
```

```json
{"phones":{"1":{"mono":false,"source":"main_mix","source_options":{"keys":["main_mix","cue_mix_1","cue_mix_2"],"labels":["Main Mix","Cue Mix 1","Cue Mix 2"]}},"2":{"ab_speaker_set":false,"mono":false,"source":"main_mix","source_options":["main_mix"]}}}
```

Indexed aggregates are also available:

```text
GET /monitoring/phones/1
GET /monitoring/phones/2
```

```json
{"1":{"mono":false,"source":"main_mix","source_options":{"keys":["main_mix","cue_mix_1","cue_mix_2"],"labels":["Main Mix","Cue Mix 1","Cue Mix 2"]}}}
```

```json
{"2":{"ab_speaker_set":false,"mono":false,"source":"main_mix","source_options":{"keys":["main_mix","cue_mix_1","cue_mix_2"],"labels":["Main Mix","Cue Mix 1","Cue Mix 2"]}}}
```

The collection aggregate returned a reduced `source_options` array for phones
2, while its indexed aggregate returned the complete keyed option object.

For `index` 1 and 2:

| GET route | Studio support |
| --- | --- |
| `/monitoring/phones/:index/mono` | Both indexes; boolean. |
| `/monitoring/phones/:index/source` | Both indexes; string enum. |
| `/monitoring/phones/:index/source_options` | Both indexes; keyed options (`main_mix`, `cue_mix_1`, `cue_mix_2`). |
| `/monitoring/phones/:index/volume` | 404 on both indexes. |
| `/monitoring/phones/2/ab_speaker_set` | Supported; boolean. |

`ab_speaker_set` on phones is documented as **index 2 only** — `GET/PUT
/monitoring/phones/1/ab_speaker_set` returns `404 Not Available` by design,
not because of a Studio-specific quirk; index 1 simply doesn't have this
field. This matches what was observed here (only phones/2 was tested with
it), but is worth stating as a hard rule rather than an incidental finding.

Writes use `POST /monitoring/phones/:index` with the selected property in the
JSON object.

## Analog inputs

Collection and item resources:

```text
GET  /input
GET  /input/analog
GET  /input/analog/:index
POST /input/analog/:index
```

The analog-input aggregate appears to combine current values, capability state,
and option metadata. Use the leaf routes below when requesting an individual
control's current value.

Both collection aggregates returned `500 Unexpected Error` **with usable JSON
bodies**. The bodies differ only in their outer wrapper:

```http
GET /api/v1/input
HTTP/1.1 500 Unexpected Error
Content-Type: application/json
```

```json
{"input":{"analog":{"1":{"48v":null,"inst":false,"pad":"Pad","pad_options":{"keys":["off","pad","boost"],"labels":["Off","Pad","Boost"]},"phase_invert":false},"2":{"48v":false,"inst":false,"pad":false,"pad_options":null,"phase_invert":false},"3":{"48v":false,"inst":false,"pad":false,"pad_options":null,"phase_invert":false},"4":{"48v":false,"inst":false,"pad":false,"pad_options":null,"phase_invert":false},"5":{"gain":false,"link":false,"mode":false,"mode_options":null,"pad":false,"pad_options":null},"6":{"gain":false,"link":false,"mode":false,"mode_options":null,"pad":false,"pad_options":null},"7":{"gain":false,"link":false,"mode":false,"mode_options":null,"pad":false,"pad_options":null},"8":{"gain":false,"link":false,"mode":false,"mode_options":null,"pad":false,"pad_options":null}}}}
```

`GET /api/v1/input/analog` returned the same `analog` object directly beneath
the root instead of beneath `input`.

All tested indexed aggregate requests (`GET /input/analog/1` through
`GET /input/analog/8`) returned HTTP `403 Invalid Request` **with a valid partial
JSON body**. For example:

```http
HTTP/1.1 403 Invalid Request
Content-Type: application/json

{"1":{"48v":null,"inst":false,"pad":"Pad","pad_options":{"keys":["off","pad","boost"],"labels":["Off","Pad","Boost"]},"phase_invert":false}}
```

The status appears to reflect at least one unavailable property in the aggregate
(represented as `null`), while supported properties are still returned. Clients
should parse the JSON body even when this aggregate returns 403, or query the
individual leaf routes to obtain independent status codes.

### Inputs 1–4

| Leaf | Input 1 | Input 2 | Input 3 | Input 4 |
| --- | --- | --- | --- | --- |
| `48v` | 403 | 403 | 403 | 403 |
| `inst` | boolean | boolean | 403 | 403 |
| `pad` | string (`"Pad"`) | string (`"Pad"`) | 403 | 403 |
| `pad_options` | off/pad/boost | off/pad/boost | off/pad/boost | off/pad/boost |
| `phase_invert` | boolean | boolean | 403 | 403 |
| `gain`, `link`, `rear`, `source` | 403 | 403 | 403 | 403 |
| `mode`, `mode_options` | 404 | 404 | 404 | 404 |

Pad options example:

```json
{"pad_options":{"keys":["off","pad","boost"],"labels":["Off","Pad","Boost"]}}
```

The 48V results may depend on device power/capability state and should not be
generalized to every Studio configuration. More generally, the official
docs confirm several of these writes are gated on **physical connector
presence**, independent of index-range support: 48V phantom requires an XLR
actually plugged into that input, instrument mode requires a jack plugged
in, and pad/phase-invert require a connector present at all — the device
rejects the write with `403 Invalid Request` if nothing is plugged in, on
top of (and distinct from) the capability-gating 403/404 responses above.

### Inputs 5–8

| Leaf | Inputs 5–6 | Inputs 7–8 |
| --- | --- | --- |
| `gain` | `0.0` float | `0.0` float |
| `link` | boolean | boolean |
| `mode` | `line_trs` | `line` |
| `mode_options` | `line_trs`, `line_rca`, `phono` | `line`, `bluetooth` |
| `pad` | 403 | 403 |
| `pad_options` | `off`, `pad` | `off`, `pad` |
| `phase_invert`, `source` | 403 | 403 |
| `48v`, `inst`, `rear` | 404 | 404 |

Example mode options for inputs 5–6:

```json
{"mode_options":{"keys":["line_trs","line_rca","phono"],"labels":["Line TRS","Line RCA","Phono"]}}
```

Example mode options for inputs 7–8:

```json
{"mode_options":{"keys":["line","bluetooth"],"labels":["Line","Bluetooth"]}}
```

## Outputs

### Aggregates

```text
GET /output
GET /output/analog
GET /output/analog/:index
```

`GET /output` returned 200 with an `output` object containing the `adat`, `aux`,
`loopback`, and `spdif` aggregates:

```json
{"output":{"adat":{"1":{"source":"usb","source_options":{"keys":["main","cue_1","cue_2","usb","adat_in_1_2"],"labels":["Main","Cue 1","Cue 2","USB","ADAT IN 1-2"]}},"2":{"source":"usb","source_options":{"keys":["main","cue_1","cue_2","usb","adat_in_1_2"],"labels":["Main","Cue 1","Cue 2","USB","ADAT IN 1-2"]}},"3":{"source":"usb","source_options":{"keys":["main","cue_1","cue_2","usb","adat_in_3_4"],"labels":["Main","Cue 1","Cue 2","USB","ADAT IN 3-4"]}},"4":{"source":null,"source_options":{"keys":["main","cue_1","cue_2","usb","adat_in_3_4"],"labels":["Main","Cue 1","Cue 2","USB","ADAT IN 3-4"]}},"5":{"source":null,"source_options":{"keys":["main","cue_1","cue_2","usb","adat_in_3_4"],"labels":["Main","Cue 1","Cue 2","USB","ADAT IN 3-4"]}},"6":{"source":null,"source_options":{"keys":["main","cue_1","cue_2","usb","adat_in_3_4"],"labels":["Main","Cue 1","Cue 2","USB","ADAT IN 3-4"]}},"7":{"source":null,"source_options":{"keys":["main","cue_1","cue_2","usb","adat_in_3_4"],"labels":["Main","Cue 1","Cue 2","USB","ADAT IN 3-4"]}},"8":{"source":null,"source_options":{"keys":["main","cue_1","cue_2","usb","adat_in_3_4"],"labels":["Main","Cue 1","Cue 2","USB","ADAT IN 3-4"]}}},"aux":{"l":{"link":null,"reamp":null,"source":null,"source_options":{"keys":["main","cue_1","cue_2","usb","adat_in_3_4"],"labels":["Main","Cue 1","Cue 2","USB","ADAT IN 3-4"]},"volume":null},"r":{"link":null,"reamp":null,"source":null,"source_options":{"keys":["main","cue_1","cue_2","usb","adat_in_3_4"],"labels":["Main","Cue 1","Cue 2","USB","ADAT IN 3-4"]},"volume":null}},"loopback":{"source":null,"source_options":{"keys":["main","cue_1","cue_2","usb","adat_in_3_4"],"labels":["Main","Cue 1","Cue 2","USB","ADAT IN 3-4"]}},"spdif":{"source":null,"source_options":{"keys":["main","cue_1","cue_2","usb","adat_in_3_4"],"labels":["Main","Cue 1","Cue 2","USB","ADAT IN 3-4"]}}}}
```

As with other top-level aggregates, several nested values and option lists are
`null` or differ from the corresponding child responses. Query the child or
leaf route when its precise current value is required.

On the tested Studio, `GET /output/analog` and the tested indexed analog
aggregates returned `500 Unexpected Error`.

### Auxiliary outputs

Collection aggregate:

```http
GET /api/v1/output/aux
```

```json
{"aux":{"l":{"link":true,"reamp":false,"source":"main_left","source_options":{"keys":["main_left","cue_1_left","cue_2_left","adc_1","adc_2","adc_3","adc_4","daw"],"labels":["Main L","Cue 1 L","Cue 2 L","ADC 1","ADC 2","ADC 3","ADC 4","DAW"]},"volume":0.0},"r":{"link":true,"reamp":true,"source":true,"source_options":null,"volume":true}}}
```

The right-side values in this collection aggregate do not have the types
returned by the corresponding leaf routes; use the leaves for control values.

For side `l` or `r`:

```text
GET  /output/aux/:side
POST /output/aux/:side
GET  /output/aux/:side/volume
GET  /output/aux/:side/link
GET  /output/aux/:side/source
GET  /output/aux/:side/source_options
GET  /output/aux/:side/reamp
```

Observed values were volume `0.0`, link `true`, reamp `false`, and sources
`main_left`/`main_right`. Source choices are side-specific:

```json
{"source_options":{"keys":["main_left","cue_1_left","cue_2_left","adc_1","adc_2","adc_3","adc_4","daw"],"labels":["Main L","Cue 1 L","Cue 2 L","ADC 1","ADC 2","ADC 3","ADC 4","DAW"]}}
```

The right side uses corresponding `*_right` keys and labels for main/cue.

### ADAT outputs

`GET /output/adat` returns an aggregate containing indexes 1 through 8. In the
observed response, indexes 1–3 had source `usb`, while indexes 4–8 had a null
source; every index also included a keyed `source_options` object.

For indexes 1 through 8:

```text
GET  /output/adat/:index/source
GET  /output/adat/:index/source_options
POST /output/adat/:index
```

All eight channels returned source `usb`. Every options list contains `main`,
`cue_1`, `cue_2`, and `usb`. Its fifth pass-through key is paired by channel:

| Indexes | Fifth key/label |
| --- | --- |
| 1–2 | `adat_in_1_2` / `ADAT IN 1-2` |
| 3–4 | `adat_in_3_4` / `ADAT IN 3-4` |
| 5–6 | `adat_in_5_6` / `ADAT IN 5-6` |
| 7–8 | `adat_in_7_8` / `ADAT IN 7-8` |

### S/PDIF output

```text
GET  /output/spdif
GET  /output/spdif/source
GET  /output/spdif/source_options
POST /output/spdif
```

```json
{"source":"usb"}
```

```json
{"source_options":{"keys":["main","cue1","cue2","usb","speakers","spdif"],"labels":["Main","Cue 1","Cue 2","USB","Speakers","S/PDIF IN 1-2"]}}
```

Note the S/PDIF keys use `cue1`/`cue2`, unlike ADAT's `cue_1`/`cue_2`.

### Loopback output

```text
GET  /output/loopback
GET  /output/loopback/source
GET  /output/loopback/source_options
POST /output/loopback
```

```json
{"source":"disabled"}
```

```json
{"source_options":{"keys":["disabled","main_mix","cue_mix_1","cue_mix_2"],"labels":["Disabled","Main Mix","Cue Mix 1","Cue Mix 2"]}}
```

### Analog output family

Static routes exist for:

```text
/output/analog/:index/volume
/output/analog/:index/link
```

On the tested Studio, the volume leaves for indexes 1–2 returned
`404 Not Available`, while indexes 3–4 returned `403 Device Not Found`.
This matches the documented rule for 16Rig: outputs 1 and 2 are the main
monitor outputs and are governed by `/monitoring/volume`, not a per-channel
endpoint — only indexes 3–10 have their own `/output/analog/:index/volume`
and `/output/analog/:index/link`.

The placeholder consumes one complete path segment. Thus
`/output/analog/volume` matches `/output/analog/:index` with the invalid textual
index `volume`; it is not an alternative collection-level volume route. The
volume leaf is `/output/analog/:index/volume`.

## Presets

Static analysis confirms these route names:

```text
/preset/name
/preset/names
/preset/saved
/preset/slot
/preset/save_to
```

All tested preset leaf GETs returned 403, and aggregate `GET /preset` returned
500 on AudioFuse Studio — `/preset` is 16Rig-only, so both are the expected
behavior of testing it against a Studio.

The `PUT`/`POST` semantics on the aggregate are documented: `PUT /preset
{"name": "new name"}` renames the loaded preset; `PUT /preset {"saved":
true}` saves in-RAM edits to the loaded slot; `POST /preset {"slot": 5,
"saved": true}` is "save current state to slot 5" (writes and loads that
slot); `PUT /preset {"saved": false}` is always a no-op
(`200 Request ignored`, since the device — not the client — is the only
thing that can clear the saved flag). Combining `name` with `slot` or
`saved` in the same request is rejected with `403 Invalid Request`; rename
must be its own atomic call. `/preset/save_to` remains unresolved — it
appears only as a route string and does not match any documented behavior
in the official reference, which describes the "save to slot" idiom above
using the plain `/preset` endpoint instead.

## Aggregate and leaf semantics

The tested `/monitoring` aggregate contained values such as:

```json
{"volume":"main_mix"}
```

Here, `"main_mix"` means that the hardware volume knob is assigned to Main,
rather than Cue Mix 1 or Cue Mix 2. It is not the monitor level. The leaf route
`GET /monitoring/volume` returns the numeric level for the currently assigned
mix:

```json
{"volume":-76.0}
```

This demonstrates that aggregate fields may describe control assignment or
capability metadata, while similarly named leaf routes return the controlled
value. The unusual boolean and null fields seen in `/input/analog` may likewise
encode availability or control state; their precise semantics remain under
investigation and should not currently be described as conversion corruption.

## Main Mix and Cue Mix controls missing from HTTP

The `/monitoring` routes cover the monitor-controller functions, not the mixer
shown on AFCC's Main Mix, Cue Mix 1, and Cue Mix 2 pages. AudioFuse Control
Center exposes that mixer internally, but the stock HTTP plugin in this AFCC
release does not expose its controls as HTTP routes.

The omitted subsystem includes, for every mix:

- Analog inputs 1–8: level, pan, mute, solo and stereo-pair state.
- S/PDIF inputs 1–2: level, pan, mute, solo and stereo-pair state.
- ADAT inputs 1–8: level, pan, mute, solo and stereo-pair state.
- USB inputs 1–6: level, pan, mute, solo and stereo-pair state.
- Mixer output level and related mixer state.

AFCC also has local-only mixer metadata such as channel name, visibility,
grouping, peak meters, and the currently selected mixer. Not every internal
parameter is sent to the device.

Static inspection of the Studio parameter definition identifies, for example,
these pan controls:

| Mix | USB input 1 pan parameter | Normalized values |
| --- | ---: | --- |
| Main Mix | 425 | `0.0` hard left, `0.5` center, `1.0` hard right |
| Cue Mix 1 | 633 | `0.0` hard left, `0.5` center, `1.0` hard right |
| Cue Mix 2 | 841 | `0.0` hard left, `0.5` center, `1.0` hard right |

The Main Mix parameter is named `Mixer 1 USB 1 Pan`. AFCC maps it to its
internal `iMAINMIXPANCHANNEL` device control. Thus the requested operation
"USB input 1, 100% right in Main Mix" is internally parameter 425 with normalized
value `1.0`.

Likewise, Main Mix Analog Input 6 mute is internal parameter 326 (`0` off,
`1` on); the corresponding Cue Mix parameters are 534 and 742.

However, `httpfuse.dll` contains no registered `/mixer`, `/input/usb`, mixer
level, `pan`, `balance`, mixer `mute`, or mixer `solo` property implementation.
The address-aware disassembly contains exactly 82 calls to the endpoint
registration function at `0x18000c280`. All 82 occur in the service-constructor
block and load literal route strings; no second or dynamically constructed
route-registration path was found.

Runtime requests distinguish two cases: `/mixer` and `/input/usb/1/pan`
returned `404 Not Found`, while `/monitoring/pan` and `/monitoring/balance`
matched the generic `/monitoring/:param` pattern but returned
`404 Not Available`. Consequently
there is currently no evidence-backed `curl` request for these operations
through the stock HTTP API. AFCC performs them through its separate internal
IPC/device-control path; reverse-engineering that transport is a distinct open
task.

## Evidence from the official Postman collection not matched by any doc

`user documentation/audiofuse-http_api.postman_collection.json` contains
several request shapes that appear in **neither** the official markdown
docs nor this document's own runtime testing. They look like remnants of a
different API generation, in-progress work, or fields later renamed —
treat them as unverified, not as documented behavior:

- `GET /clock/clock_source` and `GET /clock/preferred_clock`, and a
  `PUT /clock {"preferred_clock": "internal", "sample_rate": 44100}` body —
  using `clock_source`/`preferred_clock` instead of the documented
  `current_source`/`preferred_source`. Possibly an older field-naming
  scheme from before the current spec.
- `GET /monitoring/reference` — distinct from the documented
  `reference_level`/`is_at_reference_level` pair; unclear if it's a synonym,
  a predecessor, or an unrelated field.
- A preset `{"store": {"id": 3, "name": "test"}}` POST body, under requests
  named "Save To + Rename" — a shape not mentioned anywhere in the official
  `/preset` combination rules, which instead describe `{"slot": N, "saved":
  true}` for "save to slot." This may be what `/preset/save_to` (see above)
  was originally meant to accept.
- `{"increment_by": -5.0}` POST bodies on what look like gain-style leaves —
  i.e. relative-change semantics. This directly contradicts the documented
  (and reinforced-by-recipes) rule that the API has **no** increment
  semantics and clients must always compute and send absolute values. If
  real, it would be a valuable shortcut; until confirmed against a live
  device, assume it doesn't work and use read-then-write instead.

## Multi-device targeting

A single AFCC instance manages every connected AudioFuse, so there is one
HTTP API regardless of how many devices are plugged in. When more than one
AudioFuse is connected, every parameter endpoint — everything except
`/version`, `/devices`, `/events`, `/update`, `/update/endpoints` — must
carry a `targeted-device` header naming the device serial (from
`GET /devices`):

```http
GET /api/v1/monitoring/volume
targeted-device: AFS-67890
```

With only one device connected the header is optional and AFCC routes to it
implicitly — but send it anyway. A client that always sets `targeted-device`
keeps working unmodified the moment the user plugs in a second AudioFuse
mid-session; one that omits it silently starts talking to "the first listed
device," which may quietly change identity.

## Client design notes

Patterns worth following when building a client, drawn from the official
docs:

- **Read before writing a relative change.** The API has no "increment by"
  semantics. For a "+1 dB" / "turn it down a bit" control: `GET` the current
  value, compute the new absolute value, then `PUT`/`POST` it. Don't assume
  you know the current value from a prior write — another client or a
  hardware knob may have changed it since.
- **Coalesce scene/snapshot recalls into one `POST` per parent resource.**
  e.g. `POST /input/analog {"1": {...}, "2": {...}}` and
  `POST /monitoring {"volume": ..., "dim": ..., "mono": ...}` instead of one
  request per field — fewer round trips, less audible glitching during
  recall.
- **Subscribe to anything you display.** A value only `GET` once will drift
  from reality the moment Control Center, another client, or a physical
  button on the device changes it. Subscribe via `POST /update` for any
  value shown in a UI (LED, meter, toggle state).
- **Treat `200 Request ignored` as success**, not an error — it's normal for
  redundant writes during snapshot recall (e.g. muting something already
  muted).
- **Derive your endpoint surface from `/devices` at runtime**, not from a
  hardcoded device assumption. Hardcoding 16Rig-only endpoints (presets,
  immersive controls) into a client also used with a Studio causes
  `404 Not Available` and a confused user.
- **There is no input/channel mute.** `mute` only exists on `/monitoring`
  (the master output) and `/monitoring/phones/:index`. To fake a per-input
  "mute" for a live-production mute button, drop that input's `gain` to a
  floor value (e.g. `-60.0`) and restore the working value to "unmute" —
  there's no dedicated kill switch on an input itself.
- **There is no `talkback` endpoint.** Implement it as a DAW-side bus or a
  temporary gain change on a talkback mic input driven by your controller.
- **On-device presets (`/preset`, 16Rig only) are a different tool than
  client-side snapshots.** They're few (8 slots), coarse, and capture full
  device state including routing. For per-app scene recall (e.g. "synth jam"
  vs "guitar tracking" configs), store your own JSON snapshots client-side
  and apply them via `POST`; reserve `/preset` for state you want recallable
  from the hardware itself, independent of any client being open.
- **For AI-agent / LLM-driven control**, two tools are enough: `get(path)`
  and `set(path, body, method='PUT')`. Bootstrap the agent's context with
  `GET /devices`, `GET /clock/sample_rate_options`, `GET
  /clock/source_options`, and the device-availability matrix from the
  official endpoint reference, so it knows what values and endpoints are
  legal for the connected model before it ever writes. Clamp agent-driven
  volume changes to a safe range, subscribe to whatever it's controlling so
  its model of the state stays in sync with changes made outside the agent,
  and gate anything destructive (`POST /preset {"slot": N, "saved": true}`
  overwrites that slot) behind explicit confirmation.
- **Realistic numeric defaults from the reference recipes**, worth reusing
  rather than inventing your own: a gain floor of `-60.0` dB as a per-input
  "mute," a `-12.0` to `60.0` dB clamp range for input-gain nudges, `-3.0`
  dB as a default "turn it down a bit" step, and `-18.0` dB as a typical
  reference-level calibration point.

## Open questions

- `/preset/save_to`'s actual request body — not in the official reference,
  and possibly superseded by the `{"slot": N, "saved": true}` idiom (or by
  the Postman-only `{"store": {...}}` shape above).
- Precise numeric ranges (e.g. exact gain min/max per input class) and the
  complete enum-value sets per device/index for `source`, `mode`, and `pad`
  — the general error taxonomy (`400`'s five sub-reasons, `403`'s three
  reason phrases, the retry policy) is now documented; what's left is
  per-endpoint range/enum data, which official guidance says to read from
  each `*_options` endpoint at runtime rather than hardcode.
- Exact firmware-level nuance of state-dependent rejections beyond the
  documented "connector must be physically present" rule for 48V/pad/
  phase_invert/inst (e.g. whether behavior differs across firmware
  revisions).
- Heartbeat interval on `/events` — the official docs don't document a
  heartbeat mechanism at all, so this is unresolved by design, not an
  omission on Arturia's part.
- Capability differences across AudioFuse models other than 16Rig and
  Studio — moot for the current API, since the official docs confirm every
  other model is categorically unsupported (`403 Invalid Device`), not
  merely "different."
- Message format for AFCC's internal IPC/device-control path, needed for mixer
  level, mute, solo, pan, and balance controls absent from the HTTP plugin
  entirely — not covered by the official docs either, which document only
  the same `/api/v1` surface reverse-engineered here.
- The Postman-only `clock_source`/`preferred_clock`/`reference`/
  `increment_by` shapes above — real legacy/future surface, or dead ends?
