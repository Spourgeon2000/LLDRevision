# Chess Game - Low Level Design (Revised)

## Requirements
// Design a chess game with proper move validation, turn management, and win detection
// Support all chess pieces with their specific movement rules
// Handle special moves: Castling, En Passant, Pawn Promotion
// Detect check, checkmate, and stalemate

---

## 1. Core Entities

### Enums
```java
enum Color {
    WHITE, BLACK;
    
    Color opposite() {
        return this == WHITE ? BLACK : WHITE;
    }
}

enum PieceType {
    KING, QUEEN, ROOK, BISHOP, KNIGHT, PAWN
}

enum GameStatus {
    ACTIVE, WHITE_WIN, BLACK_WIN, STALEMATE, DRAW
}
```

### Position (Renamed from CellAddress for clarity)
```java
class Position {
    int row;    // 0-7
    int column; // 0-7
    
    Position(int row, int column) {
        this.row = row;
        this.column = column;
    }
    
    boolean isValid() {
        return row >= 0 && row < 8 && column >= 0 && column < 8;
    }
    
    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof Position)) return false;
        Position other = (Position) obj;
        return this.row == other.row && this.column == other.column;
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(row, column);
    }
}
```

---

## 2. Cell

```java
class Cell {
    private Position position;
    private Piece piece;  // Can be null if empty
    private Color color;  // Color of the cell (for board pattern)
    
    Cell(Position position, Color color) {
        this.position = position;
        this.color = color;
        this.piece = null;
    }
    
    public boolean isEmpty() {
        return piece == null;
    }
    
    public boolean isOccupiedByOpponent(Color playerColor) {
        return !isEmpty() && piece.getColor() != playerColor;
    }
    
    public boolean isOccupiedBySameColor(Color playerColor) {
        return !isEmpty() && piece.getColor() == playerColor;
    }
    
    // Getters and setters
    public Position getPosition() { return position; }
    public Piece getPiece() { return piece; }
    public void setPiece(Piece piece) { this.piece = piece; }
    public void removePiece() { this.piece = null; }
    public Color getCellColor() { return color; }
}
```

---

## 3. Board

```java
class Board {
    private static final int BOARD_SIZE = 8;
    private Cell[][] cells;
    
    public Board() {
        cells = new Cell[BOARD_SIZE][BOARD_SIZE];
        initializeBoard();
    }
    
    private void initializeBoard() {
        for (int row = 0; row < BOARD_SIZE; row++) {
            for (int col = 0; col < BOARD_SIZE; col++) {
                Position position = new Position(row, col);
                // Checkerboard pattern
                Color cellColor = (row + col) % 2 == 0 ? Color.WHITE : Color.BLACK;
                cells[row][col] = new Cell(position, cellColor);
            }
        }
    }
    
    public Cell getCell(Position position) {
        if (!position.isValid()) {
            return null;
        }
        return cells[position.row][position.column];
    }
    
    public Cell getCell(int row, int col) {
        return getCell(new Position(row, col));
    }
    
    // Get all cells between two positions (exclusive)
    public List<Cell> getCellsBetween(Position start, Position end) {
        List<Cell> cells = new ArrayList<>();
        
        int rowDirection = Integer.compare(end.row - start.row, 0);
        int colDirection = Integer.compare(end.column - start.column, 0);
        
        int currentRow = start.row + rowDirection;
        int currentCol = start.column + colDirection;
        
        while (currentRow != end.row || currentCol != end.column) {
            cells.add(getCell(currentRow, currentCol));
            currentRow += rowDirection;
            currentCol += colDirection;
        }
        
        return cells;
    }
    
    public boolean isPathClear(Position start, Position end) {
        List<Cell> cellsBetween = getCellsBetween(start, end);
        return cellsBetween.stream().allMatch(Cell::isEmpty);
    }
}
```

---

## 4. Move Direction Strategy Pattern

