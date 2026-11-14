# tiny-chess-01 — 4 KB chess engine (v2)

An original chess engine in a single self-extracting `main.py` (4,095 bytes
as committed; the readable engine source is lzma-compressed and
base85-encoded inside it, a packing the rules explicitly allow). Pure Python
3 standard library, fully deterministic: it searches a fixed number of nodes
per move (95,000) and never consults a clock, so any game replays exactly.

## How it plays

- **Board**: 0x88 mailbox with piece lists and an index array. Make pushes a
  full state snapshot; unmake restores the slices in place, while the
  evaluation terms, game phase, non-pawn material counts and the Zobrist
  hash are maintained incrementally inside make, so no full rescan happens.
- **Move generation**: complete rules — castling legality (rights, empty and
  unattacked transit squares), en passant (including the pinned-ep case),
  promotions, check evasion.
- **Search**: negamax alpha-beta with iterative deepening under the fixed
  node budget, a transposition table (depth / score / best move), null-move
  pruning (R = 2, only with non-pawn material and not in check),
  quiescence search over captures and promotions (all evasions when in
  check) with delta pruning, and a one-ply extension for checking moves.
  Ordering: TT move first, then captures by MVV-LVA plus promotion bonus.
- **Evaluation**: tapered piece-square tables (formula-generated at
  start-up: pawn structure by rank and file, centralisation for knights,
  bishops and the queen, rook on the seventh rank, a sheltered-king midgame
  table and an active-king endgame table) blended by game phase, with
  material values P100 N320 B330 R500 Q1000 (midgame) / P110 N300 B320
  R530 Q950 (endgame). Incremental scores are fuzz-checked against an
  independent board scan.
- **Protocol**: talks to the competition environment over HTTP (POST
  /game, /move) exactly as specified, rebuilds the position from the served
  move list each turn, and validates its move against the environment's
  legal-move list before sending.

## Validation

- Perft matches the canonical values on the standard suite: startpos
  (20 / 400 / 8,902), Kiwipete (48 / 2,039 / 97,862), the CPW positions, a
  castling position (26 / 568), an en-passant-pin position
  (14 / 191 / 2,812 / 43,238).
- Randomised make/unmake fuzzing: undoing restores the exact prior state
  (board, piece lists, indices, hash, evaluation); redoing reproduces it.
- Determinism: the same position searched twice returns the identical move
  and node count; a full self-play game replays byte-identically in a
  separate process.
- Strength (this machine, kit practice openings, colours swapped): 100
  games against the previous version of this engine (80,000 nodes) scored
  70.5% (54W 33D 13L, +150 Elo); 100 games against the kit's house
  baseline replica scored 72.5% (60W 25D 15L).

## Deliverables

- `main.py` — the engine (only code file; reads no other repo file).
- `LICENSE` — MIT. This README.
