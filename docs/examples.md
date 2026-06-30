# Real Prompts from RACE v2.0

These are real Engineer-to-Coder prompts used during the development of [RACE v2.0 — AI Rescue Tournament](https://github.com/PyAimind/RACE-AI-Rescue-Tournament-v2), lightly cleaned up for formatting only — the technical content and structure are unchanged from what was actually sent.

They show the methodology applied in practice: strict line limits, hallucination control through an explicit allowed-library list, no unsolicited structural changes during bug fixes, and incremental integration of a new feature (shared grid) into an existing system without touching unrelated logic (pretraining, scoring, Q-Learning).

## 1. `brain.py` — Introducing the Q-Learning agent

This prompt replaced an earlier random-based decision system with a proper Q-Learning agent, while keeping the BFS pathfinding interface compatible with the rest of the system.

```markdown
Rewrite the brain.py module. Replace the old random-based logic and BFS with a lightweight Q-Learning agent. Use ONLY Python standard library (json, random, os, collections). No numpy, no sklearn, no external packages.

### Class: BrainAgent

1. **__init__(self, map_grid, targets, alpha=0.1, gamma=0.9, epsilon=0.1)**
   - Store map_grid, targets, alpha, gamma, epsilon.
   - Initialize self.q_table as a nested dictionary: {state: {"UP": 0.0, "DOWN": 0.0, "LEFT": 0.0, "RIGHT": 0.0}}.
   - self.last_state = None, self.last_action = None.

2. **get_state(self, worker_pos) -> tuple**
   - Find the nearest target from self.targets using Manhattan distance.
   - Compute direction vector from worker_pos to that target:
     dir_x = 1 if target.x > worker.x else -1 if target.x < worker.x else 0.
     dir_y = 1 if target.y > worker.y else -1 if target.y < worker.y else 0.
   - Return (worker.x, worker.y, dir_x, dir_y).

3. **choose_action(self, state) -> str**
   - Epsilon-greedy:
     - If random.random() < self.epsilon, return a random choice from ["UP", "DOWN", "LEFT", "RIGHT"].
     - Else, return the key with the highest value in self.q_table[state]. If state not in q_table, initialize it with zeros and pick random.
   - Store state as self.last_state and the chosen action as self.last_action.

4. **update_q_value(self, reward, next_state) -> None**
   - Ensure self.q_table has entries for both self.last_state and next_state (initialize with zeros if missing).
   - old_value = self.q_table[self.last_state][self.last_action]
   - next_max = max(self.q_table[next_state].values())
   - new_value = old_value + self.alpha * (reward + self.gamma * next_max - old_value)
   - Update self.q_table[self.last_state][self.last_action] = new_value

5. **save_q_table(self, filepath) -> None** (optional but nice)
   - Convert state tuples to strings and save as JSON.
6. **load_q_table(self, filepath) -> None** (optional)
   - Load from JSON and convert keys back to tuples.

### Important Rules:
- All methods must be under 20 lines.
- brain.py must be under 150 lines total.
- Deliver ONLY raw Python code. No comments, no markdown, no explanations.
- Do NOT import anything outside standard library.
- Do NOT break the existing interface: choose_action must return a direction string ("UP", "DOWN", "LEFT", "RIGHT") that can be directly used by Mediator.
```

**Rules demonstrated:** #2 (size limits), #9 (hallucination control via explicit allowed-library list), #10 (interface preserved — `choose_action` must keep returning a direction string the Mediator already expects).

## 2. `worker.py` — A surgical bug-fix prompt

This prompt changed the return signature of `move()` without touching any other logic in the class — a direct application of "no structural changes during bug fixes."

```markdown
Modify the move method in worker.py. The class WorkerAgent must remain otherwise unchanged. All existing logic for wait_time, speed_factor, on_fire, in_smoke, respawn on fire, and rubble slow-down must stay exactly the same.

### Required change in move(self, direction, brain_allow=False) method:
- Currently returns a tuple (new_x, new_y) or current position.
- Change the return value to a tuple of TWO items: ((new_x, new_y), event_string)
- The event_string must be one of:
  - "moved": Worker successfully moved to a free cell (0) or target (4) or smoke (3) (normal movement).
  - "hit_rubble": Worker moved onto a rubble cell (1) and the move was allowed (not blocked).
  - "hit_fire": Worker moved onto a fire cell (2) and brain_allow was True, causing respawn at (0,0).
  - "rescued": Worker moved onto a target cell (4) (the target event; note: if target is step on, we treat as rescued).
  - "blocked": Movement was prevented due to: out of bounds, fire without brain_allow, or any other condition where the worker stays in place (e.g., still in waiting state but wait_time decreased to 0? Wait, wait_time decrements and returns current position; that should also be "blocked" or maybe "waiting"? We'll treat as "blocked" for simplicity, because no movement occurred.)
- The logic for determining the event:
  - If wait_time > 0: decrement wait_time, return (current position, "blocked").
  - Calculate new coordinates; if out of bounds, return (current position, "blocked").
  - cell = get_cell(map_grid, new_x, new_y)
  - If cell == 2 and not brain_allow: return (current position, "blocked").
  - If cell == 2 and brain_allow: perform respawn to (0,0) and reset statuses, return ((0,0), "hit_fire").
  - If cell == 1 (rubble): update position and set speed_factor=0.5, return (new_pos, "hit_rubble").
  - If cell == 3 (smoke): update position and set in_smoke, etc., return (new_pos, "moved").
  - If cell == 4 (target): update position, set is_on_target true, return (new_pos, "rescued").
  - If cell == 0 (free): update position, reset statuses, return (new_pos, "moved").
- All other methods (get_position, get_cell_type, is_on_target, get_status) remain exactly the same.
- Import get_cell from map_loader as before.
- Keep file under 100 lines.

### Deliver ONLY raw Python code. No comments, no markdown.
```

**Rules demonstrated:** #1 (every edge case enumerated — even the Engineer's own uncertainty about the "blocked vs waiting" case is resolved explicitly rather than left ambiguous), #10 (everything outside `move()` is frozen).

## 3. `race_manager.py` — Incremental feature integration

This prompt added shared-grid competition (both teams racing on the same map) to an already-working system, without touching pretraining, scoring, or Q-Learning — and explicitly listed which downstream file (`gui.py`) needed a corresponding small change.

```markdown
Rewrite race_manager.py with the following critical fixes. Use only standard library (json, random, os, copy, threading). Keep all existing logic for pretraining, save/load Q-tables, and scoring.

### 1. Shared Grid Management
- In __init__, load map JSON and create `self.shared_grid` using `deepcopy` of the map's grid.
- For each team being created, pass `map_grid=self.shared_grid` (the same reference) instead of a deep copy. This ensures all teams see the same grid.
- Add a method `_sync_target_removal(self, pos)`:
    - Set `self.shared_grid[pos[0]][pos[1]] = 0`.
    - For each team in `self.teams`, if `pos` in `team.brain.targets`, remove it.

### 2. Step-by-Step Turn Method
- Add a method `step_turn(self)` that performs one turn for the current team and handles everything:
    - Get current team.
    - If not `team.has_won()`:
        - Call `result = team.step()`.
        - If `result == "rescued"`:
            - Call `self._sync_target_removal(team.get_position())`.
        - Increment `self.total_turns`.
    - Move to next team: `self.current_turn = (self.current_turn + 1) % len(self.teams)`.
    - Return `True` if `self.all_targets_rescued()` else `False`.
- The old `run_race` method can now just call `step_turn()` in a loop until it returns True, then calculate winner and award tokens. But keep `run_race` as is for backward compatibility.

### 3. Termination Check
- Add method `all_targets_rescued(self) -> bool`:
    - Loop over every cell in `self.shared_grid`; if any equals 4, return False. Otherwise True.

### 4. Adjust GUI integration
- In gui.py, modify `next_turn` to:
    - Call `self.race.step_turn()` instead of directly calling `team.step()`.
    - After the call, check `self.race.all_targets_rescued()` to decide if race finished.
- Remove any manual grid updates in gui.py; `step_turn` handles everything internally.

### 5. Keep everything else unchanged (pretraining, save/load, BFS, Q-Learning, scoring).
### Deliver ONLY raw Python code for race_manager.py. No comments.
```

**Rules demonstrated:** #6 (incremental integration — a new capability layered onto a working system), #9 (standard-library-only constraint kept consistent across the whole project), #10 (pretraining/scoring/Q-Learning explicitly frozen).