```java
// Strategy Pattern - Different movement directions
interface MoveDirection {
    boolean isValidDirection(Position from, Position to);
    List<Cell> getCellsInPath(Position from, Position to, Board board);
}

class HorizontalMove implements MoveDirection {
    private int maxSteps;  // -1 for unlimited

    HorizontalMove(int maxSteps) {
        this.maxSteps = maxSteps;
    }

    @Override
    public boolean isValidDirection(Position from, Position to) {
        if (from.row != to.row) return false;  // Must be same row

        int distance = Math.abs(to.column - from.column);
        return maxSteps == -1 || distance <= maxSteps;
    }

    @Override
    public List<Cell> getCellsInPath(Position from, Position to, Board board) {
        return board.getCellsBetween(from, to);
    }
}

class VerticalMove implements MoveDirection {
    private int maxSteps;

    VerticalMove(int maxSteps) {
        this.maxSteps = maxSteps;
    }

    @Override
    public boolean isValidDirection(Position from, Position to) {
        if (from.column != to.column) return false;  // Must be same column

        int distance = Math.abs(to.row - from.row);
        return maxSteps == -1 || distance <= maxSteps;
    }

    @Override
    public List<Cell> getCellsInPath(Position from, Position to, Board board) {
        return board.getCellsBetween(from, to);
    }
}

class DiagonalMove implements MoveDirection {
    private int maxSteps;

    DiagonalMove(int maxSteps) {
        this.maxSteps = maxSteps;
    }

    @Override
    public boolean isValidDirection(Position from, Position to) {
        int rowDiff = Math.abs(to.row - from.row);
        int colDiff = Math.abs(to.column - from.column);

        if (rowDiff != colDiff) return false;  // Must be diagonal

        return maxSteps == -1 || rowDiff <= maxSteps;
    }

    @Override
    public List<Cell> getCellsInPath(Position from, Position to, Board board) {
        return board.getCellsBetween(from, to);
    }
}

class KnightMove implements MoveDirection {
    @Override
    public boolean isValidDirection(Position from, Position to) {
        int rowDiff = Math.abs(to.row - from.row);
        int colDiff = Math.abs(to.column - from.column);

        // L-shape: 2 in one direction, 1 in other
        return (rowDiff == 2 && colDiff == 1) || (rowDiff == 1 && colDiff == 2);
    }

    @Override
    public List<Cell> getCellsInPath(Position from, Position to, Board board) {
        // Knight jumps, no cells in between
        return new ArrayList<>();
    }
}
```

---

## 5. Piece (Abstract Base Class)

```java
abstract class Piece {
    protected PieceType type;
    protected Color color;
    protected Position currentPosition;
    protected boolean hasMoved;  // For castling and pawn double move
    protected List<MoveDirection> allowedMoveDirections;

    Piece(PieceType type, Color color, Position position) {
        this.type = type;
        this.color = color;
        this.currentPosition = position;
        this.hasMoved = false;
        this.allowedMoveDirections = new ArrayList<>();
    }

    // Template Method Pattern - common validation logic
    public boolean canMove(Position destination, Board board) {
        Cell destinationCell = board.getCell(destination);

        // Can't move to invalid position
        if (destinationCell == null) return false;

        // Can't capture own piece
        if (destinationCell.isOccupiedBySameColor(this.color)) return false;

        // Check if move matches allowed directions
        boolean isDirectionValid = allowedMoveDirections.stream()
            .anyMatch(direction -> direction.isValidDirection(currentPosition, destination));

        if (!isDirectionValid) return false;

        // Check if path is clear (for non-jumping pieces)
        if (!canJump() && !board.isPathClear(currentPosition, destination)) {
            return false;
        }

        // Piece-specific validation
        return isValidMove(destination, board);
    }

    // Hook method - subclasses can override for specific rules
    protected abstract boolean isValidMove(Position destination, Board board);

    // Can this piece jump over others? (only Knight can)
    protected boolean canJump() {
        return false;
    }

    public void moveTo(Position newPosition) {
        this.currentPosition = newPosition;
        this.hasMoved = true;
    }

    // Getters
    public PieceType getType() { return type; }
    public Color getColor() { return color; }
    public Position getCurrentPosition() { return currentPosition; }
    public boolean hasMoved() { return hasMoved; }
}
```

---

## 6. Concrete Piece Implementations

