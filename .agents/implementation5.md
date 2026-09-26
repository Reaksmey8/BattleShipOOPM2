# OOP Violations — Agent Fix List

All fixes below are independent and can be applied in any order.

---

## V1: Primitive Obsession — `ghostCells` uses `int[]` instead of `Coordinate`

**Files:**
- [`AbstractBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/AbstractBattleView.java)
- [`AbstractShipPlaceView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/AbstractShipPlaceView.java)

**Fix:** Change `List<int[]> ghostCells` to `List<Coordinate> ghostCells`. Replace all `new int[]{c.getRow(), c.getCol()}` with just `c`. Replace all `rc[0], rc[1]` accesses with `gc.getRow(), gc.getCol()`.

```java
// BEFORE
private final List<int[]> ghostCells = new ArrayList<>();
ghostCells.add(new int[]{c.getRow(), c.getCol()});
for (int[] rc : ghostCells) { repaintGhostCell(rc[0], rc[1]); }

// AFTER
private final List<Coordinate> ghostCells = new ArrayList<>();
ghostCells.add(c);
for (Coordinate gc : ghostCells) { repaintGhostCell(gc.getRow(), gc.getCol()); }
```

---

## V2: `instanceof` chain instead of pattern-matching `switch`

**File:** [`NetworkBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/NetworkBattleView.java) — `handleMessage()` method

**Fix:** Replace `instanceof` + cast with a `switch` expression over the sealed `NetMessage`:

```java
// BEFORE
private void handleMessage(NetMessage msg) {
    if (msg == null) return;
    if (msg instanceof NetMessage.Fire) {
        handleIncomingFire((NetMessage.Fire) msg);
    } else if (msg instanceof NetMessage.FireResult) {
        handleFireResult((NetMessage.FireResult) msg);
    }
}

// AFTER
private void handleMessage(NetMessage msg) {
    if (msg == null) return;
    switch (msg) {
        case NetMessage.Fire f        -> handleIncomingFire(f);
        case NetMessage.FireResult fr -> handleFireResult(fr);
        default -> { /* lobby-phase messages ignored during battle */ }
    }
}
```

---

## V3: Duplicated `launcherButtonStyleClass()` — identical in both subclasses

**Files:**
- [`LocalBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/LocalBattleView.java) — delete the override
- [`NetworkBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/NetworkBattleView.java) — delete the override
- [`AbstractBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/AbstractBattleView.java) — change from `abstract` to a concrete default

**Fix:** Make `launcherButtonStyleClass()` a non-abstract method with the shared implementation in `AbstractBattleView`. Remove the identical overrides from both subclasses.

```java
// In AbstractBattleView — change from abstract to default implementation:
protected String launcherButtonStyleClass(LauncherButtonState state) {
    return switch (state) {
        case DISABLED -> "weapon-button-disabled";
        case SELECTED -> "weapon-button-selected";
        case ENABLED  -> "weapon-button-enabled";
    };
}
```

---

## V4: Duplicated `updateOrientationLabel()` in both subclasses

**Files:**
- [`LocalBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/LocalBattleView.java)
- [`NetworkBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/NetworkBattleView.java)

**Fix:** Extract the shared orientation label text into a `protected` helper in `AbstractBattleView`. Both subclasses call the shared helper instead of duplicating the logic.

```java
// Add to AbstractBattleView:
protected final String orientationLabelText() {
    Orientation o = firingPlayer().getLauncherOrientation();
    return "Orientation: " + (o.isHorizontal() ? "HORIZONTAL" : "VERTICAL")
            + "  (R or Right-Click to rotate — affects Level 2 / Nuclear)";
}
```

Then in both subclasses, `updateOrientationLabel()` becomes:
```java
private void updateOrientationLabel() {
    orientationLabel.setText(orientationLabelText());
}
```

---

## V5: Duplicated audio-result logic (3 places)

**Files:**
- [`LocalBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/LocalBattleView.java) — `applyResult()` method
- [`NetworkBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/NetworkBattleView.java) — `playResultAudio()` and `handleIncomingFire()`

**Fix:** Add a shared method to `AbstractBattleView`, then have both subclasses call it:

```java
// Add to AbstractBattleView:
protected final void playResultAudio(boolean anyHit, boolean anySunk) {
    if (anySunk)       audio.playSunk();
    else if (anyHit)   audio.playHit();
    else               audio.playMiss();
}
```

Replace the duplicated if/else audio blocks in all three locations with `playResultAudio(anyHit, anySunk)`.

---

## V6: Services not injected in `GameController` (DIP violation)

