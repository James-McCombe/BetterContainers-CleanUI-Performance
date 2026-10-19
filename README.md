# Better Containers + CleanUI — inventory sorting profiler

Project Zomboid Build 42.21, tested October 18, 2026.

## Problem
With both mods enabled, an additional `table.sort(self.itemslist, ...)` in Better Containers' `finishRowCategoryRefresh()` occurs after the inventory pane's refresh. CleanUI already sorts matching search results first and applies selected-column sorting. The additional pass correlates with severe frame stutters around large inventories.

## Reproduction
Use the included `InventoryPaneRender.lua` as an instrumented **testing copy** of Better Containers' 42.21 `media/lua/client/BetterContainers/Categorize/InventoryPaneRender.lua`. It includes the author's sorting block with a toggle:

```lua
local skipCleanUISort = true
```

- `false`: execute Better Containers' original extra sort.
- `true`: skip that sort when CleanUI is detected.

Restart the game between runs. Open, search, unload and reload inventories in a densely stocked area (the tests used two fridges and twelve containers). The code logs `[BC Profiler]` counters roughly every five seconds of refresh activity. Logs in `logs/` contain only these profiler lines, not private machine paths or unrelated game diagnostics.

## Results
| Metric | Original additional sort | Skip with CleanUI |
|---|---:|---:|
| Refresh calls | 1,251 | 735 |
| Rows processed | 87,890 | 60,383 |
| Largest list | 799 | 799 |
| Category processing | 247 ms | 172 ms |
| Additional sorting-block time | 66,323 ms | 10 ms |
| Average sorting-block time per refresh | 53.0 ms | 0.014 ms |
| In-game observation | Severe spikes around storage | No noticeable spikes in similar storage-area test |

These sessions are similar, not identical controlled workloads. Timing includes comparator callbacks and conditional overhead, **not only** the Lua sorting primitive. Inventory rows are not necessarily individual items. The enabled-skip measurements include timing overhead and should not be interpreted as ten milliseconds of actual sorting.

## Proposed compatibility behaviour
Keep assigning resolved `row.cat` categories, but when CleanUI is active let it own the final inventory ordering:

```lua
if not Helpers.isCleanUIActive() then
    table.sort(self.itemslist, self.itemSortFunc)
end
```

The profiler file deliberately retains both branches to facilitate independent testing. Additional validation of Better Containers' category sorting, pinning, and equipped-item grouping is recommended before adoption upstream.

The copy is for diagnostic reproduction, **not** intended as a maintained fork of the original mod.
