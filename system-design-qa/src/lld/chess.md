# Q: Design a Chess game (Low-Level Design).

**Answer:**

A classic OOD problem. The interviewer is testing: clean class decomposition, polymorphism for piece movement, correctness of the *rules* (check, pins, castling, en passant), and whether you can extend the design (undo, AI, online play). Do not try to code every rule in 45 minutes; build the skeleton, then deepen where asked.

### Clarifying Questions to Ask

- Two humans on one machine, or networked / vs AI? (Scope: start local 2-player, discuss online later.)
- Do we need *all* rules: castling, en passant, promotion, draw rules? (Say yes, implement incrementally.)
- Do we need a clock / time controls? Undo? Move history export (PGN)? Save/load (FEN)?
- Console UI or just the engine API? (Engine API: UI is a thin layer on top.)
- Standard 8x8 only, or variants (Chess960, 4-player)? (Standard, but keep `Board` size-agnostic where cheap.)

### Requirements

**Functional**
- 8x8 board, 32 pieces, two players alternate (White first).
- Each piece type moves by its own rules; sliding pieces cannot jump; only knights jump.
- Capture by moving onto an enemy piece.
- A move is illegal if it leaves your own king in check (covers pins and "king moves into attack").
- Special moves: castling (both sides), en passant, pawn promotion.
- Game states: `ACTIVE`, `CHECK`, `CHECKMATE`, `STALEMATE`, `DRAW`, plus `RESIGNED`/`TIMEOUT` for online.
- Draws: threefold repetition, 50-move rule, insufficient material, stalemate, agreement.
- Undo, move history, FEN import/export, PGN-ish move list.

**Non-functional**
- Move validation must be fast enough for AI search (millions of positions) - design for a faster representation later.
- Rules logic independent from UI and transport.
- Easy to add a new piece type or variant.

### Core Entities / Class Diagram

```
 ┌──────────┐ 1     1 ┌────────────┐ 1      * ┌───────────┐
 │  Game    │────────►│   Board    │─────────►│  Piece    │ (abstract)
 │ state,   │         │ Piece[8][8]│          │ color     │
 │ turn,    │         │ epTarget   │          │ moved     │
 │ history  │         └─────┬──────┘          │ strategy  │
 └──┬───┬───┘               │ uses            └─────┬─────┘
    │   │ 2          ┌──────▼───────┐    ┌──┬──┬──┬─┴┬──┬──┐
    │   └──►Player   │ MoveStrategy │    K  Q  R  B  N  P
    │ *              │  (interface) │◄── Sliding / Step / Pawn / King
    ▼                └──────────────┘
 ┌──────────┐ 
 │  Move    │  from, to, piece, captured, type, promotion,
 │ (Command)│  saved state (ep target, clock, moved flags)
 └──────────┘  apply(board) / undo(board)

 enums: Color, PieceType, MoveType, GameStatus
 value: Square(row, col)
```

