# OOP Refactoring — Verified Implementation Record

**Date:** 2026-09-17  
**Build Status:** ✅ `mvn compile` — 55 source files, 0 errors  

---

## Phase 1: Fix Ship (Critical Bug + Encapsulation)

### [MODIFY] [Ship.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Ship.java)

**Bug Fixed:** The original `private int hits` counter would double-count when the same coordinate was fired at more than once (e.g. overlapping launcher patterns or re-firing at a cell). This caused ships to report sunk prematurely.

**Changes:**
- Replaced `private int hits` → `private final Set<Coordinate> hitCells = new HashSet<>()`
- `registerHit(c)` now uses `hitCells.add(c)` — **idempotent**, calling it twice with the same coordinate is a no-op
- `isSunk()` → `hitCells.size() >= type.getSize()`
- `getHits()` → `return hitCells.size()`
- `getOccupiedCells()` → `return Collections.unmodifiableList(occupiedCells)` — prevents callers from mutating the ship's internal coordinate list
- Added constructor validation: throws `IllegalArgumentException` if `occupiedCells.size() != type.getSize()`

**Impact:** `BattleView.java:551` calls `s.getHits()` — still returns `int`, no change needed. `Board.receiveShot()` calls `registerHit(c)` — same signature, no change. `BoardGridPane` calls `getOccupiedCells()` read-only — unmodifiable list is fine.

---

## Phase 2: Fix Board Encapsulation

### [MODIFY] [Board.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Board.java)

**Changes:**
- `getShips()` → returns `Collections.unmodifiableList(ships)` — all 24 call sites are read-only (iterating, `.size()`, `.stream().filter()`), none mutate the list directly
- `receiveShot(c)` → added **guard clause**: if cell is already HIT/MISS/SUNK, returns current status as a no-op result without re-processing
- `receiveShot(c)` → added **bounds check**: throws `IllegalArgumentException` for out-of-bounds coordinates
- `getCellStatus(c)` → added **bounds check**: same as above
- `getShipAt(c)` → added **bounds check**: same as above

---

## Phase 3: Introduce AmmoInventory + Refactor Player

### [NEW] [AmmoInventory.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/AmmoInventory.java)

Centralizes ammunition management that was previously scattered across `Player` as raw `int` fields with manual increment/decrement in every caller.

**API:**
| Method | Description |
|:---|:---|
| `AmmoInventory(int boardSize)` | Initializes all `LauncherType` ammo from `getStartingAmmo()` |
| `getAmmo(LauncherType)` | Current count (`Integer.MAX_VALUE` for infinite) |
| `hasAmmo(LauncherType)` | `true` if count > 0 |
| `isInfinite(LauncherType)` | `true` if count == `Integer.MAX_VALUE` |
| `consume(LauncherType)` | Decrements by 1 (no-op for infinite; throws if 0) |
| `resupply(LauncherType, int)` | Adds ammo (no-op for infinite) |

### [MODIFY] [Player.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/Player.java)

**Removed (6 methods):**
- `getLevel2Ammo()`, `setLevel2Ammo(int)`
- `getNuclearAmmo()`, `setNuclearAmmo(int)`
- `int level2Ammo`, `int nuclearAmmo` fields

**Added:**
- `private AmmoInventory ammo`
- `public AmmoInventory getAmmo()` — returns the inventory object
- `initLauncherAmmo(boardSize)` delegates to `new AmmoInventory(boardSize)`

### [MODIFY] [GameController.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/controller/GameController.java)

| Location | Before | After |
|:---|:---|:---|
| `setSelectedLauncher()` | `player.getLevel2Ammo() <= 0` | `!player.getAmmo().hasAmmo(type)` |
| `getAmmoRemaining()` | switch on type → `player.getLevel2Ammo()` / `player.getNuclearAmmo()` | `player.getAmmo().getAmmo(type)` |
| `fireLauncher()` | `setLevel2Ammo(getLevel2Ammo()-1)` | `attacker.getAmmo().consume(type)` |
| `fireAiLauncher()` | `player2.getLevel2Ammo(), player2.getNuclearAmmo()` | `player2.getAmmo()` |

### [MODIFY] [BattleView.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/BattleView.java)

| Location | Before | After |
|:---|:---|:---|
| Nuclear ammo check before fire | `attacker.getNuclearAmmo()` | `attacker.getAmmo().hasAmmo(LauncherType.NUCLEAR)` |
| Nuclear resupply check | `player.getNuclearAmmo() == 0` | `!player.getAmmo().hasAmmo(LauncherType.NUCLEAR)` |
| Nuclear resupply action | `player.setNuclearAmmo(player.getNuclearAmmo() + 1)` | `player.getAmmo().resupply(LauncherType.NUCLEAR, 1)` |

