The task is complete. Here's the full summary:

## What I found

Before writing any code, I cross-checked every issue in `.agents/Implementation7.md` against the current codebase (which has evolved through commits `Refactor-1` → `Refactor-3.4`). **8 of the 9 violations are already fixed** in the current code:

| Doc Issue | Status in current code |
|---|---|
| 1. OCP — weapon/launcher system | ✅ `LauncherType` has polymorphic `getTargetCells()`; `BattleService` has a **single** `fire(Coordinate)`; `GameController` delegates via `fireLauncher()` with no switch |
| 2. `Player.getBoard()` encapsulation leak | ✅ Split into `getOwnBoard()` (read-only) and documented `getMutableBoard()` escape hatch; views only receive `ReadOnlyBoard` |
| 3. `LocalBattleView` god object | ✅ Shared machinery extracted into `AbstractBattleView` (Template Method); save-game serialization removed from the view |
| 4. Primitive obsession (`int[]`) | ✅ `Coordinate` record used everywhere; no `List<int[]>` remains |
| 5. Direct view construction | ✅ All views navigate via the `ViewNavigator` interface; only `MainApp` (the composition root) constructs views |
| 6. Duplicated AI neighbour logic | ✅ Half fixed via shared `TargetingQueue.enqueueNeighbors()` — **but see below** |
| 7. `ShipPlaceView` empty transfer loop | ✅ Gone; placement persists directly on the placing player's board |
| 8. `AmmoInventory` per-type methods | ✅ `EnumMap`-based generic API (`getAmmo/hasAmmo/consume/resupply`) |
| 9. `SoundManager` per-shot audio creation | ✅ SFX bytes cached at init; views use the injected `GameAudio` facade |

## What I implemented

The **one residual violation** matching the doc's Issue-6 pattern: the HUNT-mode checkerboard-parity search was still copy-pasted in **three places** — `HuntTargetAI.chooseTarget`, `HuntTargetAI.chooseShotPlan`, and `SmartAI.fallbackHunt`.

1. **Created** `/home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/main/java/com/battleship/ai/ParityHunter.java` — a stateless, single-source HUNT heuristic (`ParityHunter.pick(Board, Random)`), following the project's existing utility-class conventions.
2. **Refactored** `HuntTargetAI.java` — both duplicated parity blocks now delegate to `ParityHunter.pick()`; removed the now-unused `ArrayList`/`List` imports.
3. **Refactored** `SmartAI.java` — `fallbackHunt()` is now a one-line delegate.
4. **Added** `/home/debrouillez-vous/Downloads/Projects/vB/battleshipGameOOP/src/test/java/com/battleship/ai/ParityHunterTest.java` with 4 tests: parity preference on a fresh board, fallback to odd-parity cells when even cells are exhausted, seeded determinism, and an integration check that the refactored `HuntTargetAI` still plans legal, ammo-respecting shots.

## Verification

- `mvn test` → **BUILD SUCCESS, exit 0**: 23 tests run, 0 failures, 0 errors (19 pre-existing + 4 new).

**Note on the doc's "Suggested Design Patterns"** (Command, Observer, State, Router): these were aspirational suggestions rather than violations; the codebase already satisfies their intent through the existing `ViewNavigator` abstraction, `Consumer<GameState>` callbacks, and the `GameState` enum. I deliberately did not introduce those patterns to avoid over-engineering. If you'd like any of them implemented as well, let me know which.