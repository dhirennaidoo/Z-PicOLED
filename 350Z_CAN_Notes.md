# 2008 350Z – CAN bus notes (oil temp via Pico + Waveshare CAN HAT)

Findings from sniffing the car with a Pico (MCP2515, Pico-CAN-B) and comparing against the NDS3 / NCDS Android app on a BT OBD dongle.
Status labels used below: **CONFIRMED** (seen on the bus and matched against NDS3), **OBSERVED** (seen, meaning not verified), **INFERRED** (my reading, not tested).

## 1. Summary

- Oil temperature is **not broadcast** on the bus. It is read from the engine ECU with a diagnostic request.
- Oil temp is **data ID 0x111F**, value = `byte - 50` °C. **CONFIRMED** (NDS3 oil screen requests only this ID; reply `4D` = 77 = 27 °C = NDS3 reading of 27).
- Coolant via the same method is **0x1101**, same offset (`4E` = 28 °C = NDS3 water reading of 28). **CONFIRMED** at one cold point.
- The ECU only answers these reads after a diagnostic session is opened with `10 81` then `10 C0`. Without that it replies `7F 22 80`. **CONFIRMED**.
- The −50 offset has only been checked at about 27 °C (cold engine). Re-check against NDS3 when warm.

## 2. Bus basics

| Item | Value |
|---|---|
| Bit rate | 500 kbit/s (`CAN_RATE["500KBPS"] = [0x00, 0x9E, 0x03]` in CANTest.py) |
| Frame type | 11-bit standard IDs |
| Engine ECU request ID | `0x7E0` (2016 dec) |
| Engine ECU response ID | `0x7E8` (2024 dec) |
| Transport | ISO-TP (ISO 15765-2): first byte of each frame is a PCI byte; single frames use `0x0L` where L = payload length; unused bytes padded with `00` |
| Multi-frame | First frame `10 LL ...`, consecutive frames `21, 22, 23 ...`, tester sends flow control `30 00 00` |

The Pico must be in **normal mode** to transmit (it also ACKs frames). The sniffer uses listen-only mode so it cannot disturb the bus.

## 3. Passive (broadcast) frames

IDs below in hex (decimal in brackets). The engine was off and cold in all captures.

| ID | Meaning | Decode used in CANTest.py | Status |
|---|---|---|---|
| 0x2D1 (721) | Vehicle speed | `(d[2]*256 + d[1]) / 10` | Decode inherited from the original code; speed was always 0 in captures |
| 0x23D (573) | RPM and coolant | RPM = `(d[4]*256 + d[3]) * 3.463`; coolant = `d[7] - 50` | Coolant OBSERVED: `0x4D` gave 27 °C. RPM not verified (engine off) |
| 0x233 (563) | Unknown; `d[0]` was also `0x4D` | – | OBSERVED only |

Other IDs seen but not identified: 0x1F9 (505), 0x215 (533), 0x216 (534), 0x280/0x284/0x285 (640/644/645), 0x231 (561), 0x29E (670), 0x2DE (734), 0x354/0x358 (852/856), 0x5C5 (1477), 0x60D (1549), 0x625 (1573), 0x6F1/0x6F2 (1777/1778), 0x794 (1940). Frames with ID 0x002 (`d3 ff .. 07 xx`) look like junk and were ignored.

**0x551 is not oil temperature.** The original code decoded oil from 0x551 (`d[0] - 50`); that ID never appeared in the captures.

## 4. Diagnostic session opening

The ECU starts in a default session in which it rejects data reads. NDS3 opens a session with two requests (all frames are 8 bytes, padded with `00`):

| Step | Request on 0x7E0 | Expected reply on 0x7E8 | Meaning |
|---|---|---|---|
| 1 | `02 10 81 00 00 00 00 00` | `02 50 81 00 00 00 00 00` | Start diagnostic session 0x81 |
| 2 | `02 10 C0 00 00 00 00 00` | `02 50 C0 00 00 00 00 00` | Start session 0xC0 (the one that unlocks reads) |