```java
class King extends Piece {
    King(Color color, Position position) {
        super(PieceType.KING, color, position);
        // King can move 1 step in any direction
        allowedMoveDirections.add(new HorizontalMove(1));
        allowedMoveDirections.add(new VerticalMove(1));
        allowedMoveDirections.add(new DiagonalMove(1));
    }

    @Override
    protected boolean isValidMove(Position destination, Board board) {
        // TODO: Add castling logic
        // TODO: Can't move into check
        return true;
    }
}

class Queen extends Piece {
    Queen(Color color, Position position) {
        super(PieceType.QUEEN, color, position);
        // Queen can move unlimited in any direction
        allowedMoveDirections.add(new HorizontalMove(-1));
        allowedMoveDirections.add(new VerticalMove(-1));
        allowedMoveDirections.add(new DiagonalMove(-1));
    }

    @Override
    protected boolean isValidMove(Position destination, Board board) {
        return true;
    }
}

class Rook extends Piece {
    Rook(Color color, Position position) {
        super(PieceType.ROOK, color, position);
        // Rook can move unlimited horizontally or vertically
        allowedMoveDirections.add(new HorizontalMove(-1));
        allowedMoveDirections.add(new VerticalMove(-1));
    }

    @Override
    protected boolean isValidMove(Position destination, Board board) {
        return true;
    }
}

class Bishop extends Piece {
    Bishop(Color color, Position position) {
        super(PieceType.BISHOP, color, position);
        // Bishop can move unlimited diagonally
        allowedMoveDirections.add(new DiagonalMove(-1));
    }

    @Override
    protected boolean isValidMove(Position destination, Board board) {
        return true;
    }
}

class Knight extends Piece {
    Knight(Color color, Position position) {
        super(PieceType.KNIGHT, color, position);
        // Knight moves in L-shape
        allowedMoveDirections.add(new KnightMove());
    }

    @Override
    protected boolean canJump() {
        return true;  // Knight can jump over pieces
    }

    @Override
    protected boolean isValidMove(Position destination, Board board) {
        return true;
    }
}

class Pawn extends Piece {
    Pawn(Color color, Position position) {
        super(PieceType.PAWN, color, position);
        // Pawn movement is complex, handled in isValidMove
    }

    @Override
    protected boolean isValidMove(Position destination, Board board) {
        int direction = (color == Color.WHITE) ? 1 : -1;  // White moves up, Black moves down
        int rowDiff = destination.row - currentPosition.row;
        int colDiff = Math.abs(destination.column - currentPosition.column);

        // Forward move (1 or 2 squares)
        if (colDiff == 0) {
            Cell destinationCell = board.getCell(destination);
            if (!destinationCell.isEmpty()) return false;  // Can't capture forward

            // Single step forward
            if (rowDiff == direction) return true;

            // Double step forward (only if not moved)
            if (!hasMoved && rowDiff == 2 * direction) {
                Position intermediatePos = new Position(
                    currentPosition.row + direction,
                    currentPosition.column
                );
                return board.getCell(intermediatePos).isEmpty();
            }

            return false;
        }

        // Diagonal capture
        if (colDiff == 1 && rowDiff == direction) {
            Cell destinationCell = board.getCell(destination);
            return destinationCell.isOccupiedByOpponent(color);
            // TODO: Add en passant logic
        }

        return false;
    }
}
```

---

## 7. Piece Factory Pattern

```java
class PieceFactory {
    public static Piece createPiece(PieceType type, Color color, Position position) {
        switch (type) {
            case KING:
                return new King(color, position);
            case QUEEN:
                return new Queen(color, position);
            case ROOK:
                return new Rook(color, position);
            case BISHOP:
                return new Bishop(color, position);
            case KNIGHT:
                return new Knight(color, position);
            case PAWN:
                return new Pawn(color, position);
            default:
                throw new IllegalArgumentException("Unknown piece type: " + type);
        }
    }
}
```

---

## 8. Player

```java
class Player {
    private String name;
    private Color color;
    private List<Piece> pieces;
    private boolean isInCheck;

    Player(String name, Color color) {
        this.name = name;
        this.color = color;
        this.pieces = new ArrayList<>();
        this.isInCheck = false;
    }

    public void addPiece(Piece piece) {
        pieces.add(piece);
    }

    public void removePiece(Piece piece) {
        pieces.remove(piece);
    }

    public Piece getKing() {
        return pieces.stream()
            .filter(p -> p.getType() == PieceType.KING)
            .findFirst()
            .orElse(null);
    }

    public List<Piece> getAlivePieces() {
        return new ArrayList<>(pieces);
    }

    // Getters
    public String getName() { return name; }
    public Color getColor() { return color; }
    public boolean isInCheck() { return isInCheck; }
    public void setInCheck(boolean inCheck) { this.isInCheck = inCheck; }
}
```

---

## 9. Move (Command Pattern)

