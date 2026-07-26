# M7100 / Orion Head-Unit Protocol

Reverse-engineered from bus captures of the RS485 link between an Orion M7100
radio body and its detachable head unit. This documents everything confirmed
from the captures in this repo. Anything not directly confirmed by a capture
is marked as a hypothesis.

## Physical Layer

RS485, 38400 baud, 8N1, plus a separate `RQST` line, outside the RS485 pair
itself, used for bus reservation/arbitration between devices — see `RQST`
under Firmware-Derived Details for the confirmed behavior.

DB37 connector between body and head:

| Pin | Signal |
|-----|--------|
| 4   | RS485+ |
| 5   | RS485- |
| 17  | RQST   |
| 7   | GND    |

## Devices Seen on the Bus

Every packet has a single-byte destination and source address. Three
addresses were observed in the captures:

| Addr | Role |
|------|------|
| `E1` | Radio body (tuner/amp — the "brain"). Drives the display and polls the head unit(s). |
| `96` | Head unit (LCD + volume knob). Responds to `E1`'s polls, reports knob movement, receives display updates. |
| `87` | A second head-unit address that `E1` polls but that never acks in these captures — see below. |

`E1` periodically sends a 2-byte `01 FF` message to `87` and gets no `1006`
ack back; the `CNT` byte stays fixed across many consecutive retries of that
exact same message (matching the "counter stays the same on retry" rule
below), then jumps ahead once `E1` gives up and moves on to the next poll
cycle. `96`, by contrast, always acks immediately. This looks like `E1`
polling a fixed set of possible head-unit addresses to see what's plugged
in — consistent with the multiple head-unit connector variants (RJ45, RJ90,
edge-connector, DB37) in this project's hardware notes. In the captured
session only `96` was actually connected.

## Packet Framing

```
<DST><1001|1002><SRC><CNT><MSG...><1003|1017><CRC:2 bytes>
```

- **DST** — destination address, 1 byte.
- **1001 / 1002** — 2-byte frame marker. `1001` starts a message; `1002`
  marks a continuation packet (see Multi-Packet Messages below).
- **SRC** — source address, 1 byte.
- **CNT** — message counter, 1 byte. Increases by 2 for each new logical
  message; stays the same when a message is being retried (e.g. no ack was
  received). A continuation packet (`1002`) uses `CNT+1` relative to the
  initial packet's `CNT`.
- **MSG** — payload, variable length. See Message Types below.
- **1003 / 1017** — 2-byte frame marker. `1017` means "checksum follows,
  and that's the whole message." `1003` means "checksum follows, but
  there's a continuation packet coming" (used when a message doesn't fit in
  one packet — mostly 2-line display updates).
- **CRC** — 2 bytes, see Checksum below.

### Ack Packet

```
<1006><DST><SRC>
```

4 bytes total, no payload, no checksum. Sent in reply to a message packet,
with `DST`/`SRC` swapped relative to the message it's acking (i.e. it's
addressed back to the original sender, from the original recipient).

Example:
```
96 1001 E1 B8 00 00 1B 5B 48 56 4F 4C 20 3D 20 31 36 1017 30 84   <- E1 to 96: "VOL = 16"
1006 E1 96                                                        <- 96 acks back to E1
```

### Multi-Packet Messages

When a message is too long for one packet, it's split: the first packet
uses `1001 ... 1003` and a second packet uses `1002 ... 1017`. The two
`MSG` payloads are simply concatenated. This is mainly used for updates
that rewrite both LCD lines in one logical operation.

Full worked example (clearing line 1 to "BLANK" and writing "VOL = 19" to
line 2):

```
96 1001 E1 04 0000 1B5B48 424C414E4B202020 1B5B48 564F4C203D2031 1003 E759
96 1002 E1 05 39 1017 0594
1006 E1 96
```

Concatenated payload: `00 00` + `ESC[H` + `"BLANK   "` + `ESC[H` +
`"VOL = 19"` (the digit `9` lands in the second packet).

## Checksum