Byte by byte for request 1: `02` = ISO-TP single frame, 2 payload bytes; `10` = start diagnostic session service; `81` = session type; rest padding. A positive reply echoes the service + 0x40 (`10` → `50`).

Tested on the Pico with only these two requests and nothing else (NDS3 disconnected, ignition on): the ECU accepted both and then answered reads. **CONFIRMED.** `10 03` (UDS extended session) is rejected with `7F 10 12`. **OBSERVED.**

NDS3 sends `10 C0` again when you switch to the monitor page, and does not send tester-present (`3E`). The ECU answers `3E` with `7F 3E 80` when no session is open.

### What else NDS3 does at connect (not needed for oil temp, as far as tested)

1. `02 21 83` → multi-frame reply `61 83 45 56 31 37 42 00 41 06 04 ...` (starts with ASCII "EV17B", probably an ECU ID / calibration string). The tester answers the first frame with flow control `30 00 00`.
2. A walk through data IDs `22 11 00`, `11 20`, `11 40`, `11 60`, `11 80`, `12 00`, `12 20` ... `12 C0`, then `22 13 01` ... `13 13`. **INFERRED:** the reads of the 0xNN00 / 0xNN20 / 0xNN40 style IDs are supported-ID bitmaps (see section 6); the `7F 22 12` replies mark IDs that do not exist.
3. `02 31 00` → `06 71 00 AE 10 ...`. Purpose unknown.

## 5. Poll request (oil temp) and reply

Request, sent on `0x7E0` roughly every 200 ms (NDS3 uses about 105 ms):

```
03 22 11 1F 00 00 00 00
```

| Byte | Value | Meaning |
|---|---|---|
| 0 | `03` | ISO-TP single frame, 3 payload bytes |
| 1 | `22` | Read data by identifier (ReadDataByIdentifier) |
| 2–3 | `11 1F` | Data ID 0x111F (oil temperature) |
| 4–7 | `00` | Padding |

Reply on `0x7E8`:

```
04 62 11 1F 4D 00 00 00
```

| Byte | Value | Meaning |
|---|---|---|
| 0 | `04` | Single frame, 4 payload bytes |
| 1 | `62` | Positive response to service 0x22 (0x22 + 0x40) |
| 2–3 | `11 1F` | Data ID echoed back |
| 4 | `4D` | Value: `0x4D` = 77, **oil °C = 77 − 50 = 27** |

Coolant is the same with `03 22 11 01 ...`, reply `04 62 11 01 4E ...` (78 − 50 = 28 °C).

### Negative replies seen (`03 7F <service> <code>`)

| Reply | Meaning |
|---|---|
| `7F 22 80` | Read refused: not supported in the current session (no session open, or session lost) |
| `7F 22 12` | That data ID is not supported (e.g. 0x1180, 0x1260) |
| `7F 10 12` | Session type not supported (`10 03`) |
| `7F 3E 80` | Tester-present not accepted outside a session |
| `7F xx 78` | Response pending: the ECU needs more time, wait for the real reply (not seen yet, handled in the probe) |

If reads start returning `7F` or stop answering for about 1.5 s, re-run the session opening. The ECU also stops answering entirely when the ignition is off.

## 6. Data ID bitmaps (INFERRED)

`22 11 00` returned `07 62 11 00 E4 41 00 63`. Reading `E4 41 00 63` as a bitmap, most significant bit first, starting at 0x1101, gives supported IDs 0x1101, 0x1102, 0x1103, 0x1106, 0x110A, 0x1110, 0x111A, 0x111B, 0x111F and 0x1120. That includes both 0x111F (oil) and 0x1101 (coolant), which agrees with the captures. The last bit (0x1120) being set matches NDS3 going on to read `22 11 20`, then `11 40`, `11 60`, and getting `7F 22 12` for `11 80`.

Other values seen, meaning unknown: 0x1135 returned a constant `80`; 0x1205 returned `00 45`; 0x1204 returned `00 44`; 0x1303 returned `30 30`. None of these were oil temp.

## 7. CAN hardware filters (MCP2515)