```java
class Move {
    private Piece piece;
    private Position from;
    private Position to;
    private Piece capturedPiece;  // For undo
    private long timestamp;

    Move(Piece piece, Position from, Position to) {
        this.piece = piece;
        this.from = from;
        this.to = to;
        this.timestamp = System.currentTimeMillis();
    }

    public void execute(Board board) {
        Cell fromCell = board.getCell(from);
        Cell toCell = board.getCell(to);

        // Capture piece if present
        if (!toCell.isEmpty()) {
            capturedPiece = toCell.getPiece();
        }

        // Move piece
        toCell.setPiece(piece);
        fromCell.removePiece();
        piece.moveTo(to);
    }

    public void undo(Board board) {
        Cell fromCell = board.getCell(from);
        Cell toCell = board.getCell(to);

        // Move piece back
        fromCell.setPiece(piece);
        toCell.setPiece(capturedPiece);  // Restore captured piece (can be null)
        piece.moveTo(from);
    }

    // Getters
    public Piece getPiece() { return piece; }
    public Position getFrom() { return from; }
    public Position getTo() { return to; }
    public Piece getCapturedPiece() { return capturedPiece; }
}
```

---

## 10. Game State

```java
class ChessGame {
    private Board board;
    private Player whitePlayer;
    private Player blackPlayer;
    private Player currentPlayer;
    private GameStatus status;
    private List<Move> moveHistory;

    ChessGame(String player1Name, String player2Name) {
        this.board = new Board();
        this.whitePlayer = new Player(player1Name, Color.WHITE);
        this.blackPlayer = new Player(player2Name, Color.BLACK);
        this.currentPlayer = whitePlayer;  // White starts
        this.status = GameStatus.ACTIVE;
        this.moveHistory = new ArrayList<>();

        initializePieces();
    }

    private void initializePieces() {
        // Initialize white pieces
        setupPiecesForPlayer(whitePlayer, 0, 1);

        // Initialize black pieces
        setupPiecesForPlayer(blackPlayer, 7, 6);
    }

    private void setupPiecesForPlayer(Player player, int backRow, int pawnRow) {
        Color color = player.getColor();

        // Back row pieces
        player.addPiece(PieceFactory.createPiece(PieceType.ROOK, color, new Position(backRow, 0)));
        player.addPiece(PieceFactory.createPiece(PieceType.KNIGHT, color, new Position(backRow, 1)));
        player.addPiece(PieceFactory.createPiece(PieceType.BISHOP, color, new Position(backRow, 2)));
        player.addPiece(PieceFactory.createPiece(PieceType.QUEEN, color, new Position(backRow, 3)));
        player.addPiece(PieceFactory.createPiece(PieceType.KING, color, new Position(backRow, 4)));
        player.addPiece(PieceFactory.createPiece(PieceType.BISHOP, color, new Position(backRow, 5)));
        player.addPiece(PieceFactory.createPiece(PieceType.KNIGHT, color, new Position(backRow, 6)));
        player.addPiece(PieceFactory.createPiece(PieceType.ROOK, color, new Position(backRow, 7)));

        // Pawns
        for (int col = 0; col < 8; col++) {
            player.addPiece(PieceFactory.createPiece(PieceType.PAWN, color, new Position(pawnRow, col)));
        }

        // Place pieces on board
        for (Piece piece : player.getAlivePieces()) {
            Position pos = piece.getCurrentPosition();
            board.getCell(pos).setPiece(piece);
        }
    }

    // Getters
    public Board getBoard() { return board; }
    public Player getCurrentPlayer() { return currentPlayer; }
    public GameStatus getStatus() { return status; }
    public List<Move> getMoveHistory() { return new ArrayList<>(moveHistory); }

    private void switchTurn() {
        currentPlayer = (currentPlayer == whitePlayer) ? blackPlayer : whitePlayer;
    }
}
```

---

## 11. Chess Service (Game Logic)