### [MODIFY] [NetworkBattleView.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/NetworkBattleView.java)

| Location | Before | After |
|:---|:---|:---|
| Level 2 consume | `me.setLevel2Ammo(me.getLevel2Ammo() - 1)` | `me.getAmmo().consume(LauncherType.LEVEL_2)` |
| Nuclear consume | `me.setNuclearAmmo(me.getNuclearAmmo() - 1)` | `me.getAmmo().consume(LauncherType.NUCLEAR)` |
| Nuclear empty check | `me.getNuclearAmmo() == 0` | `!me.getAmmo().hasAmmo(LauncherType.NUCLEAR)` |
| Nuclear resupply | `me.setNuclearAmmo(me.getNuclearAmmo() + 1)` | `me.getAmmo().resupply(LauncherType.NUCLEAR, 1)` |

### [MODIFY] [AIStrategy.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/ai/AIStrategy.java)

- `chooseShotPlan(Board, int, int)` → `chooseShotPlan(Board, AmmoInventory)`

### [MODIFY] [HuntTargetAI.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/ai/HuntTargetAI.java)

- Updated `chooseShotPlan` signature and body: `level2Ammo > 0` → `ammo.hasAmmo(LauncherType.LEVEL_2) && !ammo.isInfinite(LauncherType.LEVEL_2)`

### [MODIFY] [SmartAI.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/ai/SmartAI.java)

- Updated `chooseShotPlan` signature and body: `nuclearAmmo > 0` → `ammo.hasAmmo(LauncherType.NUCLEAR) && !ammo.isInfinite(LauncherType.NUCLEAR)`

---

## Phase 4: Polymorphic LauncherType — Eliminate LauncherLogic

### [MODIFY] [LauncherType.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/model/LauncherType.java)

Added `public abstract List<Coordinate> getTargetCells(Coordinate anchor, boolean horizontal)` — each enum constant implements its own blast pattern:

| Constant | Pattern |
|:---|:---|
| `DEFAULT` | Single cell: `List.of(anchor)` |
| `LEVEL_2` | 1×3 line (horizontal) or 3×1 line (vertical) |
| `NUCLEAR` | 2×3 block (horizontal) or 3×2 block (vertical) |

### [DELETE] [LauncherLogic.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/controller/LauncherLogic.java)

All 8 call sites that previously called `LauncherLogic.getTargetCells(type, anchor, horizontal)` now call `type.getTargetCells(anchor, horizontal)` directly:

- `GameController.fireLauncher()`
- `BattleView.showGhost()`, `BattleView.handleFire()`
- `NetworkBattleView.handleIncomingFire()`, `NetworkBattleView.showGhost()`, `NetworkBattleView.handleFireClick()`

---

## Phase 5: Composition Over Inheritance — AI Refactoring

### [NEW] [TargetingQueue.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/ai/TargetingQueue.java)

Extracts the target-following queue logic that was duplicated between `HuntTargetAI` and `SmartAI`.

**API:**
| Method | Description |
|:---|:---|
| `hasTargets()` | Are there queued cells to pursue? |
| `nextTarget(Board)` | Returns next valid unshot target, or `null` |
| `enqueueNeighbors(Coordinate, int)` | Adds 4 orthogonal neighbors (bounds-filtered) |
| `clear()` | Clears queue (called when a ship sinks) |

### [MODIFY] [HuntTargetAI.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/ai/HuntTargetAI.java)

- `protected Deque<Coordinate> queue` → `private final TargetingQueue targetQueue`
- `protected SecureRandom random` → `private final SecureRandom random`
- `chooseTarget()` → delegates to `targetQueue.nextTarget(enemyBoard)`
- `notifyResult()` → delegates to `targetQueue.enqueueNeighbors()` / `targetQueue.clear()`

### [MODIFY] [SmartAI.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/ai/SmartAI.java)

**Before:** `extends HuntTargetAI` — inherited queue + random; shadowed `random` with its own instance  
**After:** `implements AIStrategy` — owns its own `TargetingQueue` and single `SecureRandom`

- Has `private final TargetingQueue targetQueue` (composition, not inheritance)
- Implements `notifyResult()` directly (was relying on inherited version)
- Probability density logic and `bestBlock()` helper preserved verbatim
- No more shadowed `SecureRandom random` field

### [MODIFY] [AIFactory.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/ai/AIFactory.java)

- Added `private AIFactory() {}` — prevents accidental instantiation of this utility class

---

## Phase 6: NetworkGameSession Encapsulation

### [MODIFY] [NetworkGameSession.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/net/NetworkGameSession.java)

