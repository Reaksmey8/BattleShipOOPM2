***
# 🎓 OOP Design Review — Battleship: Naval Command

> **Reviewing Professor's Verdict:** This project shows *above-average* OOP maturity for a student submission. The fundamentals — MVC layering, Strategy pattern for AI, Factory for object creation, immutable value objects — are genuinely solid. However, several structural issues betray a partially procedural mindset hiding behind Java classes. What follows is an honest, line-by-line critique.

---

## OOP Grade & Summary

| Criterion | Grade | One-Line |
|---|---|---|
| **Encapsulation** | B+ | Good field privacy; defensive copies present. Marred by anemic `Player` and public-field DTOs. |
| **Inheritance vs. Composition** | A− | Correct use of composition (`TargetingQueue` inside `SmartAI`). No misuse of inheritance. |
| **Polymorphism** | B | `AIStrategy` is textbook Strategy. But the View layer has massive duplication instead of polymorphic screens. |
| **SOLID** | C+ | SRP violated by `GameController` (God Object) and by `BattleView`/`NetworkBattleView`. OCP violated by `LauncherType`'s hardcoded `switch` blocks. |
| **Code Smells** | C | God Objects, Primitive Obsession, massive View-layer copy-paste, all-public DTO. |
| **Overall** | **B−** | A solid domain model undermined by a bloated controller, duplicated views, and several missed abstraction opportunities. |

---

## Architecture Violations

### 1. 🔴 `GameController` — God Object / SRP Violation

**File:** [`GameController.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/controller/GameController.java)
**Lines:** 248 lines, 11+ distinct responsibilities

This single class handles:
- Game state machine transitions
- Mode/Theater selection
- Player creation and initialization
- Ship placement logic (validate, place, remove, auto-place, confirm-ready)
- Turn management (initiative roll, turn switching)
- Launcher system (select, toggle, ammo query)
- Firing logic (multi-cell resolution, sunk-ship detection)
- AI turn delegation
- Callback/event plumbing
- Pass-screen routing for hotseat

> **Why it's bad:** Any change to *any* of these subsystems risks breaking the others. A student adding "save/load" or "replay" would be forced to modify this already-overloaded class. This directly violates **SRP** (one reason to change) and **OCP** (open for extension, closed for modification).

---

### 2. 🔴 `BattleView` ↔ `NetworkBattleView` — Massive Copy-Paste (DRY Violation)

**Files:**
- [`BattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/BattleView.java) — 582 lines
- [`NetworkBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/NetworkBattleView.java) — 459 lines

These two files share **~60–70% identical code**: launcher bar building, ghost preview rendering, orientation toggling, board card construction, fire validation, `confirmExit()`, sound effect triggers. The network variant copies all of this and re-implements the firing flow with socket I/O.

> **Why it's bad:** A bug fix in one view (e.g., ghost rendering) must be manually replicated in the other. This is a textbook case where an **abstract base class** or **Template Method pattern** should extract the shared skeleton, leaving only the "how to resolve a shot" as the polymorphic hook.

---

### 3. 🟡 `NetMessage` — All Public Fields, No Encapsulation

**File:** [`NetMessage.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/net/NetMessage.java)

```java
public class NetMessage {
    public String type;
    public String code;
    public String theater;
    // ... 10+ public fields
}
```

Every field is `public` with no validation, no type safety on `type`, and inner classes (`CellResult`, `SunkInfo`) also have all-public fields. This is a **C struct masquerading as a Java class**.

> **Why it's bad:** Any code anywhere can set `type = "BANANA"` and the system won't catch it until a silent `default -> {}` in a switch. The `type` field screams for an `enum`, and the message variants scream for a **sealed interface** or at least a discriminated union with factory methods.

---

### 4. 🟡 `GameSaveDTO` — Anemic Data Model with All Public Fields

