# Spec: Tool Functions

**File:** `tools.py`
**Status:** `get_seasonal_conditions` — Pre-implemented, read through. `lookup_plant` — complete spec fields before implementing.

---

## Purpose

These two functions are the tools the agent can call. They retrieve structured data from the local plant database and seasonal data files and return it to the agent loop, which passes it to the LLM as context for generating a response.

---

## Function 1: `lookup_plant()`

### Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `plant_name` | `str` | The plant name as entered by the user or chosen by the LLM — may be any casing, common name, scientific name, or alias |

**Output:** `dict`

When the plant is **found**, return:
```python
{"found": True, "plant": <the full plant dict from _plant_db>}
```

When the plant is **not found**, return:
```python
{"found": False, "name": <normalized input>, "message": <helpful string>}
```

---

### Design Decisions

*Complete the two blank fields below before writing code. The others are pre-filled for you.*

---

#### Input normalization

Strip leading/trailing whitespace and convert to lowercase before any comparison.

```python
normalized = plant_name.strip().lower()
```

---

#### Search order

Search in this order: direct key → display name → aliases. Keys are the fastest
lookup (O(1) dict access), so check those first. Display names are the next most
likely match for clean user input. Aliases are the broadest net, so they go last.

```
1. Direct key match: normalized in _plant_db
2. Display name match: plant["display_name"].lower() == normalized
3. Alias match: normalized in [alias.lower() for alias in plant["aliases"]]
```

---

#### Alias matching approach

*Aliases are stored as a list of strings. How will you check if the normalized input matches any alias in the list? Write your approach in pseudocode or plain English.*

```
Instead of scanning each plant's alias list on every lookup, I preprocess
_plant_db once into an inverted index: a flat dict that maps every searchable
name -> the plant's slug.

Build it once at module load (the same place _plant_db is loaded), by looping
over every plant and inserting three kinds of entries, all pointing to that
plant's slug:
    - the slug itself              ("pothos"        -> "pothos")
    - the lowercased display_name  ("pothos"        -> "pothos")
    - each lowercased alias        ("devil's ivy"   -> "pothos")

  _name_index = {}
  for slug, plant in _plant_db.items():
      _name_index[slug] = slug
      _name_index[plant["display_name"].lower()] = slug
      for alias in plant["aliases"]:
          _name_index[alias.lower()] = slug

Then key / display / alias matching all collapse into a single membership test:

  if normalized in _name_index:
      slug = _name_index[normalized]
      return {"found": True, "plant": _plant_db[slug]}

Note: with this design the "key -> display -> alias" search order becomes a
BUILD/precedence order, not a runtime scan order. It only matters if two plants
ever share a name — whichever entry is inserted last wins — so insert in a
deliberate order.
```

---

#### Not-found message

*When a plant isn't found, the agent will read your message and use it to decide what to tell the user. Write the exact string you'll return — make it useful to the agent, not just to a human reading logs.*

```python
message = f"'{plant_name}' was not found as a valid plant in our records."
```

The string names the exact input that failed (interpolated with an f-string),
so the agent can tell the user which name didn't match — more useful than a
fixed string. The `"name"` field also carries the normalized input separately.

---

#### Implementation Notes

*Fill this in after implementing and running the app.*

**Test: does `"devil's ivy"` return the pothos entry?**
```
yes — matches via the alias entry in the index, returns the Pothos plant dict.
```

**Test: does `"SNAKE PLANT"` return the snake plant entry?**
```
yes — lowercased to "snake plant", matches the display_name entry, returns Snake Plant.
```

**One edge case you discovered while implementing:**
```
The slug keys contain underscores (e.g. "snake_plant"), but a user types
"snake plant" with a space. With plain .strip().lower() normalization, the
underscore slug entry ("snake_plant") never matches spaced user input — that
input is instead caught by the display_name entry ("Snake Plant" -> "snake
plant"). So the slug entries in the index are effectively redundant for normal
typing; they only matter if the LLM passes the literal slug. Harmless, but a
reminder that the three index entries don't all pull equal weight under this
normalization.
```

---

## Function 2: `get_seasonal_conditions()`

### Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `season` | `str \| None` | One of `"spring"`, `"summer"`, `"fall"`, `"winter"`, or `None` to auto-detect |

**Output:** `dict`

The full season dict from `_season_data`, plus one additional field:

| Added field | Type | Value |
|-------------|------|-------|
| `"detected_season"` | `bool` | `True` if auto-detected from the month; `False` if season was passed as an argument |

---

### Design Decisions

*This function is pre-implemented — read through these fields and the code before working on `lookup_plant`.*

---

#### Auto-detection logic

When `season` is `None`, get the current calendar month with `datetime.now().month`
and look it up in the `_MONTH_TO_SEASON` dict, which maps month numbers to season strings.

```python
current_month = datetime.now().month
season_key = _MONTH_TO_SEASON[current_month]
```

---

#### Season validation

If the caller passes an invalid season string (e.g., `"monsoon"`), the function
falls back to auto-detection — same as if `None` were passed. The `VALID_SEASONS`
set acts as the gate:

```python
VALID_SEASONS = {"spring", "summer", "fall", "winter"}
if season and season.lower() in VALID_SEASONS:
    ...  # use provided season
else:
    ...  # auto-detect
```

---

#### Return structure

The full season dict from `_season_data`, plus a `detected_season` boolean. Example for spring:

```python
{
    "season": "spring",
    "watering": "Increase watering frequency as plants break dormancy ...",
    "fertilizing": "Resume feeding with a balanced fertilizer ...",
    "light": "Days are lengthening — move plants closer to windows ...",
    "pests": "Watch for spider mites and aphids as temperatures rise ...",
    "detected_season": True   # True = auto-detected; False = caller specified
}
```

---

#### Implementation Notes

*Fill this in after testing.*

**Test: does calling with `season=None` return the correct season for the current month?**
```
Current month: [month]
Expected season: [season]
Returned season: [season]
```

**Test: does calling with `season="winter"` return winter data regardless of the current month?**
```
[yes / no]
```