**Before:** 6 bare `public` fields, no constructor  
**After:** Proper encapsulated class

| Field | Visibility | Notes |
|:---|:---|:---|
| `session` | `private final` | via `getSession()` |
| `theater` | `private final` | via `getTheater()` |
| `isHost` | `private final` | via `isHost()` |
| `me` | `private final` | via `getMe()` |
| `enemyTracker` | `private final` | via `getEnemyTracker()` |
| `myTurn` | `private volatile` | via `isMyTurn()` / `setMyTurn(boolean)` — `volatile` for thread safety |

Constructor: `NetworkGameSession(NetworkSession session, Theater theater, boolean isHost, Player me, EnemyTracker enemyTracker)`

### [MODIFY] [EnemyTracker.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/net/EnemyTracker.java)

- `getKnownSunkShips()` → `return Collections.unmodifiableList(knownSunkShips)`

### [MODIFY] [HostLobbyView.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/HostLobbyView.java)

```java
// Before (7 lines of field assignment):
NetworkGameSession netSession = new NetworkGameSession();
netSession.session = session;
netSession.theater = theater;
// ... etc

// After (3 lines via constructor):
Player me = new Player("You (Host)", true, new Board(size));
me.initLauncherAmmo(size);
NetworkGameSession netSession = new NetworkGameSession(session, theater, true, me, new EnemyTracker(size));
```

### [MODIFY] [JoinLobbyView.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/JoinLobbyView.java)

Same constructor pattern as HostLobbyView.

### [MODIFY] [NetworkShipPlaceView.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/NetworkShipPlaceView.java)

All direct field accesses replaced with getter/setter calls:
| Before | After |
|:---|:---|
| `netSession.me` | `netSession.getMe()` |
| `netSession.session.setOnMessage(...)` | `netSession.getSession().setOnMessage(...)` |
| `netSession.session.setOnDisconnected(...)` | `netSession.getSession().setOnDisconnected(...)` |
| `netSession.isHost` | `netSession.isHost()` |
| `netSession.myTurn = hostFirst` | `netSession.setMyTurn(hostFirst)` |
| `netSession.session.send(...)` | `netSession.getSession().send(...)` |
| `netSession.session.close()` | `netSession.getSession().close()` |

### [MODIFY] [NetworkBattleView.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/NetworkBattleView.java)

~30 field access sites updated to use getters/setters. Key patterns:
| Before | After |
|:---|:---|
| `netSession.me` | `netSession.getMe()` |
| `netSession.myTurn` (read) | `netSession.isMyTurn()` |
| `netSession.myTurn = true/false` | `netSession.setMyTurn(true/false)` |
| `netSession.enemyTracker` | `netSession.getEnemyTracker()` |
| `netSession.session` | `netSession.getSession()` |

### [MODIFY] [NetworkGameOverView.java](file:///home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/view/NetworkGameOverView.java)

| Before | After |
|:---|:---|
| `netSession.session` | `netSession.getSession()` |
| `netSession.me.getOwnBoard()` | `netSession.getMe().getOwnBoard()` |
| `netSession.enemyTracker` | `netSession.getEnemyTracker()` |

---

## Phase 7: Build Verification

```
$ mvn compile
[INFO] Compiling 55 source files with javac [debug target 17] to target/classes
[INFO] BUILD SUCCESS
[INFO] Total time:  2.313 s
```

### Post-build verification:
- ✅ Zero references to deleted `LauncherLogic` class
- ✅ Zero references to removed `getLevel2Ammo/setLevel2Ammo/getNuclearAmmo/setNuclearAmmo` methods
- ✅ Zero direct field accesses on `NetworkGameSession` (all go through getters/setters)
- ✅ All 55 source files compile cleanly

---

## Files Changed Summary

| Phase | New Files | Modified Files | Deleted Files |
|:---:|:---:|:---:|:---:|
| 1 | 0 | 1 (Ship) | 0 |
| 2 | 0 | 1 (Board) | 0 |
| 3 | 1 (AmmoInventory) | 7 (Player, GameController, BattleView, NetworkBattleView, AIStrategy, HuntTargetAI, SmartAI) | 0 |
| 4 | 0 | 4 (LauncherType, GameController, BattleView, NetworkBattleView) | 1 (LauncherLogic) |
| 5 | 1 (TargetingQueue) | 3 (HuntTargetAI, SmartAI, AIFactory) | 0 |
| 6 | 0 | 7 (NetworkGameSession, EnemyTracker, HostLobbyView, JoinLobbyView, NetworkShipPlaceView, NetworkBattleView, NetworkGameOverView) | 0 |
| **Total** | **2** | **~18 unique** | **1** |

