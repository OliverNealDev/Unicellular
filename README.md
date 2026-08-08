# Unicellular

An idle simulation built around sheer entity count: thousands of single-celled organisms wandering, interacting and multiplying, kept smooth by finite state machines and careful performance work.

[Play on itch.io](https://olivernealdev.itch.io/unicellular) · [Full technical breakdown](https://oliverneal.dev/unicellular.html) · [Portfolio](https://oliverneal.dev)

| | |
|---|---|
| **Role** | Solo developer |
| **Engine** | Unity 6 · C# |
| **Released** | February 2025 |
| **Entities** | Stress-tested at 1,000 and 10,000 |
| **Core tech** | Multi-rate FSM tick with time-sliced proximity AI |

---

## Overview

Unicellular is an idle sim: you spend currency to buy unicells, they drift around a petri dish living their tiny lives, and the colony's activity feeds the economy back to you. The design means the player is **directly incentivised to push the entity count up**, so each cell has to stay cheap to simulate or the game's own core loop kills its framerate.

That constraint shaped everything. Behaviour is driven by lightweight finite state machines, but the real trick is **when** work runs: instead of every cell thinking every frame, the simulation is split across several tick rates and the expensive proximity checks are spread out a batch at a time, all inside Unity's standard GameObject workflow.

On top of the simulation sits a full idle-game economy: seven species of cell, layered upgrades, rare Elder and Shiny variants, a per-species "Souls" prestige currency, and JSON saves so a colony survives between sessions.

---

## Key systems

### One FSM per cell

Every unicell runs a small finite state machine that decides what it is doing from moment to moment: drifting, seeking, interacting with neighbours. Keeping each state tiny and the transitions explicit does two jobs at once. Behaviour stays easy to reason about and extend, and the cost of "thinking" each frame stays roughly constant no matter how big the colony gets.

Emergence does the rest. With thousands of simple machines running side by side, the dish reads as alive without any cell doing anything complicated.

### A decoupled, multi-rate loop

Core behaviour ticks at **20 Hz**, slower bookkeeping like population and levelling runs at **0.5 Hz**, and physics work sits in its own pass, so nothing runs more often than it needs to. The two rates are just two independent accumulators:

```csharp
// Assets/Scripts/Managers/UnicellManager/EntityManager.cs
if (LogicUpdateTimer > LogicUpdateTimerThreshold)
{
    LogicUpdateTimer -= LogicUpdateTimerThreshold;
    LogicUpdate();
}

SlowLogicUpdateTimer += Time.deltaTime;
if (SlowLogicUpdateTimer > SlowLogicUpdateTimerThreshold)
{
    SlowLogicUpdateTimer -= SlowLogicUpdateTimerThreshold;
    isProximityUpdateComplete = false;
    ProximityBatchUpdate();
    SlowLogicUpdate();
    CheckUnicellsForHunger();
}
```

### Time-sliced proximity AI

The expensive "what is near me" queries are the bottleneck, so they are amortised across frames: roughly **150 cells are processed per pass**, carrying an index forward until the whole colony has been swept. The slicing falls out of one flag. When the slow tick fires it clears `isProximityUpdateComplete` and starts a batch; every `FixedUpdate` after that continues the sweep until the colony has been covered, so the cost of a proximity pass is spread over however many frames it needs rather than landing in one.

```csharp
void FixedUpdate()
{
    if (!isProximityUpdateComplete) ProximityBatchUpdate();
}

void ProximityBatchUpdate()
{
    // Determines end index for this batch
    int endIndex = Mathf.Min(currentIndex + ProximityBatchSize, unicellList.Count);
    // ...
}
```

### One manager, not thousands of updates

A single simulation manager iterates the cell and food lists directly instead of every cell running its own per-frame `Update`. That is what makes ten thousand entities survivable in a GameObject workflow: the benchmark scene holds **10,000 cells at around 90 FPS**, and that headroom is what lets the idle economy keep rewarding a bigger colony.

---

## What I would rebuild

Everything above keeps a GameObject workflow fast by working around its costs: batching the ticks, slicing the proximity queries, collapsing thousands of updates into one manager. Those are the right moves inside that architecture, but they are still workarounds.

If I built this again I would use **Unity's Entity Component System**. Laying the data out contiguously and letting jobs run over it is what this simulation actually wants, and it should take the entity ceiling well past ten thousand rather than making ten thousand survivable. That rebuild is the reason for this repository's name; the code here is still the GameObject implementation that shipped.

---

## Project structure

```
Assets/Scripts/
  Managers/UnicellManager/EntityManager.cs   Multi-rate tick, time-sliced proximity sweep
  Managers/EconomyManager.cs                 Currency, upgrades, per-species Souls prestige
  Managers/GameManager.cs                    Save and load, run state
  Managers/UIManager.cs, TextManager.cs      Idle-game UI and number formatting
  ChildClasses/                              The seven cell species and their behaviour overrides
  Classes/EconomyData.cs                     Serialised economy state for JSON saves
```

---

## Running it

```bash
git clone https://github.com/OliverNealDev/Unicellular.git
```

Open the project in **Unity 6 (6000.0.45f1 or newer)** and load the main scene from `Assets/Scenes`.

Or skip the editor and [play it in the browser on itch.io](https://olivernealdev.itch.io/unicellular).

---

## Author

**Oliver Neal**, gameplay programmer specialising in Unity and C#.

[oliverneal.dev](https://oliverneal.dev) · [itch.io](https://olivernealdev.itch.io) · [LinkedIn](https://www.linkedin.com/in/oliverjackneal/) · [GitHub](https://github.com/OliverNealDev)