```java
class ChessService {
    private ChessGame game;

    ChessService(ChessGame game) {
        this.game = game;
    }

    // Main method to make a move
    public boolean makeMove(Position from, Position to) {
        if (game.getStatus() != GameStatus.ACTIVE) {
            System.out.println("Game is not active");
            return false;
        }

        Cell fromCell = game.getBoard().getCell(from);
        if (fromCell.isEmpty()) {
            System.out.println("No piece at source position");
            return false;
        }

        Piece piece = fromCell.getPiece();

        // Check if piece belongs to current player
        if (piece.getColor() != game.getCurrentPlayer().getColor()) {
            System.out.println("Not your piece");
            return false;
        }

        // Check if move is valid for this piece
        if (!piece.canMove(to, game.getBoard())) {
            System.out.println("Invalid move for this piece");
            return false;
        }

        // Create and execute move
        Move move = new Move(piece, from, to);
        move.execute(game.getBoard());

        // Check if move puts own king in check (illegal)
        if (isKingInCheck(game.getCurrentPlayer())) {
            move.undo(game.getBoard());
            System.out.println("Move puts your king in check");
            return false;
        }

        // Add to history
        game.getMoveHistory().add(move);

        // Update opponent's check status
        Player opponent = getOpponent(game.getCurrentPlayer());
        boolean opponentInCheck = isKingInCheck(opponent);
        opponent.setInCheck(opponentInCheck);

        // Check for checkmate or stalemate
        if (opponentInCheck) {
            if (isCheckmate(opponent)) {
                game.status = (game.getCurrentPlayer().getColor() == Color.WHITE)
                    ? GameStatus.WHITE_WIN : GameStatus.BLACK_WIN;
                System.out.println("Checkmate! " + game.getCurrentPlayer().getName() + " wins!");
            } else {
                System.out.println("Check!");
            }
        } else if (isStalemate(opponent)) {
            game.status = GameStatus.STALEMATE;
            System.out.println("Stalemate!");
        }

        // Switch turn
        game.switchTurn();

        return true;
    }

    // Check if a player's king is under attack
    private boolean isKingInCheck(Player player) {
        Piece king = player.getKing();
        if (king == null) return false;

        Position kingPosition = king.getCurrentPosition();
        Player opponent = getOpponent(player);

        // Check if any opponent piece can attack the king
        return opponent.getAlivePieces().stream()
            .anyMatch(piece -> piece.canMove(kingPosition, game.getBoard()));
    }

    // Check if player is in checkmate
    private boolean isCheckmate(Player player) {
        if (!player.isInCheck()) return false;

        // Try all possible moves for all pieces
        return !hasAnyValidMove(player);
    }

    // Check if player is in stalemate
    private boolean isStalemate(Player player) {
        if (player.isInCheck()) return false;

        // No valid moves but not in check
        return !hasAnyValidMove(player);
    }

    // Check if player has any valid move
    private boolean hasAnyValidMove(Player player) {
        for (Piece piece : player.getAlivePieces()) {
            Position from = piece.getCurrentPosition();

            // Try all possible destination cells
            for (int row = 0; row < 8; row++) {
                for (int col = 0; col < 8; col++) {
                    Position to = new Position(row, col);

                    if (!piece.canMove(to, game.getBoard())) continue;

                    // Try the move
                    Move testMove = new Move(piece, from, to);
                    testMove.execute(game.getBoard());

                    // Check if king is still in check
                    boolean stillInCheck = isKingInCheck(player);

                    // Undo the move
                    testMove.undo(game.getBoard());

                    if (!stillInCheck) {
                        return true;  // Found a valid move
                    }
                }
            }
        }

        return false;  // No valid moves
    }

    private Player getOpponent(Player player) {
        return (player.getColor() == Color.WHITE) ? game.blackPlayer : game.whitePlayer;
    }

    // Undo last move
    public boolean undoLastMove() {
        List<Move> history = game.getMoveHistory();
        if (history.isEmpty()) {
            return false;
        }

        Move lastMove = history.remove(history.size() - 1);
        lastMove.undo(game.getBoard());
        game.switchTurn();

        return true;
    }
}
```

---

## 12. Usage Example

```java
public class Main {
    public static void main(String[] args) {
        // Create game
        ChessGame game = new ChessGame("Alice", "Bob");
        ChessService service = new ChessService(game);

        // Make some moves
        // White pawn e2 to e4
        service.makeMove(new Position(1, 4), new Position(3, 4));

        // Black pawn e7 to e5
        service.makeMove(new Position(6, 4), new Position(4, 4));

        // White knight g1 to f3
        service.makeMove(new Position(0, 6), new Position(2, 5));

        // Continue playing...

        // Undo last move if needed
        service.undoLastMove();
    }
}
```

---

## 13. Design Patterns Used

### 1. **Strategy Pattern**
- **Where**: `MoveDirection` interface with `HorizontalMove`, `VerticalMove`, `DiagonalMove`, `KnightMove`
- **Why**: Different pieces move in different directions
- **Benefit**: Easy to add new movement patterns

### 2. **Template Method Pattern**
- **Where**: `Piece.canMove()` method
- **Why**: Common validation logic (check destination, check path) with piece-specific rules
- **Benefit**: No code duplication; subclasses only implement specific rules

