***
# OOP Refactoring Implementation Plan

Refactor the BattleShip project to fix the critical issues and design flaws identified in the OOP review. Changes are organized into **7 phases**, ordered by dependency (foundational model changes first, then outward to controller → view).

> [!IMPORTANT]
> Each phase is designed to keep the project **compilable** after completion. No phase leaves dangling references.

---

## Phase 1: Fix Ship (Critical Bug + Encapsulation)

### [MODIFY] [Ship.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/model/Ship.java)
- Replace `private int hits` with `private final Set<Coordinate> hitCells = new HashSet<>()`
- `registerHit(c)` → uses `hitCells.add(c)` (idempotent — fixes the double-counting bug)
- `isSunk()` → `hitCells.size() >= type.getSize()`
- `getHits()` → `return hitCells.size()`
- `getOccupiedCells()` → return `Collections.unmodifiableList(occupiedCells)`
- Add constructor validation: `occupiedCells.size() == type.getSize()`

**Impact:** `BattleView.java:551` calls `s.getHits()` — still returns `int`, no change needed. `Board.receiveShot()` calls `registerHit(c)` — same signature, no change. `BoardGridPane` calls `getOccupiedCells()` read-only — unmodifiable list is fine.

---

## Phase 2: Fix Board Encapsulation

### [MODIFY] [Board.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/model/Board.java)
- `getShips()` → return `Collections.unmodifiableList(ships)`
- `receiveShot(c)` → add guard: if cell already HIT/MISS/SUNK, return current status as no-op result
- Add bounds check in `receiveShot()`, `getCellStatus()`, `getShipAt()`

**Impact:** 24 call sites use `getShips()` — all are **read-only** (iterating, `.size()`, `.stream().filter()`). No caller mutates the returned list directly. Safe change.

---

## Phase 3: Introduce AmmoInventory + Refactor Player

### [NEW] [AmmoInventory.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/model/AmmoInventory.java)
- `Map<LauncherType, Integer>` keyed storage
- Methods: `getAmmo(type)`, `hasAmmo(type)`, `consume(type)`, `resupply(type, amount)`, `isInfinite(type)`
- Constructor takes `boardSize`, initializes all types from `LauncherType.getStartingAmmo()`

### [MODIFY] [Player.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/model/Player.java)
- Replace `int level2Ammo`, `int nuclearAmmo` → `private final AmmoInventory ammo`
- Remove `getLevel2Ammo/setLevel2Ammo/getNuclearAmmo/setNuclearAmmo` (6 methods)
- Add `getAmmo()` returning the `AmmoInventory` object
- Keep `initLauncherAmmo(boardSize)` but have it delegate to `new AmmoInventory(boardSize)` 
  - Actually, make ammo lazy-init: constructor doesn't create it, `initLauncherAmmo` does (preserves network code where Player is constructed before ammo init)
- Keep `selectedLauncher` and `launcherHorizontal` for now (UI state — lower priority fix)

### [MODIFY] [GameController.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/controller/GameController.java)
- `setSelectedLauncher()` L165-166: replace `player.getLevel2Ammo() <= 0` → `!player.getAmmo().hasAmmo(type)`
- `getAmmoRemaining()` L175-181: replace switch → `player.getAmmo().getAmmo(type)`
- `fireLauncher()` L207-208: replace `setLevel2Ammo(getLevel2Ammo()-1)` etc → `attacker.getAmmo().consume(type)`
- `fireAiLauncher()` L228: replace `player2.getLevel2Ammo(), player2.getNuclearAmmo()` → `player2.getAmmo().getAmmo(LEVEL_2), player2.getAmmo().getAmmo(NUCLEAR)`

### [MODIFY] [BattleView.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/view/BattleView.java)
- L453, L483: `attacker.getNuclearAmmo()` → `attacker.getAmmo().getAmmo(LauncherType.NUCLEAR)`
- L532: `player.getNuclearAmmo() == 0` → `!player.getAmmo().hasAmmo(LauncherType.NUCLEAR)`
- L534: `player.setNuclearAmmo(...)` → `player.getAmmo().resupply(LauncherType.NUCLEAR, 1)`

### [MODIFY] [NetworkBattleView.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/view/NetworkBattleView.java)
- L425: `me.setLevel2Ammo(...)` → `me.getAmmo().consume(LauncherType.LEVEL_2)`
- L427: `me.setNuclearAmmo(...)` → `me.getAmmo().consume(LauncherType.NUCLEAR)`
- L428: `me.getNuclearAmmo() == 0` → `!me.getAmmo().hasAmmo(LauncherType.NUCLEAR)`
- L430: `me.setNuclearAmmo(...)` → `me.getAmmo().resupply(LauncherType.NUCLEAR, 1)`

### [MODIFY] [AIStrategy.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/ai/AIStrategy.java)
- L25-27: change `chooseShotPlan(Board, int, int)` → `chooseShotPlan(Board, AmmoInventory)`

### [MODIFY] [HuntTargetAI.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/ai/HuntTargetAI.java)
- L72-85: update `chooseShotPlan` signature and body to use `AmmoInventory`

### [MODIFY] [SmartAI.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/ai/SmartAI.java)
- L94-109: update `chooseShotPlan` signature and body to use `AmmoInventory`

---

## Phase 4: Polymorphic LauncherType — Eliminate LauncherLogic

### [MODIFY] [LauncherType.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/model/LauncherType.java)
- Add `public abstract List<Coordinate> getTargetCells(Coordinate anchor, boolean horizontal)` 
- Each enum constant implements its own blast pattern
- DEFAULT: single cell. LEVEL_2: 1×3 line. NUCLEAR: 2×3 block.

