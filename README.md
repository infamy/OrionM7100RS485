# OrionM7100RS485
Effort to decode the M7100 RS485 head control protocol.

The full write-up — framing, checksum algorithm, known message types,
addresses seen on the bus — lives in [protocol.md](protocol.md). This
README just covers the basics.

## PHY Layer
PHY layer is pretty straight forward its rs485, 38400baud, with what seems to be a seperate ctrl/priority line used to indicate if the bus is free.

## Data Layer

Some simple data captures shows a pretty easy format. 

Src 1001 Dst counter msg 1017 checksum 1016 dst  
96 1001 E1 FC 0000 1B 5B48564F4C203D203238 1017 860F 1006 E1 96  
This will display "VOL = 28" on the second line of the LCD.  
Decoding the msg portion is very straight forward. The checksum is a different story. 

CRC in this case is on 1001 E1 FC 0000 1B 5B48564F4C203D203238 1017 and uses CRC16 with the endianess flipped. 

https://crccalc.com/ crc16/x-25 is the right one.. gives 0x0F86 which is you flipp is 860F... CRC solved!

See [protocol.md](protocol.md) for the confirmed algorithm (verified against
every sample in `capture_rep_vol16.txt` plus poll and encoder packets), the
packet framing rules, and the message types decoded so far.

## Sample Captures
This repo includes some captures including one of a repeat volume message to help figure out the CRC algorythm used.

- `capture_rep_vol16.txt` — repeated "VOL = 16" messages, differing only in
  `CNT`/checksum — used to crack the CRC.
- `capture_withnotes.txt` — longer capture with some display lines
  annotated inline (`-->`).
- `capture2.txt` — longer mixed capture (display updates, encoder reports,
  keepalive polls to a second, unresponsive head-unit address).

## Tools
- `crcbrute.go` — the brute-force CRC-16 polynomial/seed search used to
  originally crack the checksum. Superseded now that the algorithm is
  known (see protocol.md), kept for reference.

## Next Steps
Plan is to tap the bus with a Waveshare ESP32-S2 7" touchscreen board (has
RS485 onboard) — first as a passive sniffer/analyzer to validate and extend
this doc against a live bus, then eventually as a full touchscreen
replacement head unit that speaks the protocol natively.
