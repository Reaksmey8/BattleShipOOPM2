***
# 🔧 OOP Fixes Checklist

All 8 issues to fix, in order. Each has exact file locations and what to do.

---

### Fix 1: Introduce `ReadOnlyBoard` interface to seal the Board aggregate

**Files:**
- [Board.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Board.java)
- [Player.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Player.java) — L73

**Problem:** `Player.getOwnBoard()` returns the mutable `Board`. Views call `player.getOwnBoard().placeShip()`, `receiveShot()`, `clearShips()` directly, bypassing `Player` entirely.

**Fix:**
1. Create a new `ReadOnlyBoard` interface in `com.battleship.model` with only query methods: `getSize()`, `getCellStatus(Coordinate)`, `getShips()`, `isAllShipsSunk()`, `getUnshotCells()`.
2. Make `Board implements ReadOnlyBoard`.
3. Change `Player.getOwnBoard()` return type to `ReadOnlyBoard`.
4. Add a package-private `Board getMutableBoard()` on `Player` for use by `PlacementService`, `BattleService`, `ShotResolver`, and `NetworkBattleMediator`.
5. Update all call sites: views use `ReadOnlyBoard`, services use `getMutableBoard()`.

---

### Fix 2: Stop exposing mutable `Player` from `GameController`

**File:** [GameController.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/controller/GameController.java) — L217–218

**Problem:** `getPlayer1()` and `getPlayer2()` return mutable `Player` references. Views call `me.consumeAmmo()`, `me.resetLauncherAfterShot()`, `me.resupplyAmmo()` directly.

**Fix:**
1. Add read-only query methods to `GameController` for everything views actually need: `getPlayerName(int)`, `getPlayerBoard(int) → ReadOnlyBoard`, etc.
2. Make `getPlayer1()` / `getPlayer2()` package-private (only services need them).
3. In views that currently call `player.consumeAmmo()` or `player.resetLauncherAfterShot()` directly (especially `NetworkBattleView.resolveShot()` L298–320), route those mutations through a controller/service method instead.

---

### Fix 3: Delete deprecated `Board.getShipAt()`

**File:** [Board.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Board.java) — L153–163

**Problem:** `@Deprecated` but still compiled. Leaks mutable `Ship` references from the internal `shipGrid`. `removeShipAt(Coordinate)` already exists as the proper replacement.

**Fix:** Delete lines 153–163 entirely. Verify no call sites remain (search for `getShipAt` across the project). If any exist, replace with `removeShipAt()` or `getCellStatus()`.

---

### Fix 4: Convert `Coordinate` to a Java record

**File:** [Coordinate.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Coordinate.java)

**Problem:** Manually implements `equals()`, `hashCode()`, `toString()`, and has `getRow()`/`getCol()` — exactly what a `record` provides for free with compile-time immutability.

**Fix:**
```java
public record Coordinate(int row, int col) {
    public int getRow() { return row; }
    public int getCol() { return col; }
    public boolean isWithinBounds(int boardSize) {
        return row >= 0 && row < boardSize && col >= 0 && col < boardSize;
    }
}
```
The `getRow()`/`getCol()` aliases maintain backward compatibility with all existing call sites.

---

### Fix 5: Make `ShotResolver` injectable (extract interface)

**Files:**
- [ShotResolver.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/controller/ShotResolver.java)
- [BattleService.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/controller/BattleService.java)

**Problem:** `ShotResolver` is a static utility class with `private` constructor. Cannot be injected, mocked, or swapped. Violates DIP.

**Fix:**
1. Create a `@FunctionalInterface ShotResolution` interface with one method: `LauncherFireResult resolve(Board, LauncherType, Coordinate, Orientation)`.
2. Make `ShotResolver` implement `ShotResolution`. Expose a `public static final ShotResolver STANDARD = new ShotResolver()` singleton.
3. Change `BattleService` to accept `ShotResolution` via constructor injection (default to `ShotResolver.STANDARD` in the no-arg constructor).
4. Do the same for `NetworkBattleMediator` if it calls `ShotResolver.resolve()` directly.

---

### Fix 6: Decouple `NetworkSession` from `Platform.runLater`

**File:** [NetworkSession.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/net/NetworkSession.java)

**Problem:** A networking class imports `javafx.application.Platform` to marshal callbacks onto the JavaFX thread. Makes the `net` package untestable without a JavaFX runtime.

**Fix:**
1. Add a `Consumer<Runnable> threadDispatcher` parameter to the constructor / factory methods.
2. Replace all `Platform.runLater(r)` calls with `threadDispatcher.accept(r)`.
3. Production callers pass `Platform::runLater`. Tests pass `Runnable::run`.
4. Overloaded factory methods (no dispatcher arg) default to `Platform::runLater` for backward compatibility.

---

### Fix 7: Move domain mutations out of `NetworkBattleView.resolveShot()`

**File:** [NetworkBattleView.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/NetworkBattleView.java) — L298–320

**Problem:** The view method performs 5 domain mutations (`consumeAmmo`, `resetLauncherAfterShot`, `resupplyAmmo`, `beginOpponentTurn`). This is Feature Envy — the logic belongs in the controller or a service.

**Fix:**
1. Create a method on `GameController` (or a new `NetworkFireService`) like `fireNetworkShot(Player, LauncherType, Coordinate, Orientation)` that handles ammo consumption, launcher reset, and returns what needs to be sent over the wire.
2. `NetworkBattleView.resolveShot()` calls that one method and then sends the `NetMessage.Fire`. The view only does UI + network I/O.

---

### Fix 8: `MainApp.start()` bypasses its own `GameAudio` abstraction

**File:** [MainApp.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/MainApp.java) — L29

**Problem:** `SoundManager.getInstance().playMenuMusic()` is called directly, even though `MainApp` implements `ViewNavigator` which provides `getAudio()`.

**Fix:** Replace L29 with `getAudio().playMenuMusic()`.