**File:** [`GameSaveDTO.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/persistence/GameSaveDTO.java)

```java
public class GameSaveDTO {
    public String version;
    public String timestamp;
    public int boardSize;
    // ... all public, no behavior
}
```

Same problem as `NetMessage`. A DTO is acceptable for serialization boundaries, but it should still use `private` fields + a builder or `record`, not wide-open `public` access.

---

### 5. 🟡 `LauncherType` — OCP Violation via Hardcoded `switch`

**File:** [`LauncherType.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/LauncherType.java#L72-L87)

```java
public boolean isAvailableFor(int boardSize) {
    return switch (this) {
        case DEFAULT -> true;
        case LEVEL_2 -> boardSize >= 8;
        case NUCLEAR -> true;
    };
}

public int getStartingAmmo(int boardSize) {
    return switch (this) {
        case DEFAULT -> Integer.MAX_VALUE;
        case LEVEL_2 -> boardSize >= 10 ? 3 : (boardSize >= 8 ? 2 : 0);
        case NUCLEAR -> 1;
    };
}
```

The `getTargetCells` method correctly uses **polymorphism** (abstract + override per constant). But `isAvailableFor` and `getStartingAmmo` regress back to `switch` on `this`. Adding a new launcher type (e.g., `TORPEDO`) requires modifying *every* switch in this enum — classic **OCP violation**.

> **Fix:** These should be abstract methods with per-constant overrides, exactly like `getTargetCells` already is.

---

### 6. 🟡 `Player` — Anemic Domain Model + Exposed Mutability

**File:** [`Player.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Player.java)

`Player` is almost entirely getters/setters. It has exactly one behavior method (`hasLost()`). The launcher state (`selectedLauncher`, `launcherHorizontal`) is directly mutated by the controller via raw setters. This is **feature envy** — the controller is doing `Player`'s job.

```java
// Controller reaches in and mutates Player's internal state:
player.setSelectedLauncher(type);       // should be player.selectLauncher(type, boardSize)
player.setLauncherHorizontal(false);    // should be player.toggleLauncherOrientation()
```

---

### 7. 🟡 Primitive Obsession — `boolean horizontal`

Across the entire codebase, ship and launcher orientation is passed as a raw `boolean horizontal`. This appears in:
- [`Ship`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Ship.java#L14) constructor
- [`Board.placeShip`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Board.java#L51)
- [`LauncherType.getTargetCells`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/LauncherType.java#L69)
- [`Player.isLauncherHorizontal`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Player.java#L44)
- [`AiShotPlan`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/ai/AiShotPlan.java)

> **Fix:** Create an `enum Orientation { HORIZONTAL, VERTICAL }` — self-documenting, extensible (e.g., diagonal placements in a variant), and eliminates `true/false` ambiguity at every call site.

---

### 8. 🟡 `MainApp` — Procedural Navigation via Method Explosion

**File:** [`MainApp.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/MainApp.java)

```java
public void showMainMenu() { setRoot(new MainMenuView(this).build()); }
public void showModeSelect() { ... }
public void showMultiplayerLobby() { ... }
public void showBoardSelect() { ... }
public void showShipPlacement() { ... }
public void showPassScreen(Runnable onContinue) { ... }
public void showBattle() { ... }
public void showGameOver(Player winner) { ... }
```

Every screen transition is a separate hand-written method. Adding a new screen requires adding a new method here. This is **procedural dispatch** — the `GameState` enum already exists but isn't being used polymorphically.

---

### 9. 🟡 `DecorUtil` + `SoundGenerator` — Utility Class Overload (338 + 484 lines)

**Files:**
- [`DecorUtil.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/DecorUtil.java) — 484 lines
- [`SoundGenerator.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/SoundGenerator.java) — 338 lines

Both are `final class` with `private` constructor and only `static` methods. This is acceptable for true utility concerns, but at 484 and 338 lines respectively, they've grown into dumping grounds. `DecorUtil` has four completely independent scene generators that share nothing.

---

### 10. 🟡 `SmartAI.bestBlock` — Hardcoded Dimensions Bypass the Type System

**File:** [`SmartAI.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/ai/SmartAI.java#L142-L144)

```java
int[][] dims = type == LauncherType.NUCLEAR
        ? new int[][]{{2, 3}, {3, 2}}
        : new int[][]{{1, 3}, {3, 1}};
```

The AI manually encodes each launcher's blast-pattern dimensions instead of asking the `LauncherType` for its shape. If a new launcher is added, this code silently produces wrong results.

---

## What the Code Gets Right ✅

Before prescribing fixes, credit where it's due:

| Strength | Where |
|---|---|
| **Strategy Pattern** for AI difficulty | [`AIStrategy`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/ai/AIStrategy.java) interface + `RandomAI`, `HuntTargetAI`, `SmartAI` |
| **Factory Pattern** | [`AIFactory`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/ai/AIFactory.java) cleanly maps difficulty → strategy |
| **Composition over Inheritance** | `SmartAI` composes `TargetingQueue` instead of extending `HuntTargetAI` |
| **Immutable Value Object** | [`Coordinate`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Coordinate.java) with `equals`/`hashCode`, `final` fields |
| **Java Records** | [`ShotResult`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/ShotResult.java), [`LauncherFireResult`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/controller/LauncherFireResult.java), [`AiShotPlan`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/ai/AiShotPlan.java) |
| **Defensive copies** | `Board.getShips()` returns `Collections.unmodifiableList(...)` |
| **Enum with behavior** | `LauncherType.getTargetCells()` is polymorphic per constant |
| **Clean domain separation** | Model package has zero JavaFX imports |
| **Aggregate Root pattern** | `Board` owns grid + ships + shot resolution — well-bounded |

---

## Suggested Design Patterns

### 1. **Template Method** — Unify `BattleView` and `NetworkBattleView`

Extract an abstract `AbstractBattleView` containing all shared UI building (board cards, launcher bar, ghost preview, sound triggers), and define a single abstract `resolveShot(Coordinate)` hook. Local play resolves via `GameController`; network play sends a `NetMessage` and awaits the result.

### 2. **State Pattern** — Replace `GameController`'s implicit state machine

The controller uses `GameState` as a dumb enum + manual `if/else` branching. A proper **State Pattern** would let each state (`PlacingState`, `BattlingState`, `GameOverState`) encapsulate its own transition logic, eliminating the monolithic controller.

### 3. **Command Pattern** — Encapsulate shots as objects

A `FireCommand` (containing anchor, launcher type, orientation, attacker, defender) would decouple shot resolution from the controller, enable undo/replay, and unify local and network firing into the same pipeline.

### 4. **Observer Pattern** — Decouple state → view updates

Replace the ad-hoc `Consumer<GameState> onStateChanged` with a proper event bus or observer list, so multiple listeners can react to state changes without the controller knowing about any specific view.

### 5. **Sealed Interface** — Type-safe network messages

Replace the stringly-typed `NetMessage.type` with a `sealed interface` hierarchy where each message variant is its own record with only the fields it actually needs.

### 6. **Builder Pattern** — For `GameSaveDTO`

Replace the all-public-field DTO with a builder that validates invariants at construction time.

---

## Refactored Design & Code

### Refactor 1: Extract `Orientation` Enum (Primitive Obsession fix)

```java
package com.battleship.model;

/** Ship and launcher orientation — replaces raw boolean flags. */
public enum Orientation {
    HORIZONTAL,
    VERTICAL;

    public boolean isHorizontal() { return this == HORIZONTAL; }

    public Orientation toggle() {
        return this == HORIZONTAL ? VERTICAL : HORIZONTAL;
    }
}
```

**Impact:** Every `boolean horizontal` parameter and field across the codebase becomes `Orientation orientation` — self-documenting and extensible.

---

### Refactor 2: Eliminate `LauncherType` switch blocks (OCP fix)

```java
public enum LauncherType {

    DEFAULT("Default", 1) {
        @Override
        public List<Coordinate> getTargetCells(Coordinate anchor, Orientation orient) {
            return List.of(anchor);
        }
        @Override
        public boolean isAvailableFor(int boardSize) { return true; }
        @Override
        public int getStartingAmmo(int boardSize)    { return Integer.MAX_VALUE; }
    },

    LEVEL_2("Level 2", 3) {
        @Override
        public List<Coordinate> getTargetCells(Coordinate anchor, Orientation orient) {
            List<Coordinate> cells = new ArrayList<>(3);
            for (int i = 0; i < 3; i++) {
                int r = orient.isHorizontal() ? anchor.getRow() : anchor.getRow() + i;
                int c = orient.isHorizontal() ? anchor.getCol() + i : anchor.getCol();
                cells.add(new Coordinate(r, c));
            }
            return cells;
        }
        @Override
        public boolean isAvailableFor(int boardSize) { return boardSize >= 8; }
        @Override
        public int getStartingAmmo(int boardSize) {
            return boardSize >= 10 ? 3 : (boardSize >= 8 ? 2 : 0);
        }
    },

    NUCLEAR("Nuclear", 6) {
        @Override
        public List<Coordinate> getTargetCells(Coordinate anchor, Orientation orient) {
            int rows = orient.isHorizontal() ? 2 : 3;
            int cols = orient.isHorizontal() ? 3 : 2;
            List<Coordinate> cells = new ArrayList<>(6);
            for (int r = 0; r < rows; r++)
                for (int c = 0; c < cols; c++)
                    cells.add(new Coordinate(anchor.getRow() + r, anchor.getCol() + c));
            return cells;
        }
        @Override
        public boolean isAvailableFor(int boardSize) { return true; }
        @Override
        public int getStartingAmmo(int boardSize)    { return 1; }
    };

    // ... constructor, fields ...

    public abstract List<Coordinate> getTargetCells(Coordinate anchor, Orientation orient);
    public abstract boolean isAvailableFor(int boardSize);
    public abstract int getStartingAmmo(int boardSize);
}
```

> Now adding a `TORPEDO` launcher requires *only* adding a new enum constant — zero changes to existing code. True OCP compliance.

---

### Refactor 3: Decompose `GameController` (SRP fix)

```
com.battleship.controller/
├── GameController.java         ← slim orchestrator (state transitions + delegates)
├── PlacementService.java       ← ship placement logic (place, remove, auto-place, validate)
├── BattleService.java          ← firing, turn management, launcher selection
├── LauncherFireResult.java     ← (existing record, stays)
```

#### `PlacementService.java`
```java
package com.battleship.controller;

import com.battleship.model.*;
import java.security.SecureRandom;
import java.util.*;

/**
 * Encapsulates all ship-placement logic, extracted from GameController.
 */
public class PlacementService {

    private static final SecureRandom RANDOM = new SecureRandom();

    public Map<ShipType, Integer> getRemainingShipCounts(Player player, Theater theater) {
        Map<ShipType, Integer> remaining = new LinkedHashMap<>(theater.getFleetComposition());
        for (Ship s : player.getOwnBoard().getShips()) {
            remaining.merge(s.getType(), -1, Integer::sum);
        }
        remaining.values().removeIf(v -> v <= 0);
        return remaining;
    }

    public boolean placeShip(Player player, Theater theater,
                             ShipType type, Coordinate start, Orientation orient) {
        Map<ShipType, Integer> remaining = getRemainingShipCounts(player, theater);
        if (!remaining.containsKey(type) || remaining.get(type) <= 0) return false;
        return player.getOwnBoard().placeShip(type, start, orient.isHorizontal());
    }

    public boolean canPlace(Player player, ShipType type, Coordinate start, Orientation orient) {
        return player.getOwnBoard().isValidPlacement(type, start, orient.isHorizontal());
    }

    public boolean removeShip(Player player, Ship ship) {
        return player.getOwnBoard().removeShip(ship);
    }

    public boolean isPlacementComplete(Player player, Theater theater) {
        return player.getOwnBoard().getShips().size() == theater.getTotalShipCount();
    }

    public void resetPlacement(Player player) {
        player.getOwnBoard().clearShips();
    }

    public void autoPlaceAll(Player player, Theater theater) {
        Map<ShipType, Integer> remaining = getRemainingShipCounts(player, theater);
        int size = theater.getBoardSize();
        for (Map.Entry<ShipType, Integer> entry : remaining.entrySet()) {
            for (int i = 0; i < entry.getValue(); i++) {
                boolean placed = false;
                for (int attempt = 0; attempt < 10_000 && !placed; attempt++) {
                    placed = player.getOwnBoard().placeShip(
                            entry.getKey(),
                            new Coordinate(RANDOM.nextInt(size), RANDOM.nextInt(size)),
                            RANDOM.nextBoolean());
                }
            }
        }
    }
}
```

#### `BattleService.java`
```java
package com.battleship.controller;

import com.battleship.model.*;
import java.util.*;

/**
 * Encapsulates turn management, launcher selection, and firing logic.
 */
public class BattleService {

    public boolean selectLauncher(Player player, LauncherType type, int boardSize) {
        if (!type.isAvailableFor(boardSize)) return false;
        if (!player.getAmmo().hasAmmo(type)) return false;
        player.setSelectedLauncher(type);
        return true;
    }

    public void toggleOrientation(Player player) {
        player.setLauncherHorizontal(!player.isLauncherHorizontal());
    }

    public LauncherFireResult fire(Player attacker, Player defender) {
        LauncherType type = attacker.getSelectedLauncher();
        int size = defender.getOwnBoard().getSize();

        List<Coordinate> cells = type.getTargetCells(
                /* anchor provided externally */ null,
                attacker.isLauncherHorizontal());
        // ... shot resolution logic (moved from GameController.fireLauncher)
        // Returns LauncherFireResult
        throw new UnsupportedOperationException("See full refactored fire()");
    }
}
```

The slimmed-down `GameController` becomes a pure **Mediator** — it holds references to `PlacementService` and `BattleService` and delegates to them:

```java
public class GameController {
    private final PlacementService placementService = new PlacementService();
    private final BattleService battleService = new BattleService();
    // ... state machine + delegation only
}
```

---

### Refactor 4: Enrich `Player` (fix Anemic Domain Model)

```java
public class Player {

    private final String name;
    private final boolean isHuman;
    private final Board ownBoard;
    private AmmoInventory ammo;
    private LauncherType selectedLauncher = LauncherType.DEFAULT;
    private Orientation launcherOrientation = Orientation.HORIZONTAL;

    // ... constructor ...

    /** Attempts to select a launcher; returns false if unavailable or out of ammo. */
    public boolean selectLauncher(LauncherType type, int boardSize) {
        if (!type.isAvailableFor(boardSize)) return false;
        if (ammo != null && !ammo.hasAmmo(type)) return false;
        this.selectedLauncher = type;
        return true;
    }

    /** Toggles launcher orientation between HORIZONTAL and VERTICAL. */
    public void toggleLauncherOrientation() {
        this.launcherOrientation = launcherOrientation.toggle();
    }

    /** Resets launcher to DEFAULT after a shot (per game rules). */
    public void resetLauncherAfterShot() {
        this.selectedLauncher = LauncherType.DEFAULT;
    }

    public boolean hasLost() { return ownBoard.isAllShipsSunk(); }

    // Only expose immutable/read-only accessors
    public String getName() { return name; }
    public boolean isHuman() { return isHuman; }
    public Board getOwnBoard() { return ownBoard; }
    public AmmoInventory getAmmo() { return ammo; }
    public LauncherType getSelectedLauncher() { return selectedLauncher; }
    public Orientation getLauncherOrientation() { return launcherOrientation; }
}
```

> The `setSelectedLauncher()` and `setLauncherHorizontal()` raw setters are replaced by behavioral methods that enforce business rules internally. The controller no longer reaches in to mutate `Player` state — it *asks* the `Player` to perform an action.

---

### Refactor 5: Sealed Interface for `NetMessage` (Type Safety fix)

```java
package com.battleship.net;

import com.battleship.model.*;
import java.util.List;

/** Type-safe network protocol — each message variant carries only its own fields. */
public sealed interface NetMessage {

    record Hello(String code) implements NetMessage {}

    record Welcome(Theater theater) implements NetMessage {}

    record Reject(String reason) implements NetMessage {}

    record Ready() implements NetMessage {}

    record Start(String firstPlayer) implements NetMessage {}

    record Fire(LauncherType launcherType, Coordinate anchor,
                Orientation orientation) implements NetMessage {}

    record FireResult(List<CellOutcome> results,
                      List<SunkShipInfo> sunkShips,
                      boolean defenderLost) implements NetMessage {}

    // Value types for the FireResult payload
    record CellOutcome(Coordinate coordinate, CellStatus outcome) {}
    record SunkShipInfo(ShipType type, List<Coordinate> cells) {}
}
```

> Now `handleMessage` uses pattern matching instead of stringly-typed dispatch:
> ```java
> switch (msg) {
>     case NetMessage.Fire fire   -> handleIncomingFire(fire);
>     case NetMessage.FireResult r -> handleFireResult(r);
>     default -> { /* ignore */ }
> }
> ```
> The compiler enforces exhaustiveness. Typos like `"BANANA"` are impossible.

---

### Refactor 6: Template Method for Battle Views (DRY fix)

```java
package com.battleship.view;

/**
 * Shared skeleton for both local and network battle screens.
 * Subclasses only override how a shot is resolved.
 */
public abstract class AbstractBattleView {

    protected final MainApp app;
    protected BoardGridPane ownGrid;
    protected BoardGridPane enemyGrid;
    protected Label turnLabel;
    protected HBox launcherBar;
    // ... all shared fields ...

    protected AbstractBattleView(MainApp app) {
        this.app = app;
    }

    /** Template method: builds the complete battle UI. */
    public final StackPane build() {
        // 1. Build command bar         (shared)
        // 2. Build launcher bar        (shared)
        // 3. Build own + enemy grids   (shared)
        // 4. Attach fire handlers      (shared — delegates to resolveShot)
        // 5. Build side panel          (hook: subclass can customize)
        // 6. Assemble layout           (shared)
        attachFireHandlers();
        return root;
    }

    /** Hook: subclass provides the board size for the enemy grid. */
    protected abstract int getEnemyGridSize();

    /** Hook: subclass resolves a shot — locally or over the network. */
    protected abstract void resolveShot(Coordinate anchor);

    /** Hook: subclass determines if it's the local player's turn. */
    protected abstract boolean isMyTurn();

    // ... all shared methods: refreshLauncherBar, showGhost, clearGhost,
    //     toggleOrientation, buildBoardCard, confirmExit, etc. ...
}
```

```java
/** Local (AI / Hotseat) battle — resolves shots via GameController. */
public class LocalBattleView extends AbstractBattleView {
    private final GameController controller;
    // ...
    @Override
    protected void resolveShot(Coordinate anchor) {
        LauncherFireResult result = controller.fireLauncher(anchor);
        applyResult(enemyGrid, result);
        // ...
    }
}

/** Network battle — resolves shots by sending FIRE and awaiting FIRE_RESULT. */
public class NetworkBattleView extends AbstractBattleView {
    private final NetworkGameSession netSession;
    // ...
    @Override
    protected void resolveShot(Coordinate anchor) {
        NetMessage.Fire fire = new NetMessage.Fire(type, anchor, orientation);
        netSession.getSession().send(fire);
        // Result arrives asynchronously via handleMessage
    }
}
```

> **Result:** ~400 lines of duplicated code eliminated. Bug fixes apply to both views automatically.

---

## Summary of Refactored Class Structure

```
com.battleship/
├── model/
│   ├── Orientation.java          [NEW]   — replaces boolean horizontal
│   ├── Coordinate.java           [KEEP]  — already excellent
│   ├── CellStatus.java           [KEEP]
│   ├── ShipType.java             [KEEP]
│   ├── Ship.java                 [KEEP]  — uses Orientation now
│   ├── Board.java                [KEEP]  — uses Orientation now
│   ├── Theater.java              [KEEP]
│   ├── GameMode.java             [KEEP]
│   ├── GameState.java            [KEEP]
│   ├── LauncherType.java         [MODIFY] — switch→abstract methods
│   ├── AmmoInventory.java        [KEEP]
│   ├── ShotResult.java           [KEEP]
│   └── Player.java               [MODIFY] — behavioral methods, remove setters
│
├── controller/
│   ├── GameController.java       [MODIFY] — slim mediator, delegates to services
│   ├── PlacementService.java     [NEW]   — extracted from GameController
│   ├── BattleService.java        [NEW]   — extracted from GameController
│   └── LauncherFireResult.java   [KEEP]
│
├── ai/
│   ├── AIStrategy.java           [KEEP]  — already excellent Strategy pattern
│   ├── AIFactory.java            [KEEP]
│   ├── RandomAI.java             [KEEP]
│   ├── HuntTargetAI.java         [KEEP]
│   ├── SmartAI.java              [MODIFY] — use LauncherType's own shape info
│   ├── TargetingQueue.java       [KEEP]
│   ├── AiShotPlan.java           [MODIFY] — uses Orientation
│   └── Difficulty.java           [KEEP]
│
├── net/
│   ├── NetMessage.java           [MODIFY] — sealed interface + records
│   ├── NetworkSession.java       [KEEP]
│   ├── NetworkGameSession.java   [KEEP]  — remove raw setMyTurn setter
│   ├── EnemyTracker.java         [KEEP]
│   └── NetUtil.java              [KEEP]
│
├── persistence/
│   ├── GameSaveDTO.java          [MODIFY] — private fields + builder
│   └── SaveGameService.java      [KEEP]  — (still TODO)
│
└── view/
    ├── AbstractBattleView.java   [NEW]   — Template Method base
    ├── LocalBattleView.java      [NEW]   — was BattleView, now slim
    ├── NetworkBattleView.java    [MODIFY] — extends AbstractBattleView
    ├── BoardGridPane.java        [KEEP]
    ├── ShipPlaceView.java        [KEEP]
    ├── ShipDockPane.java         [KEEP]
    ├── MainApp.java              [KEEP]  — (ideally use State for navigation)
    └── ... other views ...       [KEEP]
```

---

## Final Remarks

This project demonstrates genuine understanding of when to use **Strategy**, **Factory**, and **Composition**. The domain model (`Board`, `Ship`, `Coordinate`) is clean, well-encapsulated, and free of framework dependencies. These are real strengths that many student projects lack.

The primary weaknesses — the God Object controller, the duplicated battle views, and the missed `Orientation` abstraction — are all *correctable architectural habits* rather than fundamental misunderstandings. The path from a B− to an A is shorter than it looks: decompose the controller, extract the shared battle template, and replace the remaining `switch` blocks with polymorphic overrides.

> **Bottom line:** Good OOP instincts, inconsistently applied. The code knows *what* patterns to use but doesn't yet have the discipline to apply them *everywhere*.