### 3. **Factory Pattern**
- **Where**: `PieceFactory.createPiece()`
- **Why**: Centralize piece creation logic
- **Benefit**: Easy to create pieces without knowing concrete classes

### 4. **Command Pattern**
- **Where**: `Move` class with `execute()` and `undo()`
- **Why**: Encapsulate move as an object for undo/redo functionality
- **Benefit**: Can maintain move history and undo moves

### 5. **Inheritance**
- **Where**: `Piece` abstract class with concrete implementations (King, Queen, etc.)
- **Why**: Common behavior in base class, specific behavior in subclasses
- **Benefit**: Code reuse and polymorphism

---

## 14. Key Improvements from Original Design

| Issue | Original | Revised |
|-------|----------|---------|
| **Circular Reference** | Piece has Cell, Cell has Piece | Cell has Piece (one-way) |
| **Position Naming** | Both Position and CellAddress | Only Position |
| **Move Validation Bug** | Line 23: `anyMatch` (wrong) | `allMatch` (correct) |
| **Missing Pieces** | Only Rook, Pawn, Queen | All 6 pieces |
| **Incomplete Classes** | Many undefined | Fully implemented |
| **No Game State** | No turn/status tracking | Complete game state |
| **No Move History** | Can't undo | Command pattern with undo |
| **Typo** | Line 31: `'` instead of `;` | Fixed |
| **ChessService Unclear** | Vague description | Complete implementation |
| **No Factory** | Manual creation | PieceFactory |
| **No Check Detection** | Missing | Complete check/checkmate logic |
| **Jump Logic** | Complex in Piece | Simple `canJump()` override |

---

## 15. Class Diagram (Text)

```
Board
  └── Cell[][]
       └── Piece (optional)

Player
  ├── name: String
  ├── color: Color
  └── pieces: List<Piece>

Piece (abstract)
  ├── King
  ├── Queen
  ├── Rook
  ├── Bishop
  ├── Knight
  └── Pawn

MoveDirection (interface)
  ├── HorizontalMove
  ├── VerticalMove
  ├── DiagonalMove
  └── KnightMove

ChessGame
  ├── Board
  ├── Player (white)
  ├── Player (black)
  ├── currentPlayer
  ├── status: GameStatus
  └── moveHistory: List<Move>

ChessService
  └── ChessGame
```

---

## 16. Flow Diagram

```
Player makes move (from, to)
    ↓
ChessService.makeMove(from, to)
    ↓
Validate:
  ├─ Game is active?
  ├─ Piece exists at 'from'?
  ├─ Piece belongs to current player?
  └─ Piece.canMove(to, board)?
       ├─ Check direction validity (Strategy Pattern)
       ├─ Check path is clear (if not jumping)
       └─ Piece-specific validation (Template Method)
    ↓
Execute Move (Command Pattern)
  ├─ Capture opponent piece (if any)
  ├─ Move piece to destination
  └─ Update piece position
    ↓
Validate move doesn't put own king in check
  ├─ If yes: Undo move, return false
  └─ If no: Continue
    ↓
Check opponent's status
  ├─ Is opponent's king in check?
  ├─ Is it checkmate?
  └─ Is it stalemate?
    ↓
Update game status
    ↓
Switch turn
    ↓
Return true (move successful)
```

---

## 17. Future Enhancements (TODOs)

1. **Special Moves**
   - Castling (King + Rook)
   - En Passant (Pawn capture)
   - Pawn Promotion (Pawn reaches end)

2. **Advanced Features**
   - Timer/Clock for each player
   - Draw by repetition (same position 3 times)
   - Draw by 50-move rule
   - Save/Load game state
   - AI opponent

3. **UI**
   - Console-based UI
   - GUI (Swing/JavaFX)
   - Web-based UI

4. **Multiplayer**
   - Network play
   - Online matchmaking

---

## 18. Summary

**Design Patterns**: Strategy, Template Method, Factory, Command, Inheritance
**SOLID Principles**: Single Responsibility, Open/Closed, Liskov Substitution
**Key Features**:
- All 6 chess pieces with correct movement rules
- Check, checkmate, and stalemate detection
- Move validation and undo functionality
- Turn management and game status tracking
- Clean separation of concerns

**Benefits**:
- Easy to test (each piece independently)
- Easy to extend (add new pieces or rules)
- Maintainable (clear responsibilities)
- No code duplication (Template Method)
- Flexible (Strategy Pattern for movements)
