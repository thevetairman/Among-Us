# Among Us Task Distribution — Reverse Engineering Findings

> **Game Version:** Among Us (Timestamp: 69B99610 — 3/17/2026)  
> **Platform:** PC (Steam)  
> **Tools:** Il2CppDumper, dnSpy v6.1.8, Ghidra 11.2

---

## Summary

This document presents findings from reverse engineering Among Us's `GameAssembly.dll` to determine exactly how tasks are distributed to players at the start of a game. The commonly held belief that tasks are assigned independently and randomly to each player is **incorrect**. The actual algorithm uses a **single shared shuffled pool** that all players draw from sequentially, with a wrap-around reshuffle when the pool is exhausted.

---

## Methodology

Among Us is a Unity game compiled with IL2CPP, which converts C# code into native C++. This makes direct decompilation impossible, but the following toolchain reconstructs readable pseudocode:

1. **Il2CppDumper** — fed `GameAssembly.dll` and `global-metadata.dat` from the Among Us install folder. Outputs `DummyDll/Assembly-CSharp.dll` containing all class/method names with RVA memory addresses, and `script.json` + `ghidra_with_struct.py` for Ghidra labeling.

2. **dnSpy** — loads `Assembly-CSharp.dll` to browse class and method names. Used to locate target functions and retrieve their VA addresses (e.g. `PlayerControl$$SetTasks` at VA `0x105CB4D0`).

3. **Ghidra 11.2** — loads `GameAssembly.dll` (the actual compiled binary). After running `ghidra_with_struct.py` to apply all function labels, functions are navigated by VA address. The decompiler panel produces readable C pseudocode.

The investigation followed the call chain from `GameStartManager$$BeginGame` down through `ShipStatus$$Begin` and `ShipStatus$$AddTasksFromList` to reach the core distribution logic.

---

## Key Functions Found

| Function | Location | Purpose |
|---|---|---|
| `GameStartManager$$BeginGame` | `GameStartManager` class | Entry point, delegates to `ReallyBegin` after player count checks |
| `GameStartManager$$ReallyBegin` | `GameStartManager` class | UI cleanup + countdown timer, calls nothing task-related |
| `ShipStatus$$Begin` | `ShipStatus` class | **Core distribution function** — builds and assigns all task lists |
| `ShipStatus$$AddTasksFromList` | `ShipStatus` class | Draws tasks from pool for one player, handles wrap-around |
| `NetworkedPlayerInfo$$RpcSetTasks` | `NetworkedPlayerInfo` class | Host broadcasts completed task list to all clients |
| `NetworkedPlayerInfo$$SetTasks` | `NetworkedPlayerInfo` class | Deserializes incoming byte array into `TaskInfo` objects |
| `PlayerControl$$SetTasks` | `PlayerControl` class | Applies a task list to a player via coroutine |
| `NormalPlayerTask$$PickRandomConsoles` | `NormalPlayerTask` class | Separately shuffles which physical terminal a task points to |

---

## The Algorithm (Step by Step)

### Phase 1 — Common Tasks
Before the player loop, `ShipStatus$$Begin` calls `AddTasksFromList` using `RandomIdx` + `RemoveAt` to pick common tasks from the full task pool. These are identical for every player and are added to the byte list first, remaining as the base for all players.

### Phase 2 — Pool Preparation
```
Extensions$$Shuffle(longTaskPool)   // shuffled ONCE before player loop
Extensions$$Shuffle(shortTaskPool)  // shuffled ONCE before player loop
```
Both the long and short task pools are shuffled **a single time** before any player is assigned tasks. A shared index pointer (`*param_3`) is initialized to 0 for each pool.

### Phase 3 — Per-Player Loop
For each player (crewmates **and** impostors):
```
1. Reset byte list back to common tasks only (RemoveRange)
2. Call AddTasksFromList(longPool, sharedLongIndex, longTaskCount)
3. Call AddTasksFromList(shortPool, sharedShortIndex, shortTaskCount)
4. Convert byte list to array
5. Call RpcSetTasks(player, taskArray) — broadcasts to all clients
```

### Phase 4 — Pool Exhaustion (Inside AddTasksFromList)
```
if (sharedIndex >= pool.Count):
    sharedIndex = 0
    Extensions$$Shuffle(pool)        // pool reshuffled on wrap-around
    if (ALL task types already used):
        HashSet.Clear()              // type-deduplication reset
```

### Phase 5 — Per-Player Type Deduplication
```
if (HashSet.Add(task.TaskType) == false):
    skip — don't count toward goal (same type already given to this player)
else:
    byteList.Add(task.ID)           // valid task, added to this player's list
```
The `HashSet<TaskTypes>` prevents any single player from receiving the same task **type** twice. It is cleared between players via `HashSet.Clear()` in `ShipStatus$$Begin`'s loop.

---

## Code Snippets

