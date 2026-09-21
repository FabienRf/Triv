# Prompt — Diet Tracker feature (Flutter, local-only, Android)

> Paste this whole document into your AI assistant to generate the feature.
> Everything must match the app's existing theme, naming conventions, state management and storage patterns.

---

## 1. Context

- Platform: **Flutter, Android device, 100% local. No backend, no account, no internet needed.**
- Existing features in the app: sleep tracking, weight tracking, habits.
- The diet tracker is a **new tab/section** of the app, accessible from the main navigation, next to the existing trackers.
- All data lives on-device. Data must survive app restarts.

## 2. Local database (recommended)

Use **sqflite (SQLite)** — the standard local DB for Flutter/Android.

- Mature, well-maintained, official Android/iOS support.
- SQL is perfect for relational data (foods -> meals -> daily diary), which this feature is.
- Identical stack to the rest of this codebase if you already use sqflite.

**Rationale vs alternatives** (only if needed):
- `drift`: nice type-safe ORM over SQLite, but a bigger dependency + codegen.
- `Hive`: fast key/value, fine for tiny data but awkward for queries and relations.
- `Isar`: fast NoSQL but the main package has maintenance gaps.
- -> `sqflite` is the safe, consistent pick for a tracking app.

### Suggested schema

```sql
-- Reusable food database (created once, reused across days)
CREATE TABLE foods (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  calories REAL NOT NULL,   -- per serving (see serving_size)
  protein REAL NOT NULL,
  fat REAL NOT NULL,
  carbs REAL NOT NULL,
  serving_size TEXT,        -- e.g. "1 bowl", "100 g", "1 slice"
  created_at TEXT NOT NULL
);

-- One row per meal eaten on a date
CREATE TABLE meals (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  date TEXT NOT NULL,        -- 'YYYY-MM-DD'
  type TEXT NOT NULL,        -- breakfast | lunch | dinner | snack
  time TEXT,                 -- 'HH:mm', defaults to now
  note TEXT
);

-- Items composing a meal (M->N food/meal, with quantity)
CREATE TABLE meal_items (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  meal_id INTEGER NOT NULL REFERENCES meals(id) ON DELETE CASCADE,
  food_id INTEGER NOT NULL REFERENCES foods(id),
  quantity REAL NOT NULL DEFAULT 1
);

-- Daily goals (single row, user-editable)
CREATE TABLE daily_goals (
  id INTEGER PRIMARY KEY CHECK (id = 1),
  calories REAL NOT NULL DEFAULT 2000,
  protein REAL NOT NULL DEFAULT 120,
  fat REAL NOT NULL DEFAULT 70,
  carbs REAL NOT NULL DEFAULT 250
);
```

- Follow the codebase's existing `AppDatabase` pattern (singleton, `onCreate`, migration version bump).
- Macros are always **grams**.

## 3. Screen 1 — Diet Tracker (main view)

### 3.1 Top: week strip (date selector)
- Horizontal row, **Monday -> Sunday**, each cell showing:
  - Day letter/number (e.g. "M", "14" or "Mon").
  - **Calories total of that day below the day label** (small text).
- Every day cell is tappable and selects that day; the selected day is highlighted.
- The current day is visually distinct (different color/border).
- Left/right arrows let you jump **to previous/next week** (and back to "today").

### 3.2 Selected day summary (just under the week strip)
- On the main day row, show the selected day's date (and a "Today" label if relevant).
- Show **consumed / goal calories in the format `(x / y)`**, e.g. `(1240 / 2000)`.
- Just below: **one progress bar per macro**, each showing icon + label + `x / y`:
  - fire icon -> Calories (kcal)
  - `P` -> Protein (grams)
  - `F` -> Fat (grams)
  - `C` -> Carbs (grams)
- Each bar displays the current filled portion vs goal; the bar color changes when over the goal.

### 3.3 Meals of the day
- List of meals the user already logged for the selected day, chronologically.
- Each meal card shows:
  - Meal type (breakfast/lunch/dinner/snack) + time.
  - Total kcal of the meal, plus its P / F / C totals in a small row.
  - The food items under it (name + quantity + kcal).
- Tap a meal to **edit**, with a delete affordance + confirmation dialog.
- Empty day -> hint: "No meals yet — tap + to add one".

### 3.4 Floating action button
- **Bottom-right FAB (add icon)** -> opens the "Add meal / edit meal" screen, preselected for the currently selected day.
- On return, tracker totals and week strip must refresh.

## 4. Screen 2 — Add / edit meal

### 4.1 Header
- Back arrow + title "Add meal" / "Edit meal".
- Date shown (defaults to the day you came from), tappable to change.

### 4.2 Meal type
- Segmented row of 4 options: **Breakfast, Lunch, Dinner, Snack** (icons optional, match app style).
- Time field, defaults to current time (editable).

### 4.3 Food items (main body)
- Search field at top: filters the saved foods list by name.
- List of all saved `foods` (reusable favorites) — tap a food to add it to the meal.
- Quantity stepper (`- qty +`) on already-added items, plus remove.
- **Add new food** quick action: inline form "New food" with fields name, serving size, calories, protein, fat, carbs — saved into `foods` for future reuse.

### 4.4 Live meal totals
- Bottom summary bar while building the meal: current kcal + P + F + C totals for this meal, same `(x / y)` feel as the tracker.

### 4.5 Save
- Validate: at least one item.
- Save inserts/updates `meals` + `meal_items`, recomputes the day's totals, then pops back to the tracker.

## 5. Cross-cutting requirements

- **Daily goals** must be editable somewhere (e.g. settings/section header) with sensible defaults; the tracker reads the single `daily_goals` row.
- Totals are derived (sum of meal_items) and recomputed on any change.
- Week strip totals and selected-day summary update live after any add/edit/delete.
- Follow the app's theme (colors, typography, spacing), reuse existing widgets where possible.
- State management consistent with the rest of the app.

## 6. Acceptance checklist

- [ ] Week strip shows Mon -> Sun with kcal per day.
- [ ] Selecting a day updates summary + meals list.
- [ ] Calories shown as `(x / y)` with progress bars for calories, protein, fat, carbs.
- [ ] FAB opens add-meal screen preselected on the chosen day.
- [ ] Meal can be saved with multiple foods and quantities.
- [ ] Meals are editable and deletable.
- [ ] Totals recompute everywhere after any change.
- [ ] Everything persists locally across app restarts (SQLite).