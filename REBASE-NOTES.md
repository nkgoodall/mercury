# Rebase notes: crypto-2.0 onto mercuryv2

**Branch**: `crypto-2.0-rebased` — #410's `crypto-2.0` (`bd0228ef`, 2 commits by
Rafael Diniz) rebased onto `mercuryv2` (`57fb7a15`). The two commits are
replayed as-is; everything below is conflict resolution and adaptation, with
Rafael's crypto logic preserved verbatim except where the base had moved
under it. **Every non-mechanical decision is listed here** — the ones marked
AUTHOR REVIEW need Rafael's eyes (and ideally his re-authoring) before merge.

Base point: `92e2c452` (specs/connect: the encryption negotiation). Trunk had
moved 275 commits; 23 of the branch's 47 files overlapped trunk churn.

## AUTHOR REVIEW — semantic resolutions

### 1. The negotiation no longer fits a bit — it rides in the minor revision

The branch's premise was one spare bit in the CALL2 extension byte
(`ARQ_CONNECT_EXT_CRYPTO 0x10`). Since Oct 20, trunk's TX-MFSK-only work took
exactly that bit (`ARQ_CONNECT_EXT_TXMFSK 0x10`) and bumped the minor
revision to 1 — and the framer's extension field is five bits
(`FRAME_EXT_MASK 0x1f`): bandwidth (2) + minor (2) + TX-MFSK (1). **There is
no free bit.** The rebased branch's original encoding would have been
silently masked off: `connect_crypto()` always false, the feature dead but
fail-closed (caught by `test_connect_crypto_bit`).

Resolution (mechanical, but a wire-format decision — **re-author as you see
fit**): the negotiation rides in the minor field's fourth value.

- `ARQ_CONNECT_MINOR` stays 1 (trunk's, TX-MFSK's).
- `ARQ_CONNECT_MINOR_CRYPTO = 2` (arq_protocol.h): a **CALL sent at minor 2
  offers**, an **ACCEPT at minor 2 takes**.
- `arq_protocol_connect_crypto()` reads `connect_minor(buf) == 2`.
- `connect_ext()` writes minor 2 when crypto is offered/taken.
- A minor-0/1 station accepts the frame (version_ok takes any minor), reads
  the bandwidth, and never answers with minor 2: the session goes clear —
  optional mode falls back, required mode refuses (fails closed, like the
  branch's design).

Consequences checked: 1.9.x drops CALL2 entirely (unchanged); the TLA+ model
(`ConnectHandshake.tla`) models the offer/answer as booleans above the wire
position, so it survives; the prologue binding (offer_bit/accept_bit) is
unchanged. Docs updated: arq_crypto.h's WIRE FORMAT comment,
arq_protocol.h's byte-0 comment, CAROUSEL-ARQ.md's wire table row.

### 2. `bcast_file.c` — sealing moved into trunk's refactored `tx_open_bundle`

Trunk split bundle-building out of `bcast_file_tx_open` into
`tx_open_bundle(bundle, len, ...)` and added `bcast_file_tx_open_mem`. The
branch's sealing step (written into the old monolith between
`bcast_bundle_build` and `calloc`) is preserved **verbatim**, placed at the
top of `tx_open_bundle`, so both open paths (file and mem) seal. The
receive-side unseal and the `s_bkey` statics merged cleanly.

### 3. `tests/Makefile` — trunk's new broadcast tests link `bcast_aead.c`

`test_bcast_carousel` and `test_bcast_rx_host` (trunk additions) link
`bcast_file.c`, which now references `bcast_aead_available()` etc. Added
`../datalink_broadcast/bcast_aead.c` to both rules — safe without libsodium
(the file compiles to stubs, same as the branch's own `test_bcast_file`
rule).

### 4. `tests/datalink_arq/test_arq_protocol.c` — merged signatures, minor-2 asserts

Trunk's `tx_mfsk` parameter and the branch's `crypto` parameter both added to
`arq_protocol_build_call`/`build_accept` (order: `..., tx_mfsk, crypto`);
every call site in the tests updated. `test_connect_crypto_bit` now asserts
minor == `ARQ_CONNECT_MINOR_CRYPTO` when on, `ARQ_CONNECT_MINOR` when off,
and that `parse_call`'s TX-MFSK output stays false with the offer set. The
`RUN_TEST` list keeps both trunk's `test_connect_dst_matches_short_buf_refused`
and the branch's `test_connect_crypto_bit`.

### 5. `datalink_arq/arq_fsm.c` — negotiate call moved per commit 2

Commit 2 (bd0228ef) moves `crypto_negotiate(ARQ_NEG_CALL_IN, ...)` inside
`!same_session` ("the answer to its offer, once for the session"). Trunk
meanwhile made `call_rx_mode` conditional (`same_session ? accept_carrier :
ev->mode`). Both kept: trunk's conditional `call_rx_mode`, commit 2's
negotiate placement.

### 6. `datalink_arq/arq.c` — branch's crypto session state + trunk's signature

`crypto_abort()` and `crypto_session_start()` added as the branch wrote them;
`cb_notify_connected` keeps trunk's 3-parameter signature (`we_called`), with
the branch's tail (crypto-off fast path, else `crypto_session_start`)
auto-merged by git and verified.

## Mechanical resolutions (both sides kept, no judgement involved)

- `mercury.ini.example` — trunk's `[hl2]` section + branch's `[crypto]` section.
- `mercury.1` — trunk's `channel:busy_reseed_on_hop` row + branch's four `crypto:` rows.
- `gui_interface/fyne-ui/engine/mercury_bridge.c` — trunk's `bcast_carousel.h`
  / `bcast_rx_host.h` (Windows-guarded) includes + branch's `bcast_aead.h`.
- `Makefile` — `BCAST_FILE_OBJS` gains `bcast_carousel.o`, `bcast_rx_host.o`
  (trunk) and `bcast_aead.o` (branch; `bcast_rx_host` stays out of the W64
  list per trunk's Windows choice); the fyne-ui rule keeps trunk's
  `FYNE_UI_TAGS` expression and the branch's `CRYPTO_LDFLAGS`.
- `common/cfg_utils.c` — trunk's `[hl2]` read/write + branch's `[crypto]`
  read/write (unknown mode still refused at read).
- `docs/CAROUSEL-ARQ.md` — wire row per item 1 above.

## Validation (Debian 13, gcc 14.2, libsodium 1.0.18, go 1.24.4)

| Pass | Build | `make test` | Integration |
|---|---|---|---|
| `ARQ_CRYPTO=1` (default, sodium present) | clean | 48 suites, 0 failures | `ok mercuryintegration 710.136s`, all pass |
| `ARQ_CRYPTO=0` | clean | 45 suites, 0 failures (the 3 crypto-only suites are conditionally excluded, as designed) | — |

Integration includes all three crypto scenarios against real daemons:
`TestMercuryARQEncrypted` (39.8 s), `TestMercuryARQRequiredRefusesAClearPeer`
(23.0 s), `TestMercuryARQOptionalFallsBackToClear` (12.6 s) — all PASS.

Not yet re-run from Rafael's own list: the TLA+ model check (TLC) on the
rebased tree, `car_explore` at scale, and the handshake-cost measurement.