### `ShipStatus$$Begin` — Pool Shuffle + Player Loop (Core Finding)
```c
// Pools shuffled ONCE before player loop begins
Extensions$$Shuffle(longTaskPool,  Method$Extensions.Shuffle<NormalPlayerTask>());
Extensions$$Shuffle(shortTaskPool, Method$Extensions.Shuffle<NormalPlayerTask>());

// Per-player loop (includes impostors)
for (uVar7 = 0; uVar7 < playerList.Count; uVar7++) {
    HashSet.Clear();
    List.RemoveRange(byteList, commonTaskCount, byteList.Count - commonTaskCount);
    ShipStatus$$AddTasksFromList(ship, &longIndex,  longTaskCount,  byteList, usedTypes, longTaskPool);
    ShipStatus$$AddTasksFromList(ship, &shortIndex, shortTaskCount, byteList, usedTypes, shortTaskPool);
    
    player = playerList[uVar7];
    if (DummyBehaviour.enabled == false) {  // false = real player, not a bot
        taskArray = byteList.ToArray();
        NetworkedPlayerInfo$$RpcSetTasks(player, taskArray);
    }
}
```

### `ShipStatus$$AddTasksFromList` — Wrap-Around + Deduplication
```c
while (tasksAssigned < targetCount) {
    if (iVar5 == 1000) return;  // infinite loop guard

    // Wrap-around: pool exhausted, reshuffle
    if (*sharedIndex >= pool.Count) {
        *sharedIndex = 0;
        Extensions$$Shuffle(pool);
        if (Enumerable.All(pool, alreadyUsedCheck)) {
            HashSet.Clear();  // all types used, reset deduplication
        }
    }

    task = pool[*sharedIndex];
    *sharedIndex += 1;

    // Per-player type deduplication via HashSet
    if (HashSet.Add(task.TaskType) == false) {
        tasksAssigned--;  // duplicate type, don't count
    } else {
        byteList.Add(task.ID);  // valid, add to player's list
    }

    tasksAssigned++;
    iVar5++;
}
```

### `NetworkedPlayerInfo$$RpcSetTasks` — Host-Side Broadcast Proof
```c
void NetworkedPlayerInfo$$RpcSetTasks(player, taskArray) {
    if (AmongUsClient.IsHost()) {
        NetworkedPlayerInfo$$SetTasks(player, taskArray);  // apply locally
    }
    message = new RpcSetTasksMessage(player.PlayerId, taskArray);
    AmongUsClient.LateBroadcastReliableMessage(message);  // send to all clients
}
```

### `NormalPlayerTask$$PickRandomConsoles` — Console Location Shuffle
```c
// NOTE: This is a SEPARATE shuffle from task type assignment.
// Determines which physical terminal on the map a task points to.
consoles = ShipStatus.AllConsoles
    .Where(c => c.ValidForTask(this))
    .ToList();
Extensions$$Shuffle(consoles);  // shuffled independently per task instance
return consoles;
```

---

## Implications

### Tasks Are Not Independently Random Per Player
The pre-shuffle before the player loop means all players draw from the **same ordered sequence**. Player 1 gets pool positions 0–1, player 2 gets positions 2–3, and so on. This is a soft-balanced distribution, not independent random assignment.

### Impostors Consume Pool Positions
Impostors are included in the player loop and draw from the same pools as crewmates. Their fake task lists are **real task assignments** that the game flags as non-completable. This means in a 10-player lobby, all 10 players (including impostors) consume pool positions.

### Polus 10-Player Example (2-2-2 settings, 2 impostors)
- **10 players × 2 long tasks = 20 draws** from a pool of 7
  - Pool wraps ~2.86 times → 6 long tasks assigned to **3 players**, 1 long task assigned to **2 players**
- **10 players × 2 short tasks = 20 draws** from a pool of 12
  - Pool wraps ~1.67 times → 8 short tasks assigned to **2 players**, 4 short tasks assigned to **1 player**

### Task Overlap Is Predictable
Because the pool is shuffled once and drawn sequentially, players who join the lobby adjacent to each other will tend to have **different** tasks, while players at wrap-around boundaries may share tasks. Task overlap is more common with larger player counts and smaller task pools.

### Strategic Implication for Impostors
When an impostor claims a long task on Polus in a 10-player game, that task will genuinely be on their fake task list. However, because most long tasks are shared by exactly 3 players, a crewmate can use task overlap as a soft alibi signal — if 3 people already confirmed a task, a 4th claim is **mathematically impossible** without a pool re-evaluation.

---

## Limitations

- This analysis was performed on **Among Us version timestamped 3/17/2026**. Innersloth could change the algorithm in future updates.
- The `AddTasksFromList` function contains a **1000-iteration guard** whose exact trigger conditions were not fully traced.
- Common task assignment was not fully traced — only confirmed they are pre-built before the player loop.
- Analysis is client-side only. Any server-side validation logic is not visible from `GameAssembly.dll`.

---

## Tools & References

- [Il2CppDumper](https://github.com/Perfare/Il2CppDumper)
- [dnSpy](https://github.com/dnSpy/dnSpy)
- [Ghidra](https://github.com/NationalSecurityAgency/ghidra)
- [Among Us Wiki — Tasks](https://among-us.fandom.com/wiki/Tasks)
