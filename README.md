# esphome-uponor

ESPHome external component for a CC1101 868MHz transceiver talking to
Uponor Smatrix Wave (X-165/I-167) underfloor-heating thermostats, on an
ESP32.

This is a companion to [idoiten/uponor-x165](https://github.com/idoiten/uponor-x165),
which has the RTL-SDR-based Python receiver this project's RF parameters
and protocol knowledge come from (frame structure, CRC, TLV tags,
room-name mapping). That repo is the reference implementation to compare
against while bringing the CC1101 path up.

**Current phase: receive-only bring-up.** The CC1101 is configured to
receive and hardware-validate (sync + CRC) the same RF frames the
RTL-SDR already decodes, and raw packets are logged as hex for
comparison. No TLV parsing, no sensors, and no transmit yet -- those are
follow-up phases once raw reception is confirmed on real hardware. The
longer-term goal is also being able to *write* setpoint changes to the
thermostats via the CC1101.

## 1. Wiring (SPI)

Hardware used: an SZHJW CC1101 868MHz module, which has 2.0mm pin pitch
on its 8-pin SPI side -- wired to the ESP32 via a 2.0mm-to-2.54mm adapter
board and JST PH2.0 connector cables.

| CC1101 pin | Function                 | ESP32 (VSPI, default) |
|------------|---------------------------|------------------------|
| VCC        | 3.3V                      | 3V3 (**not** 5V!)      |
| GND        | GND                       | GND                    |
| SI (MOSI)  | SPI data in               | GPIO23                 |
| SO (MISO)  | SPI data out              | GPIO19                 |
| SCK        | SPI clock                 | GPIO18                 |
| CSN        | SPI chip select           | GPIO5                  |
| GDO0       | RX interrupt (packet/sync)| GPIO4                  |
| GDO2       | (unused for RX-only)      | leave unconnected       |

Pitfalls:

- **3.3V, not 5V.** The CC1101 chip cannot tolerate 5V. Many ESP32 dev
  boards have a 5V/VIN pin right next to 3V3 -- double-check with a
  multimeter before powering up, especially since the adapter board
  makes it easy to pick the wrong row.
- **Short, thick VCC/GND wires.** The CC1101 draws short current pulses
  during TX/calibration; long thin JST leads cause voltage sag that shows
  up as random packet errors. Keep the power leads as short as practical.
- **Antenna.** If the module doesn't have a fixed helical/chip antenna: a
  straight ~8.2cm wire (quarter-wave for 868MHz) in the ANT pin works for
  bring-up. Swap in a real 868MHz whip antenna (SMA) once reception is
  confirmed working, or range will be poor.
- **GDO0 is the only interrupt pin needed** for RX: it's configured
  (IOCFG0 = 0x06) to go high once a valid packet (CRC OK) has been
  received, and low again once the first byte is read out of the RX
  FIFO.
- The pins above (GPIO23/19/18/5/4) just match ESP32's hardware VSPI bus
  directly -- freely change them in YAML if they clash with something
  else on your board.

## 2. Why these register values

Derived from what's already known about the protocol (see
idoiten/uponor-x165's `uponor_smatrix_wave_x165/{protocol,crc,dsp}.py`):

| Parameter         | Value                            | Source |
|--------------------|-----------------------------------|--------|
| Carrier            | 868.25 MHz                        | `receiver.py --frequency` default (868_250_000) |
| Modulation         | 2-FSK/GFSK (GFSK tried first)     | two-tone FM observed in `dsp.py` |
| Baud rate          | ~38,378-38,382 baud               | `estimate_bitrate()` in logs -> matches TI's standard "38.4 kBaud" preset almost exactly |
| Frequency deviation| ~16.5-20 kHz                      | tone separation in logs (`robust_two_tones`) |
| Sync word          | `D3 91` repeated (`D3 91 D3 91`)  | `PREAMBLE_SYNC` in `protocol.py` -- exactly CC1101's native "32-bit sync" (a 16-bit word sent twice) |
| Preamble           | `AA AA AA AA` (4 bytes)           | same constant |
| Length byte        | byte after sync = payload length (excl. CRC) | `declared = raw[8] + 11` in `protocol.py` |
| CRC                | CRC-16, poly 0x8005, init 0xFFFF, non-reflected | `crc.py` -> **identical** to CC1101's hardware CRC |

The last two rows are good news: CC1101's variable packet length mode
(`PKTCTRL0.LENGTH_CONFIG=1`) matches this protocol as-is -- the length
byte CC1101 looks for is exactly `raw[8]`, and the hardware CRC
(`CRC_EN=1`) is bit-for-bit the same algorithm as `crc16_cms()`. That
means the chip can validate packets entirely on its own (the RX FIFO
only yields valid packets, flagged via `CRC_OK` in the status bytes) --
no CRC implementation needed in C++ at all.

See `components/cc1101_uponor/registers.h` for the full register table
with per-register rationale, and `cc1101_uponor.cpp` for how it's
applied at startup.

## 3. Bring-up plan

1. Wire as above, flash `example-bringup.yaml`.
2. The component reads `PARTNUM`/`VERSION` over SPI at startup and logs
   them. If that doesn't work (0x00 or 0xFF back), it's almost always
   miswired SPI or wrong voltage -- not the RF configuration.
3. Once SPI is verified: the component enters RX and logs every packet
   CC1101's hardware accepts (sync found + CRC OK) as hex. Keep a
   receiver (the RTL-SDR log) running at the same time and compare -- the
   hex bytes should be identical to what the RTL-SDR receiver already
   decodes.
4. If nothing is received at all: try `MDMCFG2 = 0x03` (plain 2-FSK)
   instead of `0x13` (GFSK) -- see the comment in `registers.h`. If
   packets arrive but often with a bad CRC: increase `DEVIATN` slightly
   (gives the chip more margin against drift) or double-check the
   antenna/distance to a thermostat.
5. Next step once raw byte reception is confirmed (separate task): port
   the TLV parsing (`protocol.py`) to C++ and publish temperature/setpoint
   per room as ESPHome sensors, using the same room-name mapping as
   `rooms.py`.
