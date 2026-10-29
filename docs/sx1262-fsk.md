# SETFSK on SX1262 at 868 MHz: bench results and fixes

Branch `sx1262-fsk`, on top of `sx1280-fsk-flrc`, measured on two ProMicro nRF52840 + Ebyte E22P (SX1262) boards
(`nrf52_promicro_diy_sx1262`). As shipped, SETFSK worked up to 50 kbps on this pair and degraded above it (300 kbps:
11/20 ACKs at a 3.9 s median, 2/20 broadcasts received). With one commit every rate from 25 to 300 kbps delivers
20/20, and a direct message is confirmed in **~100 ms** from 150 kbps up (LoRa MEDIUM_FAST: 962 ms).

## Results

Two nodes in one room at −9 dBm chip output (RSSI −6 dBm), region EU_868 at 869.525 MHz, 20 messages of 200 bytes per
test, measured through the serial API. Modes set with `/nexus-local SETFSK <kbps>` (deviation br/2, receiver filter
from Carson's rule, 16-bit preamble: the command's defaults).

| Mode | Before | After |
|---|---|---|
| LoRa MEDIUM_FAST | ACK 20/20 at 960 ms, stream 20/20 (2.2 kb/s) | 962 ms, 20/20 (2.1 kb/s) |
| FSK 25 | 20/20 at 311 ms, stream 20/20 (9.2 kb/s) | **203 ms**, 20/20 (13.7 kb/s) |
| FSK 50 | 20/20 at 303 ms, stream 20/20 (9.4 kb/s) | **147 ms**, 20/20 (21.5 kb/s) |
| FSK 100 | 20/20 at 570 ms, stream 13/20 | **114 ms**, 20/20 (28.5 kb/s) |
| FSK 150 | 20/20 at 607 ms, stream 20/20 (3.2 kb/s) | **103 ms**, 20/20 (30.1 kb/s) |
| FSK 200 | 14/20 at 1780 ms, stream 20/20 (1.6 kb/s) | **101 ms**, 20/20 (29.1 kb/s) |
| FSK 250 | – | **102 ms**, 20/20 (20.5 kb/s) |
| FSK 300 | 16/20 at 2720 ms, stream 12/20 | **99 ms**, 20/20 (30.1 kb/s) |

~30 kb/s is the PC side's ceiling (the Python library hands the node one message every ~50 ms). A 32-bit preamble
made no difference before the fix, so the default 16 bits stays.

## Fixes

- **RX detection (`isActivelyReceiving`)**: only a valid sync word counts. A bare PREAMBLE_DETECTED started
  `receiveDetected()`'s preamble hold, which lasts one max-length LoRa frame of the parked preset (~0.7 s on
  MEDIUM_FAST). That is longer than the fast link's whole retransmission timeout, so the receiver's ACK left after the
  sender had already re-sent, and the duplicates filled the channel. GFSK's preamble detector also trips on noise.
- **Carrier sense (`isChannelActive`)**: busy = 10 dB over the larger of the filter's sensitivity
  (−162 dBm + 10·log10(rxBw), which matches the datasheet's 250 kb/s / 500 kHz row) and the measured noise floor. The
  fixed −100 dBm threshold sat inside the noise of the 312-467 kHz filters behind the module's LNA, so nodes kept
  backing off on an idle channel. This is the rule the SX128x alt modes already use.

## Notes

- Above ~150 kbps the signal is wider than EU_868's 250 kHz band (869.4-869.65 MHz); those rates are for the bench or
  for regions with room for them.
- The boards' LNA saturates easily: at the region's default power (27 dBm, chip at 22 dBm into a ~30 dBm PA) two nodes
  in the same room did not decode each other at all. Use the minimum (`lora.tx_power -9`) on the bench.