The checksum is **CRC-16/X-25** (poly `0x1021` reflected = `0x8408`,
init `0xFFFF`, refin/refout, xorout `0xFFFF` — the standard "CRC-16/X-25" /
CCITT variant used by X.25, PPP, etc.), computed over every byte **from the
`1001`/`1002` marker through the `1003`/`1017` marker inclusive** (i.e.
everything except the leading `DST` byte and the checksum itself). The
resulting 16-bit value is then **byte-swapped** to produce the two bytes
seen on the wire.

Reference implementation (Python):

```python
def crc16_x25(data: bytes) -> int:
    crc = 0xFFFF
    for b in data:
        crc ^= b
        for _ in range(8):
            crc = (crc >> 1) ^ 0x8408 if crc & 1 else crc >> 1
    return crc ^ 0xFFFF

def m7100_checksum(marker_thru_marker: bytes) -> bytes:
    crc = crc16_x25(marker_thru_marker)
    return crc.to_bytes(2, "big")[::-1]  # byte-swapped
```

This was confirmed against every sample in `capture_rep_vol16.txt` (eight
repeats of the same "VOL = 16" message, each with a different `CNT` and
thus a different CRC), the multi-packet example above, an `01 FF` poll
packet, and a knob-movement report packet — all match exactly.

`crcbrute.go` in this repo is the brute-force tool used to originally find
this (it searches CRC-16 polynomial/seed space against a known
plaintext/checksum pair); it's kept here for reference/provenance now that
the answer is known. It's since been independently confirmed by
disassembling a head-unit firmware image — see Firmware-Derived Details
below — which contains the exact same nibble-wise CRC routine (init
`0xFFFF`, final bitwise NOT) operating on the marker-to-marker span.

### Byte-Stuffing

