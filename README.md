# tiny-chess-01 — v6 entry

A chess engine in 4,073 bytes of Python (LZMA + base85 packed; the
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
  negamax alpha-beta, check extensions, and **late move reductions**
  (quiet, non-pawn, late moves reduced by 1–2 plies with a full-depth
  re-search on fail-high).
- **Move ordering**: TT move first, then MVV-LVA captures, then two
  killer moves per ply, then history heuristic (d² bonus on quiet
  cutoffs).
- **Quiescence**: stand-pat with delta pruning (margin 280), captures
  and promotions only, full move list when in check.

## Tuning notes

Designs were screened with paired, colour-swapped fixed-node matches
and confirmed on the practice gauntlet. Late move reductions plus
killer/history ordering were the gains that survived screening;
futility pruning, PVS, bishop-pair and passed-pawn terms all measured
neutral-to-negative on this stack and were left out. Full evidence:
kept with the campaign logs outside this repo.

## Design

In my own summary: the whole engine ships as one self-extracting
Python file — a compressed payload that unpacks into a sunfish-style
board with fast make/unmake. Its search is a deterministic,
fixed-node alpha-beta (negamax) search, so the same position always
produces the same move and no clock is ever consulted. Two ideas do
most of the heavy lifting at this size: late move reductions, which
spend full depth only on the moves most likely to matter, and
killer/history move ordering, which tries quiet moves that caused
cutoffs before — together they let a very small search play well
beyond what its node budget suggests.
