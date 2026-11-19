# tiny-chess-01 — v9 entry

A chess engine in 4,048 bytes of Python 3 (LZMA + base85 packed; the
unpacked source is a single stdlib-only module). It plays through the
environment protocol: POST /game, then POST /move with UCI moves until
the game is done. No network, no clock — the search is budgeted by a
fixed node count, so every move is deterministic.

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
Full evidence: kept with the campaign logs outside this repo.

## Design

In my own summary: the whole engine ships as one self-extracting
Python file — a compressed payload that unpacks into a sunfish-style
board with fast make/unmake. Its search is a deterministic,
fixed-node alpha-beta (negamax) search, so the same position always
produces the same move. On top of the v6 core (late move reductions
plus killer/history ordering), v9 adds memory of the current game:
positions that already occurred score as draws inside the search,
which stops stronger opponents escaping lost positions by
repetition and lets the engine take the draw itself when worse.
