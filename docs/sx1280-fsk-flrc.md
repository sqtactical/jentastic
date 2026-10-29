# SETFSK / SETFLRC on SX1280: bench results and fixes

> **Status (2026-10-29):** the code changes described here are in Jentastic `testing` since `712debb` ("More fixes
> from … bench test for the SX1280"), in some cases extended to the LR11x0/LR20x0. This branch now sits on `testing`
> at `0ea3007` and only adds this document; the original commits, on `26a0451`, are kept on `sx1280-fsk-flrc-26a0451`.

Originally branch `sx1280-fsk-flrc`, based on `testing` at `26a0451`, measured on two ProMicro nRF52840 + Ebyte
E28-2G4M27S boards (`nrf52_promicro_diy_e28`). As shipped, SETFLRC delivered nothing on an SX1280 pair and SETFSK only worked at
250 kbps. With these commits every FSK and FLRC setting the SX1280 supports delivers 20/20, and a direct message is
confirmed in **69-97 ms** (LoRa SHORT_TURBO: 123 ms), with XEdDSA packet signing untouched.

## Results

Two nodes in one room at −18 dBm, 20 messages of 200 bytes per test, measured through the serial API. Modes set with
`/nexus-local SETFLRC …` / `SETFSK …`; LoRa is SHORT_TURBO on LORA_24, 2482 MHz.

| Mode | `testing` as shipped | After the fixes, before the CryptoCell commit | Final |
|---|---|---|---|
| LoRa SHORT_TURBO | ACK 10/10 at 691 ms, stream 5/10 | 222 ms, 20/20 | **123 ms**, 20/20 |
| FLRC 1300 CR 1/1 | 0 delivered | 177 ms | **79 ms** |
| FLRC 1300 CR 1/2 | 0 delivered | 307 ms (18/20 stream) | **81 ms** |
| FLRC 650 CR 1/2 | 0 delivered | 182 ms | **86 ms** |
| FLRC 650 CR 3/4 | 0 delivered | 180 ms | **84 ms** |
| FLRC 260 CR 1/2 | 0 delivered | 194 ms | **97 ms** |
| FSK 2000 / 1000 / 500 | rejected by SETFSK (> 300 kbps) | 168-170 ms | **69-75 ms** |
| FSK 250 | ACK 6/6 at 683 ms, stream 1/6 | 174 ms | **79 ms** |

Final run: all 200 ACKs delivered and every stream 20/20 at 25-31 kb/s (the PC side caps the stream). A wider sweep
with 10 messages per setting passed every FLRC rate (260/325/520/650/1000/1300 kbps) at CR 1/2, 3/4 and 1/1, BT 0 and
0.5, and FSK at 125-2000 kbps with deviations from br/4 to br/2.

For comparison, unmodified upstream Meshtastic 2.8.1 built for the same board gives a 692 ms median ACK on SHORT_TURBO,
the same as `testing` as shipped (691 ms): the slowness was in the 2.8 base, not in the Nexus code.

## What was wrong

| Symptom | Cause | Commit |
|---|---|---|
| FLRC: 0 delivered at any rate | FLRC payloads stop at 127 bytes; Meshtastic frames reach 255 and RX was armed for 255 | FLRC fragmentation (3 fragments max, 1-byte header, 10 ms apart); RX length 127 |
| FLRC CR 1/2 never configured | `beginFLRC()` takes 2/3/4; the code passed `RADIOLIB_SX128X_FLRC_CR_*` (0x00/0x02/0x04), so 1/2 became an invalid 0 and 3/4 configured 1/2 | Pass the shorthand |
| FLRC CR 1/2 and 3/4: CRC errors on every packet | RadioLib's default FLRC sync word `2D 01 4B 1D` (seven zeros in a row) with a 16-bit preamble | 4-byte sync `2D 4B D4 93`, 32-bit preamble floor |
| FSK ≥ 500 kbps: 0 delivered | 2-byte sync `0x12AD`: the receiver locked onto noise (CRC-failed frames at −88…−95 dBm) and missed every real packet | 4-byte sync on SX128x |
| FSK latency grew with bit rate (174 → 614 ms) | Carrier sense at a fixed −100 dBm sits inside the noise of the 1.2-2.4 MHz filters | Busy above max(sensitivity, noise floor) + 10 dB |
| ~100 ms per DM on every mode | 2.8's `HardwareRNG::fill()` ends CryptoCell after the DM nonce, so decrypting the ACK re-runs `SaSi_LibInit()` (~97 ms) | Keep CryptoCell up, power it down after 2 s idle |
| SETFSK capped at 300 kbps on SX1280 | Validation used the SX126x limits | SX128x rates 125-2000 kbps, deviation 62.5-1000 kHz |
| A bad SETFLRC/SETFSK left the node deaf until `OFF` | `begin*()` failed after the reply | Reject what the SX128x cannot take (BT 0.3, preamble > 32 bits, unsupported rates) |

Also: 1 ms CSMA slot and 1000 ms retransmission base in alt modes, one ToRadio per StreamAPI run, GPS default "not
present" on the E28 variant, an X25519 cache (it keeps pairwise secrets in RAM, unlike `ackProofCompute()`'s wipe), and
the ProMicro DIY DFU-by-command fix.

## XEdDSA signing

The ~66 extra bytes on small broadcasts come from upstream 2.8 (`Data.xeddsa_signature`, added only when the signature
fits, never on PKI DMs). Signing plus verification costs 200-300 ms of CPU per signed broadcast on nRF52: with 80-byte
broadcasts 50-200 ms apart the receiver keeps every other one; from 300 ms up it gets all. In LoRa the airtime hides it;
in FSK/FLRC it is the limit for small broadcasts. These commits leave signing unchanged.

## SETFLRC / SETFSK on SX1280

| Command | Valid on SX128x |
|---|---|
| `SETFLRC <br> <cr> [bt] [pre]` | br 260, 325, 520, 650, 1000, 1300 · cr 2 (1/2), 3 (3/4), 4 (1/1) · bt 0 or 2 · pre 4-32 in steps of 4 (32-bit floor applied) |
| `SETFSK <br> [dev] [rxbw] [pre]` | br 125, 250, 400, 500, 800, 1000, 1600, 2000 · dev 62.5-1000 kHz and br/8-2·br (default br/4) · pre 4-32 in steps of 4 |

## JenCore and RNode (LoRa)

Built with `-DNEXUS_ENABLE_JENCORE=1 -DNEXUS_ENABLE_RNODE=1 -DMESHTASTIC_ENABLE_RNODE_BLE=1`:

- **JenCore:** group texts on a `JC:` channel crossed LoRa 5/5 each way between the two nodes. Needs a valid clock.
- **RNode:** over BLE NUS, Reticulum frames crossed LoRa 10/10 each way after the BLE crash fix in this branch.

Open issues, not changed here:
- `configuration.h` evaluates `HAS_BLUETOOTH` / `HAS_WIFI` before `#include "architecture.h"`, so
  `MESHTASTIC_ENABLE_RNODE_BLE` and `_TCP` are always 0 unless forced.
- `Channels::seedJenCoreChannelsOnce()` is never called, so nodes updated from Meshtastic never get the `JC:` channels.
- RNode reports 869.618 MHz / 62.5 kHz / SF5 to the Reticulum host even on 2.4 GHz.
- On SX128x JenCore frames use the Meshtastic LoRa sync word, not MeshCore's 0x12 (only SX126x switches).

## Profiler

Build with `-D PROFILE_PACKETS`. Three seconds after packet traffic stops the node prints `PROF stage=…` (API in,
encode, TX queue, airtime, ISR latency, RX, router, decode, modules, NodeDB, phone queue, API out, logs, XEdDSA
sign/verify, ack proof, X25519) and `PROF thread=…` lines (each OSThread and each module by name) on the USB serial.
