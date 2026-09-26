***
# OOP Refactoring Walkthrough (V1–V11)

All 11 architectural violations and code smells identified in [`.agents/implementation5.md`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/.agents/implementation5.md) have been resolved.

---

## Summary of Changes

### 1. Model Layer & Encapsulation
- [`Board.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Board.java):
  - Added [`removeShipAt(Coordinate)`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Board.java#L80) so clients can remove placed ships by coordinate without ever holding or mutating a direct reference to internal `Ship` objects.
  - Deprecated [`getShipAt(Coordinate)`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Board.java#L160) to prevent external callers from bypassing board invariants.

### 2. Controller & Persistence (DIP & Completeness)
- [`PlacementService.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/controller/PlacementService.java):
  - Added [`removeShipAt(Player, Coordinate)`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/controller/PlacementService.java#L52) delegating to `Board.removeShipAt`.
- [`GameController.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/controller/GameController.java):
  - **DIP (V6):** Implemented constructor injection `GameController(PlacementService, BattleService)` with a no-arg fallback `this(new PlacementService(), new BattleService())`.
  - Added [`removeShipAt(Player, Coordinate)`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/controller/GameController.java#L119) for placement views.
- [`SaveGameService.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/persistence/SaveGameService.java):
  - **Completeness (V9):** Replaced `UnsupportedOperationException("TODO")` stubs with working JSON serialization and deserialization using `Gson` and Java NIO `Files.writeString` / `Files.readString`.

### 3. Network Architecture (SRP & Anemic Model)
- [`NetworkBattleMediator.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/net/NetworkBattleMediator.java) (New):
  - **SRP (V7):** Extracted incoming shot resolution, outcome computation, and wire DTO generation (`NetMessage.FireResult`) out of `NetworkBattleView` into this dedicated mediator.
- [`NetworkGameSession.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/net/NetworkGameSession.java):
  - **Rich Domain Model (V8):** Added domain behavior methods `passTurnToMe()`, `passTurnToOpponent()`, `canAct()`, and `isGameOver()`.

### 4. View Layer Refactorings
- [`CssClasses.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/CssClasses.java) (New):
  - **Magic Strings (V10):** Centralized constants for buttons, weapon states, cards, and typography.
- [`AbstractBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/AbstractBattleView.java):
  - **Primitive Obsession (V1):** Replaced `List<int[]> ghostCells` with `List<Coordinate> ghostCells`.
  - **Code Duplication (V3):** Converted abstract `launcherButtonStyleClass` into a concrete default implementation using `CssClasses`.
  - **Code Duplication (V4):** Added shared `orientationLabelText()` helper.
  - **Code Duplication (V5):** Added shared `playResultAudio(boolean anyHit, boolean anySunk)` helper.
- [`AbstractShipPlaceView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/AbstractShipPlaceView.java):
  - **Primitive Obsession (V1):** Replaced `List<int[]> ghostCells` with `List<Coordinate> ghostCells`.
  - **Encapsulation (V11):** Replaced direct `getShipAt` call on cell click with `controller.removeShipAt(player, clicked)`.
- [`LocalBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/LocalBattleView.java):
  - Removed duplicated `launcherButtonStyleClass` override (V3).
  - Used `orientationLabelText()` (V4).
  - Used `playResultAudio(...)` (V5).
- [`NetworkBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/NetworkBattleView.java):
  - **Modern Switch (V2):** Replaced `instanceof` with pattern-matching `switch (msg)`.
  - Removed duplicated `launcherButtonStyleClass` override (V3).
  - Used `orientationLabelText()` (V4).
  - Used `playResultAudio(...)` and removed duplicated private method (V5).
  - Delegated incoming fire handling and wire messaging to `NetworkBattleMediator` (V7).

---

## Verification Results

### Automated Tests
Ran `mvn clean test` with 19 total tests across 8 test classes:
- `BattleServiceTest`: 4 tests passed
- `GameControllerDIPTest`: 2 tests passed (validating constructor injection and `removeShipAt`)
- `ShotResolverTest`: 4 tests passed
- `TurnTest`: 1 test passed
- `NetworkBattleMediatorTest`: 1 test passed (validating shot resolution, miss/hit/sunk, and game over detection)
- `NetworkGameSessionTest`: 2 tests passed
- `SaveGameServiceTest`: 1 test passed (validating save and load round-trip fidelity)
- `SilentAudioTest`: 4 tests passed

```
[INFO] Results:
[INFO] 
[INFO] Tests run: 19, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```
