# Backlog

Outstanding work on this fork, and what belongs upstream.

## Offer SX1231/RF69 support to NorthernMan54/rtl_433_ESP

The whole `sx1231-support` branch. Upstream has no RF69 support at all — `grep
RF69 src/rtl_433_ESP.h` on `main` returns nothing — so this is an added module
type rather than a change to existing behaviour. SX127x and CC1101 paths are
untouched.

What it adds:

- `RF_SX1231` and `RF_RF69` module types with their own `RADIO_LIB_MODULE`
  definitions, OOK threshold handling, and `getModuleStatus` output.
- A triggered RSSI read. The RF69 needs the measurement started rather than read
  from a stale register, and a triggered reading follows the OOK envelope, so it
  collapses in the gaps between pulses. `RSSI_PEAK_HOLD` (`src/rtl_433_ESP.h:109`)
  holds the peak for longer than the widest in-packet gap so a sample landing in
  a gap does not tear down a capture in progress.
- `MINIMUM_SIGNAL_DURATION` (`src/rtl_433_ESP.h:116`), which separates the
  decoder accept gate from the dropout bridging window. It defaults to
  `MINIMUM_SIGNAL_LENGTH`, so existing builds are unaffected.
- `RF_MODULE_SPI_SETTINGS` (`src/rtl_433_ESP.h:177`), building the `Module` at
  500 kHz. Prototype wiring here is unreliable at RadioLib's 2 MHz default and
  the chip version read inside `begin()` fails outright above it. Applies to the
  RF69 and SX1231 blocks only.
- `example/SX1231_Receiver`, a minimal reference for the new module type.

Before offering it:

- Decide whether the 500 kHz SPI setting should be conditional. It is right for
  this wiring; a board with short traces could run faster, and upstream may want
  it overridable per build rather than fixed.
- Rebase onto upstream `main` at the time and rebuild. It currently sits on
  `9882050`.
- Watch open PR #211, "Add raw pulse-train callback API". It touches the same
  receive path and would conflict.

## Report the RadioLib RF69 regression to jgromes/RadioLib

`RF69::begin()` cannot succeed on RadioLib 7.7.0, 7.7.1, or current `master`.

`RF69.h:486` has `using PhysicalLayer::setSyncWord;`, which brings the base
overload `setSyncWord(uint8_t*, size_t)` into scope alongside RF69's own
`setSyncWord(const uint8_t*, size_t, uint8_t = 0)`. At `RF69.cpp:77`, `begin()`
calls it with a non-const `uint8_t[]`, so the base is an exact match and wins
over RF69's, which needs a qualification conversion. The base is a stub
returning `RADIOLIB_ERR_UNSUPPORTED` (-25), so `begin()` stops there and never
applies the data shaping, encoding and CRC settings that follow.

7.2.1 through 7.6.0 do not have the `using` line and are unaffected.

The fix is one of: make `syncWord` const at the call site, or qualify it as
`this->RF69::setSyncWord(...)`.

Once that lands, delete the workaround at `src/rtl_433_ESP.cpp:207-216` and
raise the RadioLib floor in `library.json` to the fixed release. Without the
workaround, `RegDataModul` keeps a reserved value (`0b11`) in its
modulation-shaping field, because `setDataShaping` never runs.

## Possibly offer the web receiver as an example

`jbroll/rtl433-web-receiver` is a standalone project that depends on this fork.
If SX1231 support lands upstream it could be contributed as an example instead,
though it is considerably larger than the existing ones and pulls in WiFi, a web
server and an SSE stream.

## Not carried here

`RX_DIAG`, the 1 Hz DIO2 edge and RSSI report used to diagnose the original
receive stall, was deliberately left out of this branch. It is on the local
`SX1231` branch in the original working tree if it is ever needed again.