Key choices:
- `Piece` is an abstract class; each subclass supplies its **MoveStrategy** (pseudo-legal generation). The *legality* filter (don't leave king in check) is **not** per piece - it is a game-level rule applied uniformly.
- `Move` is a **Command**: it knows how to `apply` and `undo` itself, so history = stack of commands.
- Attack detection uses reverse geometry (look outward from the square) - avoids infinite recursion between "generate moves" and "is square attacked" (castling needs both).

### Design Patterns Used and Why

| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `MoveStrategy` (Sliding, Step, Pawn, King) | Rook/Bishop/Queen share sliding logic with different direction sets; reuse without deep inheritance |
| **Polymorphism / Template** | `Piece` subclasses | Adding a new piece = one small class |
| **Command** | `Move.apply/undo` | Undo/redo, history, replay |
| **Memento (embedded)** | state saved in `Move` (ep target, clocks, moved flags) | Exact undo of irreversible state |
| **State** (light) | `GameStatus` enum recomputed after each move | A full State pattern is overkill; enum is enough |
| **Factory** | `Board.initial()` / `Piece.of(type,color)` | Centralized setup, FEN loading |
| **Observer** | `GameListener` | UI, clock, network get notified of moves/status |

### Implementation (Java)

```java
// ===== Basic types =====
enum Color {
    WHITE, BLACK;
    Color opposite() { return this == WHITE ? BLACK : WHITE; }
    int forward() { return this == WHITE ? 1 : -1; }      // row direction for pawns
    int homeRow() { return this == WHITE ? 0 : 7; }
}

enum PieceType {
    KING('K'), QUEEN('Q'), ROOK('R'), BISHOP('B'), KNIGHT('N'), PAWN('P');
    final char letter;
    PieceType(char c) { letter = c; }
}

enum MoveType { NORMAL, DOUBLE_PUSH, EN_PASSANT, CASTLE_KING, CASTLE_QUEEN, PROMOTION }

enum GameStatus { ACTIVE, CHECK, CHECKMATE, STALEMATE, DRAW, RESIGNED, TIMEOUT }

/** row 0 = rank 1, col 0 = file a. */
record Square(int row, int col) {
    boolean inBounds() { return row >= 0 && row < 8 && col >= 0 && col < 8; }
    Square plus(int dr, int dc) { return new Square(row + dr, col + dc); }
    static Square of(String s) { return new Square(s.charAt(1) - '1', s.charAt(0) - 'a'); }
    @Override public String toString() { return "" + (char) ('a' + col) + (row + 1); }
}
```

```java
// ===== Strategy: pseudo-legal move generation =====
interface MoveStrategy {
    /** Generates moves ignoring "own king left in check". */
    void generate(Board board, Square from, Piece piece, List<Move> out);
}

class SlidingStrategy implements MoveStrategy {
    static final int[][] ORTHO = {{1,0},{-1,0},{0,1},{0,-1}};
    static final int[][] DIAG  = {{1,1},{1,-1},{-1,1},{-1,-1}};
    private final int[][] dirs;
    SlidingStrategy(int[][]... sets) {
        List<int[]> all = new ArrayList<>();
        for (int[][] s : sets) all.addAll(Arrays.asList(s));
        dirs = all.toArray(new int[0][]);
    }
    public void generate(Board b, Square from, Piece p, List<Move> out) {
        for (int[] d : dirs) {
            Square s = from.plus(d[0], d[1]);
            while (s.inBounds()) {                         // path blocking handled here
                Piece t = b.at(s);
                if (t == null) out.add(Move.normal(from, s, p, null));
                else {
                    if (t.color != p.color) out.add(Move.normal(from, s, p, t));
                    break;                                  // blocked either way
                }
                s = s.plus(d[0], d[1]);
            }
        }
    }
}

class StepStrategy implements MoveStrategy {                // knight + king basic steps
    static final int[][] KNIGHT = {{2,1},{2,-1},{-2,1},{-2,-1},{1,2},{1,-2},{-1,2},{-1,-2}};
    static final int[][] KING = {{1,0},{-1,0},{0,1},{0,-1},{1,1},{1,-1},{-1,1},{-1,-1}};
    private final int[][] offsets;
    StepStrategy(int[][] offsets) { this.offsets = offsets; }
    public void generate(Board b, Square from, Piece p, List<Move> out) {
        for (int[] o : offsets) {
            Square s = from.plus(o[0], o[1]);
            if (!s.inBounds()) continue;
            Piece t = b.at(s);
            if (t == null || t.color != p.color) out.add(Move.normal(from, s, p, t));
        }
    }
}

class PawnStrategy implements MoveStrategy {
    public void generate(Board b, Square from, Piece p, List<Move> out) {
        int dir = p.color.forward();
        Square one = from.plus(dir, 0);
        if (one.inBounds() && b.at(one) == null) {
            addWithPromotion(from, one, p, null, out);
            Square two = from.plus(2 * dir, 0);
            boolean startRow = from.row() == (p.color == Color.WHITE ? 1 : 6);
            if (startRow && b.at(two) == null) out.add(new Move(from, two, p, null, MoveType.DOUBLE_PUSH, null));
        }
        for (int dc : new int[]{-1, 1}) {
            Square diag = from.plus(dir, dc);
            if (!diag.inBounds()) continue;
            Piece t = b.at(diag);
            if (t != null && t.color != p.color) addWithPromotion(from, diag, p, t, out);
            else if (t == null && diag.equals(b.enPassantTarget())) {
                Piece victim = b.at(new Square(from.row(), diag.col()));
                out.add(new Move(from, diag, p, victim, MoveType.EN_PASSANT, null));
            }
        }
    }
    private void addWithPromotion(Square f, Square t, Piece p, Piece cap, List<Move> out) {
        if (t.row() == p.color.opposite().homeRow()) {
            for (PieceType pt : new PieceType[]{PieceType.QUEEN, PieceType.ROOK, PieceType.BISHOP, PieceType.KNIGHT})
                out.add(new Move(f, t, p, cap, MoveType.PROMOTION, pt));
        } else out.add(Move.normal(f, t, p, cap));
    }
}

class KingStrategy implements MoveStrategy {
    private final StepStrategy steps = new StepStrategy(StepStrategy.KING);
    public void generate(Board b, Square from, Piece king, List<Move> out) {
        steps.generate(b, from, king, out);
        if (king.moved || b.isAttacked(from, king.color.opposite())) return;   // no castling out of check
        int r = from.row();
        tryCastle(b, king, from, new Square(r, 7), new int[]{5, 6}, new int[]{5, 6}, MoveType.CASTLE_KING, out);
        tryCastle(b, king, from, new Square(r, 0), new int[]{1, 2, 3}, new int[]{3, 2}, MoveType.CASTLE_QUEEN, out);
    }
    private void tryCastle(Board b, Piece king, Square from, Square rookSq, int[] empty, int[] safe,
                           MoveType type, List<Move> out) {
        Piece rook = b.at(rookSq);
        if (rook == null || rook.type() != PieceType.ROOK || rook.moved || rook.color != king.color) return;
        for (int c : empty) if (b.at(new Square(from.row(), c)) != null) return;      // path clear
        for (int c : safe) if (b.isAttacked(new Square(from.row(), c), king.color.opposite())) return;
        int destCol = type == MoveType.CASTLE_KING ? 6 : 2;
        out.add(new Move(from, new Square(from.row(), destCol), king, null, type, null));
    }
}
```

```java
// ===== Pieces =====
abstract class Piece {
    final Color color;
    boolean moved;
    private final MoveStrategy strategy;
    protected Piece(Color c, MoveStrategy s) { color = c; strategy = s; }
    abstract PieceType type();
    List<Move> pseudoLegalMoves(Board b, Square from) {
        List<Move> out = new ArrayList<>();
        strategy.generate(b, from, this, out);
        return out;
    }
    char symbol() { char c = type().letter; return color == Color.WHITE ? c : Character.toLowerCase(c); }

    static Piece of(PieceType t, Color c) {
        return switch (t) {
            case KING -> new King(c); case QUEEN -> new Queen(c); case ROOK -> new Rook(c);
            case BISHOP -> new Bishop(c); case KNIGHT -> new Knight(c); case PAWN -> new Pawn(c);
        };
    }
}
class King   extends Piece { King(Color c)   { super(c, new KingStrategy()); } PieceType type() { return PieceType.KING; } }
class Queen  extends Piece { Queen(Color c)  { super(c, new SlidingStrategy(SlidingStrategy.ORTHO, SlidingStrategy.DIAG)); } PieceType type() { return PieceType.QUEEN; } }
class Rook   extends Piece { Rook(Color c)   { super(c, new SlidingStrategy(SlidingStrategy.ORTHO)); } PieceType type() { return PieceType.ROOK; } }
class Bishop extends Piece { Bishop(Color c) { super(c, new SlidingStrategy(SlidingStrategy.DIAG)); } PieceType type() { return PieceType.BISHOP; } }
class Knight extends Piece { Knight(Color c) { super(c, new StepStrategy(StepStrategy.KNIGHT)); } PieceType type() { return PieceType.KNIGHT; } }
class Pawn   extends Piece { Pawn(Color c)   { super(c, new PawnStrategy()); } PieceType type() { return PieceType.PAWN; } }
```

```java
// ===== Move = Command with embedded memento =====
class Move {
    final Square from, to;
    final Piece piece, captured;
    final MoveType type;
    final PieceType promotion;
    // memento: state needed to undo
    boolean pieceWasMoved, rookWasMoved;
    Square prevEpTarget;
    int prevHalfmove;

    Move(Square f, Square t, Piece p, Piece cap, MoveType type, PieceType promo) {
        this.from = f; this.to = t; this.piece = p; this.captured = cap; this.type = type; this.promotion = promo;
    }
    static Move normal(Square f, Square t, Piece p, Piece cap) { return new Move(f, t, p, cap, MoveType.NORMAL, null); }

    void apply(Board b) {
        prevEpTarget = b.enPassantTarget(); prevHalfmove = b.halfmoveClock(); pieceWasMoved = piece.moved;
        b.setEnPassantTarget(null);
        switch (type) {
            case EN_PASSANT -> {
                b.set(new Square(from.row(), to.col()), null);        // victim is beside, not on 'to'
                b.move(from, to);
            }
            case CASTLE_KING, CASTLE_QUEEN -> {
                int rookFrom = type == MoveType.CASTLE_KING ? 7 : 0, rookTo = type == MoveType.CASTLE_KING ? 5 : 3;
                Square rf = new Square(from.row(), rookFrom), rt = new Square(from.row(), rookTo);
                rookWasMoved = b.at(rf).moved;
                b.move(from, to); b.move(rf, rt); b.at(rt).moved = true;
            }
            case PROMOTION -> { b.set(from, null); b.set(to, Piece.of(promotion, piece.color)); }
            case DOUBLE_PUSH -> { b.move(from, to); b.setEnPassantTarget(new Square((from.row() + to.row()) / 2, from.col())); }
            default -> b.move(from, to);
        }
        piece.moved = true;
        b.setHalfmoveClock(piece.type() == PieceType.PAWN || captured != null ? 0 : prevHalfmove + 1);
    }

    void undo(Board b) {
        switch (type) {
            case EN_PASSANT -> { b.move(to, from); b.set(new Square(from.row(), to.col()), captured); }
            case CASTLE_KING, CASTLE_QUEEN -> {
                int rookFrom = type == MoveType.CASTLE_KING ? 7 : 0, rookTo = type == MoveType.CASTLE_KING ? 5 : 3;
                Square rf = new Square(from.row(), rookFrom), rt = new Square(from.row(), rookTo);
                b.move(to, from); b.move(rt, rf); b.at(rf).moved = rookWasMoved;
            }
            case PROMOTION -> { b.set(from, piece); b.set(to, captured); }
            default -> { b.move(to, from); b.set(to, captured); }
        }
        piece.moved = pieceWasMoved;
        b.setEnPassantTarget(prevEpTarget);
        b.setHalfmoveClock(prevHalfmove);
    }

    String notation() {   // simplified long algebraic: e2e4, e7e8q
        return from.toString() + to + (promotion != null ? Character.toLowerCase(promotion.letter) : "");
    }
}
```

```java
// ===== Board =====
class Board {
    private final Piece[][] grid = new Piece[8][8];
    private Square epTarget;
    private int halfmove;

    static Board initial() {
        Board b = new Board();
        PieceType[] back = {PieceType.ROOK, PieceType.KNIGHT, PieceType.BISHOP, PieceType.QUEEN,
                            PieceType.KING, PieceType.BISHOP, PieceType.KNIGHT, PieceType.ROOK};
        for (Color c : Color.values()) {
            for (int col = 0; col < 8; col++) {
                b.grid[c.homeRow()][col] = Piece.of(back[col], c);
                b.grid[c == Color.WHITE ? 1 : 6][col] = Piece.of(PieceType.PAWN, c);
            }
        }
        return b;
    }
    Piece at(Square s) { return grid[s.row()][s.col()]; }
    void set(Square s, Piece p) { grid[s.row()][s.col()] = p; }
    void move(Square f, Square t) { set(t, at(f)); set(f, null); }
    Square enPassantTarget() { return epTarget; }
    void setEnPassantTarget(Square s) { epTarget = s; }
    int halfmoveClock() { return halfmove; }
    void setHalfmoveClock(int h) { halfmove = h; }

    Square findKing(Color c) {
        for (int r = 0; r < 8; r++) for (int k = 0; k < 8; k++) {
            Piece p = grid[r][k];
            if (p != null && p.color == c && p.type() == PieceType.KING) return new Square(r, k);
        }
        throw new IllegalStateException("no king");
    }

    /** Reverse geometry: look outward from sq for an attacker of colour 'by'. No recursion into move gen. */
    boolean isAttacked(Square sq, Color by) {
        for (int[] o : StepStrategy.KNIGHT) if (is(sq.plus(o[0], o[1]), by, PieceType.KNIGHT)) return true;
        for (int[] o : StepStrategy.KING)   if (is(sq.plus(o[0], o[1]), by, PieceType.KING)) return true;
        int pawnRow = -by.forward();                       // pawn attacking us sits "behind" in its own direction
        if (is(sq.plus(pawnRow, -1), by, PieceType.PAWN) || is(sq.plus(pawnRow, 1), by, PieceType.PAWN)) return true;
        return ray(sq, by, SlidingStrategy.ORTHO, PieceType.ROOK) || ray(sq, by, SlidingStrategy.DIAG, PieceType.BISHOP);
    }
    private boolean is(Square s, Color c, PieceType t) {
        if (!s.inBounds()) return false;
        Piece p = at(s);
        return p != null && p.color == c && p.type() == t;
    }
    private boolean ray(Square sq, Color by, int[][] dirs, PieceType slider) {
        for (int[] d : dirs) {
            Square s = sq.plus(d[0], d[1]);
            while (s.inBounds()) {
                Piece p = at(s);
                if (p != null) {
                    if (p.color == by && (p.type() == slider || p.type() == PieceType.QUEEN)) return true;
                    break;
                }
                s = s.plus(d[0], d[1]);
            }
        }
        return false;
    }

    String fen(Color turn, int fullmove) {
        StringBuilder sb = new StringBuilder();
        for (int r = 7; r >= 0; r--) {
            int empty = 0;
            for (int c = 0; c < 8; c++) {
                Piece p = grid[r][c];
                if (p == null) empty++;
                else { if (empty > 0) { sb.append(empty); empty = 0; } sb.append(p.symbol()); }
            }
            if (empty > 0) sb.append(empty);
            if (r > 0) sb.append('/');
        }
        sb.append(turn == Color.WHITE ? " w " : " b ").append(castlingRights());
        sb.append(' ').append(epTarget == null ? "-" : epTarget).append(' ').append(halfmove).append(' ').append(fullmove);
        return sb.toString();
    }
    private String castlingRights() {
        StringBuilder s = new StringBuilder();
        s.append(right(0, 7, 'K')).append(right(0, 0, 'Q')).append(right(7, 7, 'k')).append(right(7, 0, 'q'));
        return s.length() == 0 ? "-" : s.toString();
    }
    private String right(int row, int rookCol, char ch) {
        Piece k = grid[row][4], r = grid[row][rookCol];
        boolean ok = k != null && k.type() == PieceType.KING && !k.moved && r != null && r.type() == PieceType.ROOK && !r.moved;
        return ok ? String.valueOf(ch) : "";
    }
    List<Piece> pieces() {
        List<Piece> l = new ArrayList<>();
        for (Piece[] row : grid) for (Piece p : row) if (p != null) l.add(p);
        return l;
    }
    Iterable<Square> squares() {
        List<Square> l = new ArrayList<>();
        for (int r = 0; r < 8; r++) for (int c = 0; c < 8; c++) l.add(new Square(r, c));
        return l;
    }
}
```

```java
// ===== Game: turn management, legality, status, history =====
class Player {
    final String name; final Color color;
    Player(String n, Color c) { name = n; color = c; }
}

interface GameListener { void onMove(Move m, GameStatus status); }

class Game {
    private final Board board = Board.initial();
    private final Player white, black;
    private Color turn = Color.WHITE;
    private GameStatus status = GameStatus.ACTIVE;
    private int fullmove = 1;
    private final Deque<Move> history = new ArrayDeque<>();
    private final Deque<String> fenHistory = new ArrayDeque<>();
    private final Map<String, Integer> repetition = new HashMap<>();
    private final List<GameListener> listeners = new ArrayList<>();

    Game(Player w, Player b) { white = w; black = b; recordPosition(); }

    /** All legal moves for the side to move: pseudo-legal filtered by "own king not attacked". */
    List<Move> legalMoves(Color side) {
        List<Move> legal = new ArrayList<>();
        for (Square sq : board.squares()) {
            Piece p = board.at(sq);
            if (p == null || p.color != side) continue;
            for (Move m : p.pseudoLegalMoves(board, sq)) {
                m.apply(board);                                           // try it
                boolean ownKingSafe = !board.isAttacked(board.findKing(side), side.opposite());
                m.undo(board);                                            // always undo
                if (ownKingSafe) legal.add(m);   // handles pins, king stepping into check, en-passant discovered checks
            }
        }
        return legal;
    }

    synchronized Move makeMove(String from, String to, PieceType promo) {
        if (status != GameStatus.ACTIVE && status != GameStatus.CHECK)
            throw new IllegalStateException("game over: " + status);
        Square f = Square.of(from), t = Square.of(to);
        Move chosen = null;
        for (Move m : legalMoves(turn)) {
            boolean promoOk = m.type != MoveType.PROMOTION || m.promotion == (promo == null ? PieceType.QUEEN : promo);
            if (m.from.equals(f) && m.to.equals(t) && promoOk) { chosen = m; break; }
        }
        if (chosen == null) throw new IllegalArgumentException("illegal move " + from + to);
        chosen.apply(board);
        history.push(chosen);
        if (turn == Color.BLACK) fullmove++;
        turn = turn.opposite();
        recordPosition();
        status = evaluate();
        listeners.forEach(l -> l.onMove(chosen, status));
        return chosen;
    }

    synchronized void undo() {
        if (history.isEmpty()) return;
        String key = positionKey();
        repetition.merge(key, -1, Integer::sum);
        Move m = history.pop();
        turn = turn.opposite();
        if (turn == Color.BLACK) fullmove--;
        m.undo(board);
        status = evaluate();
    }

    private GameStatus evaluate() {
        boolean inCheck = board.isAttacked(board.findKing(turn), turn.opposite());
        boolean hasMove = !legalMoves(turn).isEmpty();
        if (!hasMove) return inCheck ? GameStatus.CHECKMATE : GameStatus.STALEMATE;
        if (insufficientMaterial() || board.halfmoveClock() >= 100 || repetition.get(positionKey()) >= 3)
            return GameStatus.DRAW;
        return inCheck ? GameStatus.CHECK : GameStatus.ACTIVE;
    }

    private String positionKey() {                         // FEN minus the clocks
        String[] parts = board.fen(turn, fullmove).split(" ");
        return String.join(" ", parts[0], parts[1], parts[2], parts[3]);
    }
    private void recordPosition() { repetition.merge(positionKey(), 1, Integer::sum); }

    private boolean insufficientMaterial() {
        int minors = 0;
        for (Piece p : board.pieces()) {
            switch (p.type()) {
                case KING -> {}
                case BISHOP, KNIGHT -> minors++;
                default -> { return false; }               // any pawn/rook/queen = sufficient
            }
        }
        return minors <= 1;                                // K v K, K+minor v K (K+B v K+B same colour: omitted)
    }

    void addListener(GameListener l) { listeners.add(l); }
    GameStatus status() { return status; }
    Color turn() { return turn; }
    String fen() { return board.fen(turn, fullmove); }
    String pgn() {
        StringBuilder sb = new StringBuilder();
        int i = 0;
        List<Move> ordered = new ArrayList<>(history);
        Collections.reverse(ordered);                      // ArrayDeque stack iterates newest-first
        for (Move m : ordered) {
            if (i % 2 == 0) sb.append(i / 2 + 1).append(". ");
            sb.append(m.notation()).append(' ');
            i++;
        }
        return sb.toString().trim();
    }
}
```

```java
// ===== Demo: Fool's mate =====
public class Main {
    public static void main(String[] args) {
        Game g = new Game(new Player("Alice", Color.WHITE), new Player("Bob", Color.BLACK));
        g.addListener((m, s) -> System.out.println(m.notation() + " -> " + s));
        g.makeMove("f2", "f3", null);
        g.makeMove("e7", "e5", null);
        g.makeMove("g2", "g4", null);
        g.makeMove("d8", "h4", null);                      // checkmate
        System.out.println(g.status());                    // CHECKMATE
        System.out.println(g.fen());
        System.out.println(g.pgn());
        g.undo();
        System.out.println(g.status() + " " + g.turn());   // ACTIVE BLACK
    }
}
```

> [!NOTE]
> The `Piece` constructors take a fresh `MoveStrategy` per instance for clarity; strategies are stateless so make them `static final` singletons (Flyweight) in production to save allocations.

### How Each Special Rule Falls Out

| Rule | Where handled | Detail |
|---|---|---|
| Path blocking | `SlidingStrategy` | Stop the ray at first occupied square; include it only if enemy |
| Check | `Board.isAttacked(king, enemy)` | Evaluated after `evaluate()` for side to move |
| Pins | `legalMoves` filter | Simulate move, king attacked -> reject. No explicit pin logic needed |
| Castling | `KingStrategy` | King and rook unmoved, empty between, king not in/through/into check |
| En passant | `PawnStrategy` + `Move.apply` | `epTarget` set only after a double push, cleared on any next move; victim removed from beside `to` |
| Promotion | `PawnStrategy` generates 4 moves | UI picks the piece; default queen |
| Checkmate / stalemate | `evaluate()` | No legal move: in check -> mate, else stalemate |
| 3-fold repetition | position key (FEN minus clocks) counted in a map | Key includes side, castling rights, ep square |
| 50-move | `halfmoveClock >= 100` | Reset on pawn move/capture |
| Insufficient material | piece census | K v K, K+B v K, K+N v K (and same-colour bishops) |

> [!NOTE]
> The ep square in a repetition key should only count if a capture is actually legal (strict FIDE). Using the raw FEN field is a common, acceptable simplification.

### Concurrency Considerations

- Single game, two clients: serialize with `synchronized` on `Game` (or a per-game single-threaded executor / actor). Move validation is microseconds, no need for finer locks.
- `legalMoves` mutates the board temporarily (apply/undo). It is therefore **not** safe to call concurrently with itself - another reason to confine each `Game` to one thread. Alternatives: copy-on-make (immutable boards) for lock-free read-only analysis.
- AI search must run on a *copy* of the board (or a separate thread-confined instance), never on the live game.
- Clock: a scheduled executor per game firing a `TIMEOUT` task; cancel/reschedule after each move, guarded by the same game lock.
- Listeners must be invoked off the lock (or via a queue) so a slow websocket does not stall moves.

### Extensibility

- **New piece (e.g., Archbishop)**: subclass `Piece`, compose existing strategies (sliding diag + knight steps). No other change.
- **Chess960**: castling uses initial rook files; store them on the board.
- **Undo/redo**: a second stack for redo; clear on new move.
- **AI hook**: `interface MoveSelector { Move choose(Game g); }` - `HumanSelector`, `RandomSelector`, `MinimaxSelector` (Strategy again). `Player` owns a selector.
- **Time controls**: `Clock` class (increment, Fischer, sudden death) as a Game collaborator.
- **Persistence**: Game = initial FEN + list of moves. Replay to restore.

```java
interface MoveSelector { Move choose(Game game); }

class MinimaxSelector implements MoveSelector {
    private final int depth;
    MinimaxSelector(int depth) { this.depth = depth; }
    public Move choose(Game g) { /* iterate legalMoves, apply, alphaBeta(depth-1, -INF, INF, false), undo, keep best */ return null; }
    // alphaBeta: if depth==0 return evaluate(material + piece-square tables)
    // order captures first (MVV-LVA) to maximise cutoffs; add transposition table (Zobrist hash)
}
```

### Performance: Bitboards

The object-based board is ideal for an interview and readable code. Engines (Stockfish) use **bitboards**: one `long` per (piece type, colour); bit i = square i.

- Occupancy = OR of all boards; attack masks via precomputed tables (knight, king, pawn) and **magic bitboards** for sliders.
- "Is square attacked" becomes a few AND operations instead of loops.
- Make/undo = XOR a few bits; Zobrist hash updated incrementally (XOR key).
- Gives 10-100x speedup, which matters for minimax depth 10+.

```java
long knightAttacks(int sq) { return KNIGHT_TABLE[sq]; }          // precomputed
boolean knightChecks = (knightAttacks(kingSq) & enemyKnights) != 0;
```

### Scaling to Distributed / Online Multiplayer (HLD)

```
 Client ◄──WebSocket──► Gateway ──► Game Server (stateful, sticky by gameId)
                                      │   in-memory Game + clock
                                      ├──► Redis (live game snapshot, presence)
                                      ├──► Kafka (move events) ──► Analytics, spectators, anti-cheat
                                      └──► PostgreSQL (finished games: PGN, result, ratings)
 Matchmaker: queue by (time control, Elo band) ──► pairs players ──► allocates a Game Server
```

- **Transport**: WebSocket (bidirectional, low latency). Server is authoritative: client sends `{gameId, from, to}`; server validates with the same engine and broadcasts.
- **Stickiness**: consistent-hash `gameId -> game server`; a game's state lives in one process (single writer, no distributed locks). Snapshot FEN + moves to Redis each move for failover; on crash another node rebuilds from Redis.
- **Clock**: server-side authority. Store `remainingMs` and `lastMoveTs`; lag compensation = subtract measured RTT/2. Never trust client clocks.
- **Matchmaking**: sorted sets per time control keyed by Elo; widen the window over wait time; pair, create game, notify both. **Elo**: `R' = R + K*(S - E)`, `E = 1/(1+10^((Rb-Ra)/400))`; Glicko-2 adds rating deviation.
- **Reconnect**: client sends last move number; server replays missed moves.
- **Spectators**: fan-out via pub/sub topic per game.
- **Idempotency**: each move carries a move number; duplicates/out-of-order are rejected.
- **Anti-cheat**: engine-correlation analysis offline on the Kafka stream.

### Common Mistakes

| Mistake | Why wrong | Fix |
|---|---|---|
| One giant `if/else` on piece type in `Board` | Violates Open/Closed; untestable | `Piece` polymorphism + strategies |
| Validating "king in check" per piece class | Duplicates logic; pins missed | Single legality filter at game level |
| Generating attacks via move generation for castling | Infinite recursion; pawns attack diagonally but don't *move* there | Reverse-geometry `isAttacked` |
| Forgetting `moved` flags restore on undo | Castling rights wrongly lost/regained | Memento in `Move` |
| En passant: removing captured pawn from `to` | Victim is on `from.row`, `to.col` | Remove from beside |
| En passant target not cleared | Capture allowed a move late | Clear on every move |
| Castling through attacked square | Only checking final square | Check start, pass-through, and destination |
| Counting repetition without side-to-move/castling | False draws | Use FEN-minus-clocks key |
| Treating pawn promotion as separate turn | Breaks undo and notation | Promotion is part of the Move |
| Coupling UI to engine | Cannot test or add network | Listener interface, engine pure |
| Mutating shared board in AI thread | Race conditions, corrupt game | Search on a copy |

### Testing Strategy

- **Perft** (performance test): count leaf nodes at depth N from known positions (start: 20, 400, 8902, 197281, 4865609). Any rule bug changes the number. The gold standard for move generators.
- Unit tests: fool's mate, scholar's mate, stalemate position, castling blocked/through check, en passant discovered check, pinned piece cannot move, promotion with capture, 3-fold repetition by knight shuffling.
- FEN round-trip: `load(fen).fen() == fen`.

### Interview Follow-ups

- *"Why is Piece abstract and not an interface?"* — It holds shared state (`color`, `moved`) and shared behavior (`symbol`, generation delegation). An interface plus an abstract base also works; the point is that move logic lives with the piece type, not in `Board`.
- *"Strategy vs subclassing - isn't it redundant?"* — Subclasses name the piece; strategies share movement algorithms (Rook/Bishop/Queen share one sliding implementation parameterized by directions). It avoids Queen extends Rook or duplicate loops.
- *"How do you detect check efficiently?"* — Reverse geometry from the king: knights, pawns, king offsets, and rays. O(1)-ish, about 8+8+8 lookups. Bitboards make it a handful of ANDs.
- *"How do you handle pinned pieces?"* — Don't special-case: simulate the move and see if own king is attacked. Optimization for engines: compute pin masks once per position.
- *"How do you detect checkmate vs stalemate?"* — No legal moves for side to move; check present -> checkmate, else stalemate. Needs full legal move generation, so keep it fast.
- *"Why generate pseudo-legal then filter?"* — Simpler and less error-prone than generating legal moves directly; filter cost is acceptable. Engines do it too (with pin/check masks to avoid make/unmake when possible).
- *"How do you implement castling correctly?"* — King and chosen rook never moved; squares between empty; king not currently in check, not passing through or landing on attacked squares. Track `moved` flags (or castling-rights bitmask).
- *"Explain en passant."* — After an enemy pawn double-push, the square it skipped is the target for exactly one reply. The capturing pawn lands on the target but the victim is removed from its own square beside. Reset the target after each move.
- *"How would undo work?"* — Command pattern: every `Move` stores what it needs (captured piece, prior ep target, halfmove clock, moved flags) and implements `undo`. History is a stack.
- *"How do you do threefold repetition?"* — Hash position (placement + side + castling + ep) into a map count; draw when count hits 3. Use Zobrist hash in engines. Undo must decrement.
- *"What is the 50-move rule and how is it tracked?"* — Halfmove clock counts plies since last pawn move or capture; at 100 plies a player may claim a draw (auto-draw at 150 under FIDE 75-move rule).
- *"Insufficient material cases?"* — K v K, K+B v K, K+N v K, K+B v K+B with same-colour bishops. K+N+N v K is not forced mate but is technically not "dead" under FIDE.
- *"How do you represent moves for PGN?"* — Standard Algebraic Notation needs disambiguation (Nbd7), capture `x`, check `+`, mate `#`, castling `O-O`, promotion `=Q`. Generate by comparing against other legal moves of the same piece type to the same square.
- *"What's FEN?"* — One-line position: placement, side, castling, ep square, halfmove, fullmove. Used for save/load, testing, repetition keys.
- *"How would you add an AI?"* — `MoveSelector` strategy. Minimax with alpha-beta, iterative deepening, move ordering (captures, killer moves), transposition table, quiescence search, evaluation = material + piece-square tables + mobility + king safety.
- *"What's alpha-beta's complexity gain?"* — Best case O(b^(d/2)) vs O(b^d) for plain minimax, if moves are well ordered.
- *"Why bitboards?"* — Whole board ops in single CPU instructions, cache-friendly, incremental updates. Costs readability.
- *"How do you make the engine thread-safe?"* — Confine each Game to one thread/actor. Share only immutable data (position snapshots, FEN). AI works on copies.
- *"Design the clock."* — `Clock` with remaining time per side, increment, `start(side)`, `switchTo(side)`, timeout callback via `ScheduledExecutorService`. Server authoritative. Pause on game end.
- *"How would online play work?"* — WebSocket to a sticky game server holding the authoritative state; validate every move server-side; broadcast; persist PGN at the end; Redis snapshots for failover.
- *"How to match players?"* — Per-time-control pools sorted by rating; expand acceptable Elo difference with wait time; pair and create game; optionally colour balance based on history.
- *"How do you prevent cheating by clients?"* — Server validates all moves; clocks server-side; statistical engine-correlation detection offline; rate limit.
- *"How do you handle disconnect/abandon?"* — Grace timer (e.g., 30-60s) then forfeit; reconnect restores from Redis; if both disconnected, abort without rating change.
- *"How do you test the move generator?"* — Perft against published node counts (Kiwipete, positions 3-6). Catches castling/ep/promotion bugs.
- *"How do you support variants (Chess960, Crazyhouse)?"* — Rules injected via a `Variant` interface (setup, castling, drops); core `Piece`/`Move` stay.
- *"How do you support different board sizes?"* — Parametrize dimensions; avoid hard-coded 8s; bitboards need 128-bit or arrays for >64 squares.
- *"How to store millions of games?"* — Game row (ids, result, PGN compressed, ratings) in PostgreSQL; move list as compact bytes (about 1-2 bytes/move); archive to object storage; index by player and date.
- *"How would you let players resign / offer draw?"* — Additional commands on `Game`: `resign(color)`, `offerDraw`, `acceptDraw`; set `RESIGNED` / `DRAW`; same listener notifications.
- *"Serialization of the game for replays?"* — Initial FEN + move list (long algebraic) is complete and tiny; replay through the engine to rebuild.
- *"Where would you apply Observer?"* — UI refresh, clock, spectators, move logger, WebSocket broadcaster all subscribe to `GameListener`.
- *"Single Responsibility check?"* — `Board` holds pieces and attack queries; `Piece` generates; `Move` mutates; `Game` enforces rules and turn; `Player`/`MoveSelector` decide. UI and network are outside.
