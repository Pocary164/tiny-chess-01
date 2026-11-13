# tiny-chess-01 — a 4 KB chess engine

An original chess engine in a single self-extracting `main.py` (4,065 bytes as
committed; the readable engine source is lzma-compressed and base85-encoded
inside it, a packing the rules explicitly allow). Pure Python 3 standard
library, fully deterministic: it searches a fixed number of nodes per move
(80,000) and never consults a clock or an unseeded random source, so any game
replays move-for-move.

## How it plays

- **Board**: 0x88 mailbox with piece lists and an index array; make/unmake by
  full state snapshots (copy-make), Zobrist hashing (seeded PRNG tables built
  at start-up), incremental castling/en-passant/side-to-move hash terms.
- **Move generation**: complete rules — castling legality (rights, empty and
  unattacked transit squares), en passant (including the pinned-ep case),
  promotions, check evasion. Validated by exact perft counts on the standard
  test suite (startpos, Kiwipete, the CPW positions, a castling-rights
  position) and by randomised make/unmake state-integrity fuzzing.
- **Search**: negamax alpha-beta with iterative deepening under the fixed node
  budget, a transposition table (depth / score / best move, mate scores
  stored ply-adjusted), and quiescence search over captures and promotions
  (all evasions when in check). Move ordering: transposition-table move
  first, then captures by MVV-LVA with a promotion bonus.
- **Evaluation**: tapered piece-square tables (formula-generated at start-up:
  pawn structure by rank and file, centralisation for knights, bishops and
  the queen, rook on the seventh rank, a sheltered-king midgame table and an
  active-king endgame table) blended by game phase, with material values
  P100 N320 B330 R500 Q1000 (midgame) / P110 N300 B320 R530 Q950 (endgame).
- **Protocol**: the program talks to the competition environment over HTTP
  (POST /game, /move) exactly as specified, rebuilds the position from the
  served move list each turn, and validates its chosen move against the
  environment's legal-move list before sending.

## Validation

- Perft (legal-move tree counts) matches the canonical published values
  exactly on the standard suite: startpos (20 / 400 / 8,902), Kiwipete
  (48 / 2,039 / 97,862), the CPW test positions, a castling-rights position
  (26 / 568) and an en-passant-pin position (14 / 191 / 2,812 / 43,238).
- Randomised make/unmake fuzzing over full games: after every move, undoing
  restores the exact prior state (board, piece lists, indices, hash,
  evaluation) and redoing reproduces the exact posterior state.
- Determinism: the same position searched twice from a fresh start returns
  the identical move and node count, and a full self-play game replays
  byte-identically in a separate process.
- The engine plays complete games against the kit's real environment server
  with every move accepted.

## Deliverables

- `main.py` — the engine (the only code file; it reads no other repo file).
- `LICENSE` — MIT.
- This README.
