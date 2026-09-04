# Unofficial Arturia AudioFuse Control Center HTTP API

This document describes the undocumented HTTP API shipped with Arturia
AudioFuse Control Center (AFCC) 2.4.0.347. It is based on static analysis of
`httpfuse.dll`, read-only runtime enumeration, and a small number of confirmed
write tests with an AudioFuse Studio.

The API is undocumented and may change without notice. Device capability varies
by model, input/output index, power state, and possibly firmware.

## Base URL

```text
http://localhost:64347/api/v1
```

The listener belongs to `AudioFuseControlCenterAgent.exe`. API version discovery:

```http
GET /api/v1/version
```

```json
{"version":"1.0.0"}
```

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

Setters use POST with a partial JSON object sent to the parent resource, not to
the leaf URL:

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

## Setting values

There is no `/set` endpoint. Do not POST to the leaf GET route. Send a partial
object to the leaf's parent resource:

```text
GET  /api/v1/monitoring/mute
POST /api/v1/monitoring       {"mute": true}

GET  /api/v1/input/analog/1/inst
POST /api/v1/input/analog/1   {"inst": false}
```

Include `Content-Type: application/json`. A successful setter returns the field
that was applied, for example `{"mute":true}`.

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

| Status | Observed meaning |
| --- | --- |
| 200 | Successful read/write or OPTIONS request. |
| 403 | Route exists, but the device/index does not currently expose that capability. The server may describe this as `Device Not Found`. |
| 404 | No matching registered route. |
| 429 | Device-backed requests were issued too quickly; retry after a delay. |
| 500 | Handler failed unexpectedly. Observed for `GET /preset` on AudioFuse Studio. |

`HEAD` is handled generically, but the server closes the response with the GET
`Content-Length` and no body; some clients report this as a short transfer.

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

The precise PUT operation remains unresolved. `/version` and `/devices` do not
have an OPTIONS route and return 404.

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

### Events

```http
GET /api/v1/events
Accept: text/event-stream
```

Observed stream records:

```text
event: update
: heartbeat
```

No `data:` payload has been observed. The likely client behavior is to treat an
`update` event as invalidation and re-fetch state; this still needs confirmation
from the event handler.

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

## Monitoring

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

The following registered monitoring leaves returned 403 on AudioFuse Studio:

```text
/monitoring/reference_level
/monitoring/is_at_reference_level
/monitoring/bass_management
/monitoring/lip_sync
/monitoring/lfe_10db
```

Static analysis also confirms this immersive solo/mute subgroup:

```text
/monitoring/solo_fronts
/monitoring/mute_fronts
/monitoring/solo_left_right
/monitoring/mute_left_right
/monitoring/solo_center
/monitoring/mute_center
/monitoring/solo_surrounds
/monitoring/mute_surrounds
/monitoring/solo_heights
/monitoring/mute_heights
/monitoring/solo_lfe
/monitoring/mute_lfe
```

It has not yet been exhaustively runtime-probed in this pass and is expected to
be device-dependent.

### Headphones

For `index` 1 and 2:

| GET route | Studio support |
| --- | --- |
| `/monitoring/phones/:index/mono` | Both indexes; boolean. |
| `/monitoring/phones/:index/source` | Both indexes; string enum. |
| `/monitoring/phones/:index/source_options` | Both indexes; keyed options (`main_mix`, `cue_mix_1`, `cue_mix_2`). |
| `/monitoring/phones/:index/volume` | 404 on both indexes. |
| `/monitoring/phones/2/ab_speaker_set` | Supported; boolean. |

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
generalized to every Studio configuration.

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

### Auxiliary outputs

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

On the tested Studio, indexes 1–2 returned 404 and indexes 3–4 returned 403.

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
500 on AudioFuse Studio. The aggregate advertises GET, PUT, POST, and OPTIONS.
The meaning and request body of PUT, plus `/preset/save_to`, remain unresolved.

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

## Open questions

- Exact PUT routes and request bodies, especially preset operations.
- Setter validation, numeric ranges, and error bodies.
- Whether 48V availability changes with external power/device state.
- Exact SSE update semantics and heartbeat interval.
- Authentication, binding/interface exposure, and cross-origin security model.
- Capability differences across other AudioFuse models.