The MCP2515 has two receive buffers. RXB0 uses filters 0 and 1 and mask 0; RXB1 uses filters 2 to 5 and mask 1. Filters can only be written in configuration mode. Filter IDs are written as two bytes: `SIDH = id >> 3`, `SIDL = (id & 7) << 5`. A full 11-bit match uses mask `SIDH = 0xFF`, `SIDL = 0xE0`.

Filters used in `CANTest.py` (`can.SetFilters([SID_RPM, SID_V], [SID_DIAG_RESP])`):

| Slot | Buffer | ID | SIDH | SIDL |
|---|---|---|---|---|
| RXF0 | RXB0 | 0x23D (RPM / coolant) | `0x47` | `0xA0` |
| RXF1 | RXB0 | 0x2D1 (speed) | `0x5A` | `0x20` |
| RXF2–RXF5 | RXB1 | 0x7E8 (ECU replies) | `0xFD` | `0x00` |

Setup order (`MCP2515.SetFilters`):

1. `CANCTRL = 0x80` (configuration mode), short delay.
2. `RXB0CTRL = 0x04` (use filters, roll over into RXB1 when RXB0 is full); `RXB1CTRL = 0x00` (use filters).
3. Both masks (`RXM0`, `RXM1`) = `FF E0` (all 11 bits must match).
4. Write the six filters above; unused slots repeat the last ID.
5. `CANINTF = 0`, `CANINTE = 0x03` (interrupt flags for RXB0 and RXB1).
6. `CANCTRL = 0x04` (normal mode, CLKOUT enabled), so the Pico can transmit and ACK.

Sniffer filters (`CANSniff7E.py`): the same scheme with 0x7E0 and 0x7E8 alternated across all six slots, and `CANCTRL = 0x64` (listen-only, CLKOUT on) so nothing is transmitted or ACKed.

Reading: `CANINTF` bit 0 = RXB0 full, bit 1 = RXB1 full. After reading a buffer, clear only its own bit (the MCP2515 BIT MODIFY command `05 2C <mask> 00`); clearing the whole register can lose a frame sitting in the other buffer.

## 8. Pitfalls found along the way

- **NDS3 and the Pico share the bus.** If the Pico accepts all 0x7E8 frames while NDS3 is also polling, it receives NDS3's replies too, so replies get mismatched with requests. Run the Pico probe with the dongle unplugged.
- **`MCP2515.Send()` in the original code does not work reliably.** It reads the status with `ReadByte(CAN_RD_STATUS)` (a register read at 0xA0 instead of the READ STATUS command) and has an operator-precedence bug in its wait loop. `SendFrame()` replaces it.
- **`MCP2515.Receive()` blocks and handles one frame at a time**, and with no hardware filter on a busy bus it drops most frames. `Poll()` is non-blocking and the hardware filters keep traffic low.
- **The session is not remembered across ignition cycles or ECU sleep.** Open it again whenever reads fail.
- **The OLED refresh is slow** (a separate SPI write per byte). Refreshing it every 100 ms starves CAN reads, so the display updates at about 4 Hz.
- `import CANTest` in a probe script only works if `CANTest.py` is also on the Pico.

## 9. Files (folder `CANTest`)

| File | Purpose |
|---|---|
| `CANTest.py` | Main display: OLED, speed, coolant, oil temp via diagnostic poll |
| `CANTest_before_oil.py` | Backup of the original before oil temp was added |
| `CANSniff7E.py` | Listen-only logger for 0x7E0 / 0x7E8 with timestamps |
| `CANOilProbe.py` | Opens the session and prints oil and coolant temps, no display |

## 10. Open items

- Check oil and coolant (0x111F / 0x1101) against NDS3 with a **warm engine** to confirm the −50 offset over a wider range.
- Check the passive RPM scaling (`× 3.463`) with the engine running.
- Find out how long the ECU keeps a session open with no requests, and whether a tester-present (`3E 00`) is needed at slower poll rates.
- Identify the unknown broadcast IDs listed in section 3, if wanted.