### [DELETE] [LauncherLogic.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/controller/LauncherLogic.java)

### [MODIFY] [GameController.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/controller/GameController.java)
- L194: `LauncherLogic.getTargetCells(type, anchor, ...)` → `type.getTargetCells(anchor, ...)`
- Remove import of `LauncherLogic`

### [MODIFY] [BattleView.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/view/BattleView.java)
- L389, L422-423: `LauncherLogic.getTargetCells(...)` → `type.getTargetCells(...)`
- Remove import of `LauncherLogic`

### [MODIFY] [NetworkBattleView.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/view/NetworkBattleView.java)
- L155, L363, L395: `LauncherLogic.getTargetCells(...)` → `type.getTargetCells(...)`
- Remove import of `LauncherLogic`

### [MODIFY] [SmartAI.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/ai/SmartAI.java)
- Update hardcoded dimensions to use `LauncherType.NUCLEAR.getCellCount()` etc (secondary cleanup)

---

## Phase 5: Composition Over Inheritance — AI Refactoring

### [NEW] [TargetingQueue.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/ai/TargetingQueue.java)
- Encapsulates the `Deque<Coordinate>` + methods: `hasTargets()`, `nextTarget(Board)`, `enqueueNeighbors(Coordinate, int)`, `clear()`

### [MODIFY] [HuntTargetAI.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/ai/HuntTargetAI.java)
- Replace `protected Deque<Coordinate> queue` → `private final TargetingQueue targetQueue`
- Make fields `private` instead of `protected`
- Update `chooseTarget`, `notifyResult`, `chooseShotPlan` to use `targetQueue`

### [MODIFY] [SmartAI.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/ai/SmartAI.java)
- Change `extends HuntTargetAI` → `implements AIStrategy`
- Add `private final TargetingQueue targetQueue = new TargetingQueue()` (composition)
- Remove shadowed `private final SecureRandom random` (use single instance)
- Keep probability density logic in `chooseTarget()`, delegate target-following to `targetQueue`

### [MODIFY] [AIFactory.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/ai/AIFactory.java)
- Add `private AIFactory() {}` (prevent instantiation)
- Return `Optional<AIStrategy>` instead of nullable (or keep null but document it — lower priority)

---

## Phase 6: NetworkGameSession Encapsulation

### [MODIFY] [NetworkGameSession.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/net/NetworkGameSession.java)
- Change all 6 public fields → `private`
- Add constructor: `NetworkGameSession(NetworkSession session, Theater theater, boolean isHost, Player me, EnemyTracker enemyTracker)`
- Add getters
- Add `setMyTurn(boolean)` / `isMyTurn()` for the mutable turn flag
- Mark `myTurn` as `volatile` for thread safety

### [MODIFY] [EnemyTracker.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/net/EnemyTracker.java)
- `getKnownSunkShips()` → return `Collections.unmodifiableList(knownSunkShips)`

### [MODIFY] All 5 network view files that access `netSession.xxx` directly:
- [HostLobbyView.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/view/HostLobbyView.java): L132-138 → use constructor
- [JoinLobbyView.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/view/JoinLobbyView.java): L119-125 → use constructor
- [NetworkShipPlaceView.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/view/NetworkShipPlaceView.java): L59, L150-151, L170-172, L194, L201-204 → use getters/setters
- [NetworkBattleView.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/view/NetworkBattleView.java): L55, L59, L80, L82, L123-124, L213-216, L264-266, L339, L360, L363, L394, L396-402, L425-435, L444, L451-453 → use getters/setters
- [NetworkGameOverView.java](file:///home/debrouillez-vous/Downloads/Projects/BattleShipGameOOP/src/main/java/com/battleship/view/NetworkGameOverView.java): L42, L106, L129, L134, L138 → use getters

---

## Phase 7: Verify

### Build Verification
- Run `mvn compile` to ensure no compilation errors across all 55 source files

### Manual Verification
- Verify the game launches and plays through a full local AI game
- Verify ship placement → battle → game over flow works

---

## Open Questions

> [!IMPORTANT]
> ~1,200 LOC across `ShipPlaceView`/`NetworkShipPlaceView` and `BattleView`/`NetworkBattleView`). Please, Extract shared abstract base classes.

> [!IMPORTANT]
> **Constructor change in Player:** Network code (`HostLobbyView`, `JoinLobbyView`) currently does `new Player(name, true, board)` and then calls `initLauncherAmmo(size)` as a separate step. Should I change Player's constructor to accept `boardSize` and initialize ammo eagerly, or keep the two-step pattern?

---

## Files Changed Summary

| Phase | New Files | Modified Files | Deleted Files |
|:---:|:---:|:---:|:---:|
| 1 | 0 | 1 (Ship) | 0 |
| 2 | 0 | 1 (Board) | 0 |
| 3 | 1 (AmmoInventory) | 7 (Player, GameController, BattleView, NetworkBattleView, AIStrategy, HuntTargetAI, SmartAI) | 0 |
| 4 | 0 | 4 (LauncherType, GameController, BattleView, NetworkBattleView) | 1 (LauncherLogic) |
| 5 | 1 (TargetingQueue) | 3 (HuntTargetAI, SmartAI, AIFactory) | 0 |
| 6 | 0 | 6 (NetworkGameSession, EnemyTracker, HostLobbyView, JoinLobbyView, NetworkShipPlaceView, NetworkBattleView, NetworkGameOverView) | 0 |
| **Total** | **2** | **~18 unique** | **1** |
m