Any literal `0x10` byte occurring inside the header/payload/trailing-marker
span is transmitted **twice in a row** (`10 10`) so a receiver scanning for
the next `0x10` marker byte can't be fooled by data that happens to equal
the marker. The CRC is computed over the stream *after* stuffing (i.e. over
exactly what's on the wire, doubled bytes included) — confirmed from the
firmware's transmit- and receive-side framing code. No stuffed byte has
turned up in this project's captures yet since none of the sampled payloads
happen to contain a literal `0x10`, but the mechanism is unambiguous in the
firmware.

## Message Types

The `MSG` payload's leading byte(s) act as an opcode. Three have been
identified:

### `01 FF` — Poll / Keepalive

2-byte payload, no other fields. Sent periodically by `E1` to each known
head-unit address (`96`, `87`) and observed being sent by `96` back to
`E1` as well. Content is always `01 FF` — doesn't appear to carry state,
looks like a "you there?" liveness check.

### `00 00 ...` — Display Update

```
00 00 [optional ESC sequence] <ASCII text>
```

Sent from `E1` (body) to `96` (head) to update the LCD. The two leading
`00 00` bytes are constant in every sample and are likely a
command/line-select opcode pair (unconfirmed which nibble means what —
all captured examples target the same visible line).

Text is plain ASCII, optionally preceded by an escape sequence that
repositions the cursor before the text is drawn:

| Bytes | Meaning |
|---|---|
| `1B 5B 48` | `ESC[H` — ANSI "cursor home" |
| `1B 5B 32 48` | `ESC[2H` — ANSI cursor to row 2 |
| `1B 40 <3 ASCII digits> 68` | `ESC@nnnh` — custom, non-ANSI. Seen setting values like `146`, `188`, `173`. Hypothesis: drives a segment/bargraph indicator (signal meter, EQ, etc.) rather than text — not confirmed. |

Examples decoded:
- `00 00 1B5B48 "VOL = 16"` → writes "VOL = 16" at cursor-home.
- `00 00 1B5B48 "BLANK   "` → clears the line to 8 spaces (title "BLANK" is this project's label for it, not part of the payload).
- `00 00 "VOL = 25"` (no escape prefix) → text-only update, continuing from wherever the cursor was left by a prior escape sequence.

### `05 01 ...` — Input Report (Encoder / Button)

```
05 01 <tag> <data:3 bytes>
```

Sent from the head (`96`/HHC) to the body (`E1`) — the reverse direction
of the display updates. This is the head reporting a raw physical input
change; see Firmware-Derived Details below for the confirmed breakdown of
`tag`/`data` for both the volume knob (`82 81 <value> 00`) and discrete
buttons (`C0`/`80 <button-code> 00 00`, press/release).

## Firmware-Derived Details

The findings above come from bus captures alone. This section adds detail
pulled from disassembling a hand-held-controller (HHC) firmware image — a
lower-button-count remote that wires onto the same bus as the main head
unit and, per the boot code, appears to share firmware with it (role is
chosen by configuration, not by a separate build). None of the firmware or
disassembly is redistributed here — findings are described, not the code
itself.

### RS485 / UART

Bus I/O goes through the on-chip SCI in polled mode:

- `TDR` (transmit data register) at short absolute address `0xFFB3`.
- `SSR` (serial status register) at `0xFFB4` — bit 7 (`TDRE`) is polled
  before writing each byte, then cleared to kick off the send.
- A GPIO bit at `0xFFD2` bit 1 is set immediately before a send starts and
  cleared once the transmit cycle is over — this is the local RS485
  transceiver's driver-enable, distinct from `RQST` below (each device's
  own chip needs this regardless of what's happening bus-wide).
- A control register at `0xFFB2` is written `0x20` right before
  transmitting and `0x24` immediately after — consistent with a
  transmit-then-listen turnaround (enable TX only while sending, then
  switch back to TX+RX/idle).

### `RQST` — Bus Reservation Line

Settled directly, not just inferred: the firmware has a small diagnostic
routine, reachable from a larger "dump all I/O state" debug report over
the separate debug UART, whose entire job is printing the literal string
`RQST=` followed by `1` or `0` — read straight off a GPIO port (short
absolute address `0xFFD7`, bit 0). That pins down which physical pin
`RQST` is internally.

Its actual use, traced through the transmit-scheduling state machine:

- **Before a device starts sending anything**, a routine checks that bit —
  idle reads as `1`. Only if it's idle does the device proceed: it drives
  the bit to `0` (claiming it) and *then* asserts its local driver-enable
  (`0xFFD2` bit 1) and starts transmitting. If the bit reads `0` (someone
  else already has it), the routine just returns without transmitting.
- **After the send cycle ends** — whether that's a normal ack, one of the
  `0x15`/`0x16` reply types, or a timeout — every one of those outcome
  paths calls the same cleanup routine, which drops the local
  driver-enable and drives the bit back to `1` (releasing it).

In other words, `RQST` is a shared bus-reservation/busy line, separate
from and checked *before* each device's own local RS485 driver-enable —
exactly the kind of out-of-band arbitration signal you'd want on a
half-duplex multidrop bus with more than one potential talker (the body,
plus whichever heads/HHCs are attached), since RS485 itself has no
built-in collision detection. Idle is logic `1`, asserted/busy is logic
`0` — consistent with an open-drain, pulled-up shared line where any
device can pull it low, though that's an inference about the electrical
design, not something visible from firmware alone. Worth confirming with
a scope before actively driving it from other hardware: if it turns out
to be push-pull rather than open-drain on the real bus, two devices
driving it at once would contend.

There's also a second, separate UART (registers around `0xFFBB`/`0xFFBC`)
used only for a boot-time debug message — see Addressing below. It's
distinct from the bus UART and could be a useful tap point on real hardware
for boot diagnostics.

### CRC, Confirmed in Code

The firmware has a dedicated CRC routine matching everything already
derived from the captures: seed `0xFFFF`, a per-byte nibble-wise
shift/XOR update (no lookup table — this is a code-size-optimized
CRC-16/CCITT variant, not a table-driven one), and a final bitwise `NOT`
(equivalent to XOR `0xFFFF`) on completion. It's invoked with a buffer
pointer and length taken directly from the assembled, already-stuffed
frame (marker byte through trailing marker), and the result is written out
low-byte-then-high-byte — i.e. byte-swapped — exactly matching the wire
behavior documented above. This independently confirms the CRC-16/X-25
match found via `crcbrute.go` against the captures; it isn't a coincidence.

### Addressing

- The device's own bus address lives in a single RAM byte, set by a
  dedicated setter routine and used everywhere the firmware needs to fill
  in `SRC` (both for messages and acks).
- At boot, a config record (read via a bit-banged interface, integrity
  checked with the *same* CRC-16 routine used for bus framing) supplies a
  requested address. A validation routine only accepts it if it's one of
  `0x69`, `0x78`, or `0x87` — any other value (including a failed/missing
  config record) falls back to `0x96`.
- `0x96` is the address this project's captures show acting as the main
  head unit. That the HHC's own fallback is `0x96` supports the
  shared-firmware theory: this looks like one firmware image that can act
  as the main head or as one of up to three HHCs, with role picked by an
  external strap/config rather than by build.
- `0x69`, `0x78`, `0x87`, `0x96` are evenly spaced by `0x0F` (15) — and so
  is `E1`, the body's address seen in every capture:
  `0x69, 0x78, 0x87, 0x96, 0xA5, 0xB4, 0xC3, 0xD2, 0xE1`. That's 9 evenly
  spaced slots total. Whether the other four (`0xA5`, `0xB4`, `0xC3`,
  `0xD2`) are used by real products (amp, CD changer, etc.) or just
  reserved is unknown — but the address space clearly isn't random.
- This also resolves why `0x87` never acked in this project's captures:
  it's a valid HHC address, just not the one attached during that
  particular session (`0x96`, the main head, was).
- The firmware prints a one-line boot debug message over the separate
  debug UART: `NA=<addr>\r\n` (own address, printed once at startup) —
  worth watching for if you tap that second UART on real hardware.

### Byte-Level Receive State Machine

The receiver is a small state machine (~10 states) advanced one byte at a
time, matching the framing already described from the wire: idle/expect
`DST` → expect `0x10` → expect marker-type byte → `SRC` → `CNT` → payload
(watching for `0x10` to start either a stuffed pair or the trailing
marker) → checksum bytes → verify/dispatch. Two details not visible from
the captures alone:

- **Address filtering happens at both ends of the header.** The `DST` byte
  is checked against the device's own address *and* a second
  "also-accept" address held in a separate RAM slot (value not resolved —
  it's read-only in this firmware image, so its initializer lives outside
  the disassembled code, possibly a broadcast/group address). The `SRC`
  byte of an accepted message is separately checked against a short table
  of up to three recognized peer addresses. A non-matching `DST` causes
  the whole in-progress frame to be dropped and the parser reset.
- **There are more terminal/reply marker types than `1006`.** The
  marker-type dispatch (the byte right after `0x10` at the start of a
  frame) explicitly recognizes `0x01`, `0x02`, `0x06`, `0x15`, and `0x16`
  — not just `01`/`02`/`06`. `0x15` and `0x16` behave like `0x06`
  (immediately terminal, no `SRC`/`CNT`/payload/checksum follow) but set a
  different internal flag, i.e. they're a *different kind* of
  acknowledgment/reply, not just an alias for ack. Neither has shown up in
  this project's captures yet. (Interestingly, this project's very first
  scratch notes on the protocol mentioned a `1016` marker before that
  detail got lost in later write-ups — the firmware confirms it's real.)
  `0x03`/`0x17`, the "checksum follows" markers, are recognized later, at
  the payload stage, as expected.

### Button & Encoder Reporting — the Heads Really Are "Dumb"

Traced the transmit side for `05 01 ...` messages back to the actual GPIO
reads, which settles the "is the head just a display" question: **yes**.
There is no button logic on the head end beyond debouncing a pin — no
concept of what a button *means*. Every physical input change is shipped
to the body as a raw event, and the body owns all interpretation and
whatever gets written back to the display.

**Volume knob** — `05 01 82 81 <value> 00`. Built by a small function that
always emits the fixed bytes `82 81`, with the caller supplying a single
value byte (an encoder tick count/delta). This is the function whose
output shows up in the captures as `0501 8281 xx 00`.

**Discrete buttons** — `05 01 <tag> <code> 00 00`, one message per edge:

- `tag = 0xC0` → **press**. `tag = 0x80` → **release**. Confirmed directly
  from a GPIO-edge-detector routine tied to one physical button (port
  `0xDA` bit 6 in that instance): pin goes low (active-low button, pulled
  up normally) → send `05 01 C0 <code> 00 00`; pin goes back high → send
  `05 01 80 <code> 00 00`. Press detection is debounced (the pin has to
  read low for a set number of consecutive scans before the press event
  fires); release fires as soon as the pin reads high again after a
  latched press.
- `0xC0` and `0x80` are literally `0x80` with bit 6 set or clear
  (`0x80 | 0x40 = 0xC0`) — the firmware builds them by OR-ing a `0x40`
  ("pressed") or `0x80` ("released") flag onto a per-button base byte that
  is always `0x80` in the table described below, rather than treating
  press/release as unrelated constants.
- `code` identifies *which* button. Simple, older-style buttons have their
  own small hand-written sender function with a hardcoded one-byte code
  (`0xBD`, `0xBE`, ... seen so far). A larger set of buttons goes through
  one shared, table-driven sender instead: each button has an index
  (`0x0F`, `0x14`, ... seen so far), and a ROM table of small
  `[tag-base, code-hi, code-lo, 0x00]` records (~20+ entries, every
  observed `tag-base` byte is `0x80`) supplies the actual 2-byte code sent
  for that index. That there's a ~20-entry table at all, on firmware for a
  *reduced-button* remote, is more evidence the main head and the HHC
  share one firmware image with a superset button table — the HHC's board
  just doesn't wire up GPIOs for the buttons it physically lacks.
- None of the button codes have been matched to physical labels (Option /
  Menu / Clear / Scan / etc. from the companion `m7100_ui` mockup) — that
  needs a live capture of an actual button press, see Open Questions.

## Open Questions

- Exact semantics of the `00 00` display-update opcode bytes (line
  select? attribute flags?).
- The `ESC@nnnh` custom escape sequence — what it drives and how `nnn` is
  scaled.
- Which physical button maps to which `05 01` button code — need a live
  capture of actual button presses (see Button & Encoder Reporting above;
  the framing is solved, the code↔label mapping isn't).
- Whether the head-to-body encoder deltas (`05 01 82 81 <value> 00`) are
  relative ticks or an absolute position, and the scale/direction.
- Whether `RQST` is genuinely open-drain on the real bus wiring (assumed
  from its logic-level behavior in firmware, not confirmed electrically) —
  worth checking with a scope before driving it from other hardware.
- What `0x15`/`0x16` replies actually mean (NAK? busy? checksum-error?) —
  need a capture where one actually occurs.
- Whether the four unaccounted-for addresses in the `0x69`..`0xE1` family
  (`0xA5`, `0xB4`, `0xC3`, `0xD2`) correspond to real M7100 accessories.
- What the "also-accept" secondary `DST` address and the "recognized
  peer" table entries actually contain — not resolved from code alone
  since they're populated outside the disassembled `.text` sections.

## Files in This Repo

- `protocol.md` — this document.
- `capture_rep_vol16.txt` — repeated "VOL = 16" display messages with
  differing `CNT`, used to isolate the checksum algorithm.
- `capture_withnotes.txt` / `capture2.txt` — longer raw captures with
  mixed message types (display updates, encoder reports, polls),
  `capture_withnotes.txt` has some inline `-->` annotations of decoded
  display text.
- `crcbrute.go` — brute-force CRC-16 polynomial/seed finder used to
  originally crack the checksum (Go, needs `github.com/howeyc/crc16` and
  `github.com/cheggaaa/pb/v3`).