**File:** [`GameController.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/controller/GameController.java)

**Fix:** Add constructor injection with a no-arg convenience constructor:

```java
// BEFORE
private final PlacementService placementService = new PlacementService();
private final BattleService battleService = new BattleService();

// AFTER
private final PlacementService placementService;
private final BattleService battleService;

/** Testable constructor — inject dependencies. */
public GameController(PlacementService placementService, BattleService battleService) {
    this.placementService = placementService;
    this.battleService = battleService;
}

/** Production convenience constructor. */
public GameController() {
    this(new PlacementService(), new BattleService());
}
```

---

## V7: Feature Envy — `NetworkBattleView.handleIncomingFire()` does controller work

**File:** [`NetworkBattleView.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/NetworkBattleView.java) — `handleIncomingFire()` method

**Fix:** Create a new class `NetworkBattleMediator` in the `com.battleship.net` package that encapsulates shot resolution + DTO construction + network reply. The view should only call the mediator and render the result.

**New file:** `src/main/java/com/battleship/net/NetworkBattleMediator.java`

```java
package com.battleship.net;

import com.battleship.controller.LauncherFireResult;
import com.battleship.controller.ShotResolver;
import com.battleship.model.Player;
import com.battleship.model.Ship;
import com.battleship.model.ShotResult;

import java.util.ArrayList;
import java.util.List;

public class NetworkBattleMediator {

    private final NetworkGameSession session;
    private final Player me;

    public NetworkBattleMediator(NetworkGameSession session) {
        this.session = session;
        this.me = session.getMe();
    }

    public record IncomingFireOutcome(
            LauncherFireResult resolution, boolean lost,
            boolean anyHit, boolean anySunk) {}

    public IncomingFireOutcome resolveAndReply(NetMessage.Fire fire) {
        LauncherFireResult resolution = ShotResolver.resolve(
                me.getOwnBoard(), fire.launcherType(), fire.anchor(), fire.orientation());

        boolean lost = me.getOwnBoard().isAllShipsSunk();
        boolean anyHit = resolution.results().stream().anyMatch(ShotResult::isHit);
        boolean anySunk = !resolution.sunkShips().isEmpty();

        List<NetMessage.CellResult> cellResults = new ArrayList<>();
        for (ShotResult r : resolution.results()) {
            cellResults.add(new NetMessage.CellResult(r.coordinate(), r.outcome()));
        }
        List<NetMessage.SunkShipInfo> sunkInfos = new ArrayList<>();
        for (Ship s : resolution.sunkShips()) {
            sunkInfos.add(new NetMessage.SunkShipInfo(s.getType(), s.getOccupiedCells()));
        }
        session.getSession().send(new NetMessage.FireResult(cellResults, sunkInfos, lost));

        return new IncomingFireOutcome(resolution, lost, anyHit, anySunk);
    }
}
```

Then simplify `NetworkBattleView.handleIncomingFire()` to only rendering + audio (no DTO construction, no `session.send()`).

---

## V8: `NetworkGameSession` is an Anemic Domain Model

**File:** [`NetworkGameSession.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/net/NetworkGameSession.java)

**Fix:** Move turn-transition logic and the "am I the defender?" check into `NetworkGameSession` as behavioral methods rather than having views query raw state and mutate turns externally. For example, the `resolveAndReply()` logic from V7's mediator could live directly on this class instead, giving it real domain behavior.

---

## V9: `SaveGameService` is a stub (throws `UnsupportedOperationException`)

**File:** [`SaveGameService.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/persistence/SaveGameService.java)

**Fix:** Either implement the `save()` and `load()` methods, or remove the class entirely if save/load is out of scope. Shipping dead code with `throw new UnsupportedOperationException("TODO")` is a code smell.

---

## V10: Magic CSS class strings scattered across 15+ files

**Files:** All view files that reference style classes like `"weapon-button-disabled"`, `"board-card-title"`, `"side-card"`, etc.

**Fix:** Create a constants class:

```java
package com.battleship.view;

public final class CssClasses {
    private CssClasses() {}

    public static final String WEAPON_DISABLED = "weapon-button-disabled";
    public static final String WEAPON_SELECTED = "weapon-button-selected";
    public static final String WEAPON_ENABLED  = "weapon-button-enabled";
    public static final String BOARD_CARD       = "board-card";
    public static final String SIDE_CARD        = "side-card";
    // ... etc
}
```

Then replace all string literals with constant references.

---

## V11: `Board.getShipAt()` returns a mutable `Ship` reference

**File:** [`Board.java`](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Board.java)

**Fix:** Either remove `getShipAt()` if it's only used internally, or return a read-only view/DTO of the ship. The risk is that external code could call `ship.registerHit()` directly, bypassing `Board.receiveShot()` and breaking cell-status invariants.
