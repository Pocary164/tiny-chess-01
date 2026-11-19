# tiny-chess-01 — v9 entry

[![make.fun · tiny-chess-01](https://make.fun/badge/tiny-chess-01.svg)](https://make.fun/bounty/tiny-chess-01?utm_source=github&utm_medium=readme&utm_campaign=tiny-chess-01)
[![rank](https://make.fun/badge/tiny-chess-01/Pocary164/tiny-chess-01/rank.svg)](https://make.fun/bounty/tiny-chess-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=tiny-chess-01)
[![reward](https://make.fun/badge/tiny-chess-01/Pocary164/tiny-chess-01/reward.svg)](https://make.fun/bounty/tiny-chess-01?utm_source=github&utm_medium=readme&utm_campaign=tiny-chess-01)

A chess engine in 4,048 bytes of Python 3 (LZMA + base85 packed; the
unpacked source is a single stdlib-only module). It plays through the
environment protocol: POST /game, then POST /move with UCI moves until
the game is done. There is no network and no clock: the search is
budgeted by a fixed node count, so every move is deterministic.

## Design

- **Board**: 0x88 mailbox, signed piece codes, piece lists with an
  index array; make/unmake by append / swap-remove (exact state and
  hash restoration), incremental material + piece-square scores and
  incremental Zobrist hash (splitmix-generated keys).
- **Evaluation**: tapered MG/EG piece-square tables (own values;
  material folded in), phase = non-pawn material. The tables are
  left-right symmetric and stored at half width.
- **Search**: iterative deepening over a 95,000-node budget with a
  transposition table (depth / value / bound flag / best move),
  negamax alpha-beta with PVS, check extensions, and **late move
  reductions** (quiet, non-pawn, late moves reduced by 1–2 plies with
  a full-depth re-search on fail-high).
- **Move ordering**: TT move first, then MVV-LVA captures, then two
  killer moves per ply, then history heuristic (d² bonus on quiet
  cutoffs).
- **Quiescence**: stand-pat with delta pruning (margin 280), captures
  and promotions only, full move list when in check.
- **Repetition awareness (new in v9)**: the bot tracks the Zobrist
  hashes of every position it has faced in the current game (its own
  turns and the positions after its moves). Inside the search, a
  position already in that game history scores as a draw (0), so the
  engine steers toward repetition when worse and avoids it when
  better. Draws that save a worse position stay reachable — only the
  search's evaluation of repeated positions changes.

## Tuning notes

v9 is the v6 engine with two changes: a behaviour-identical byte
golf of the packed source (perft suite, state/hash-restoration fuzz
and probe identity — same moves and node counts — all verified
unchanged), and the repetition rule above. Measured on paired
practice-gauntlet rolls (same six instances for both engines, v6
re-rolled same day): v9 scored 1598 / 1598 / 1530 / 1598 / 1669 /
1845 (min 1530, mean 1640) against v6's 1396 / 1669 / 1530 / 1464 /
1530 / 1748 — a paired mean gain of +84 Elo.
