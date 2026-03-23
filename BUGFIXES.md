# WaterBot Bug Fixes

Four root-cause bugs were identified and fixed that caused the bot to skip crops or water the same tile multiple times.

---

## Bug 1 — Thread-pool Race Condition

**Files:** `WaterBot/ModEntry.cs`, `WaterBot/Framework/WaterBotController.cs`

### Problem
`Task.Delay(800).ContinueWith(o => { this.startWatering(...); })` scheduled the next watering action on a .NET thread-pool thread. Stardew Valley and SMAPI are single-threaded; accessing game state from a background thread caused corruption, duplicate triggers, and skipped tiles.

### Fix
Removed `Task.Delay`. A tick-based state machine now drives the delay on the main thread via SMAPI's `GameLoop.UpdateTicked` event:

- `ModEntry.cs` subscribes to `UpdateTicked` and forwards each tick to `bot.onUpdateTick()`.
- `WaterBotController` adds `waitUntilTick`, `pendingCharacter`, and `pendingLocation` fields.
- After each watering animation, `startWatering` sets `waitUntilTick = Game1.ticks + 48` (~800 ms at 60 fps) instead of spawning a task.
- `onUpdateTick()` checks each frame and resumes when the counter expires.

---

## Bug 2 — Blocked Tile Used as Default Standing Point

**File:** `WaterBot/Framework/Map.cs` — `findFillPath()`

### Problem
When a crop tile was impassable (e.g. a trellis crop), the code searched for a walkable adjacent neighbor to stand on. However, `bestOption` was initialised to the **blocked tile itself** and `score = 0`. Because adjacent tile scores were compared with `>` (strict), a neighbor with score 0 would not replace the default. If no neighbor scored above 0, the blocked tile remained the stand destination. `PathFindController` would immediately fire its callback (destination unreachable) from wherever the player was standing, causing watering at wrong positions or silent skips.

### Fix
```diff
- int score = 0;
- Tile bestOption = this.map[current.y][current.x];
+ int score = -1;
+ Tile? bestOption = null;
```
Also added `score = neighborScore` inside the winning branch so the score updates correctly. The existing `if (bestOption == null) { continue; }` guard now correctly skips tiles with no reachable stand.

---

## Bug 3 — Diagonal Neighbors Not Pushed to DFS Stack

**File:** `WaterBot/Framework/Map.cs` — `findFillPath()`

### Problem
`findFillPath` uses a depth-first search to build the watering order. The cardinal-directions loop correctly did two things for each neighbor: marked it `watered = true` and pushed it onto the DFS stack. The **diagonals loop** marked neighbors `watered = true` but **never pushed them onto the stack**. A diagonally-adjacent tile was permanently locked out of traversal — tiles reachable only through it were either skipped or visited out-of-order by the outer `keepGoing` pass.

### Fix
Added the missing stack push inside the diagonals loop:

```diff
  if (neighbor.waterable && !neighbor.watered)
  {
      actionable.pushExecuteOn(neighbor.getPoint());
      neighbor.watered = true;
  }
+ if (!neighbor.visited && group.Contains(neighbor))
+ {
+     stack.Add(neighbor);
+ }
```

---

## Bug 4 — PathFindController Callback Fires from Wrong Position

**File:** `WaterBot/Framework/WaterBotController.cs` — `startWatering()`

### Problem
`PathFindController` fires its "arrived" callback both on successful arrival **and** when it gives up on an unreachable destination. `startWatering` never verified that the player actually reached the intended stand point, so it would water from wherever the player happened to be — producing wrong facing direction, mis-targeted watering, and silent skips.

### Fix
Added a Manhattan distance check at the start of `startWatering`. If the player is more than 1 tile away from the intended stand point, the actionable tile is skipped cleanly by calling `navigate()`:

```csharp
Point stand = this.order[this.currentTile].getStand();
Point playerTile = Game1.player.TilePoint;
if (Math.Abs(playerTile.X - stand.X) + Math.Abs(playerTile.Y - stand.Y) > 1)
{
    this.navigate();
    return;
}
```

---

## Bonus — Pending Timer Cleared on Interrupt

`stop()` and `exhausted()` now set `waitUntilTick = 0` so that a pending tick delay cannot fire after the bot has been stopped.

---

## Summary

| # | Root Cause | File(s) | Impact |
|---|---|---|---|
| 1 | `Task.Delay` on thread-pool thread | `ModEntry.cs`, `WaterBotController.cs` | Double-watering, skips, crashes |
| 2 | `bestOption` initialised to blocked tile | `Map.cs` | Bot stands on impassable tile, skips groups |
| 3 | Diagonal DFS neighbors not stacked | `Map.cs` | Diagonal crops silently skipped |
| 4 | No position check after `PathFindController` | `WaterBotController.cs` | Waters from wrong tile on unreachable stands |
