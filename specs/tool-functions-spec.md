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

Search in this order: direct key → display name → scientific name → aliases.
Keys are the fastest lookup (O(1) dict access), so check those first. Display
names are the next most likely match for clean user input. Scientific names
matter because the tool definition tells the LLM it can pass them. Aliases are
the broadest net, so they go last.

```
1. Direct key match: normalized in _plant_db
2. Display name match: plant["display_name"].lower() == normalized
3. Scientific name match: plant["scientific_name"].lower() == normalized
4. Alias match: normalized in [alias.lower() for alias in plant["aliases"]]
```

---

#### Alias matching approach

*Aliases are stored as a list of strings. How will you check if the normalized input matches any alias in the list? Write your approach in pseudocode or plain English.*

```
Loop over every plant in _plant_db. For each one, lowercase every alias in its
"aliases" list and test membership with `in`:

    normalized in [alias.lower() for alias in plant["aliases"]]

The input is already normalized (stripped + lowercased) once up front, so each
alias only needs to be lowercased for the comparison. This is O(plants × aliases),
which is fine for 15 plants.

If the database grew to thousands of plants, I'd precompute a single lookup index
once at module load — a dict mapping every normalized name (key, display name,
scientific name, and each alias) to its plant slug. Then each lookup is one O(1)
dict access instead of scanning every plant and rebuilding the lowercased alias
list on every call.
```

---

#### Not-found message

*When a plant isn't found, the agent will read your message and use it to decide what to tell the user. Write the exact string you'll return — make it useful to the agent, not just to a human reading logs.*

```
"No plant matching '<normalized>' was found in the database. The database
currently covers these plants: Pothos, Snake Plant, ZZ Plant, Spider Plant,
Peace Lily, Aloe Vera, Monstera, Rubber Plant, Boston Fern, Orchid, Fiddle Leaf
Fig, Calathea, Philodendron, Chinese Evergreen, Succulent. Suggest the closest
match if one of these is likely what the user meant, or let the user know this
plant isn't covered."

The list of known plants is built dynamically from _plant_db so it never drifts
out of sync with the data. A bare "not found" gives the agent nothing to act on;
naming the covered plants lets the LLM either suggest a close alternative (e.g.
"dragon tree" → maybe they want a similar care guide) or honestly tell the user
the plant isn't supported, instead of hallucinating care advice.
```

---

#### Implementation Notes

*Fill this in after implementing and running the app.*

**Test: does `"devil's ivy"` return the pothos entry?**
```
Yes — matches via the alias list and returns {"found": True, "plant": <Pothos>}.
```

**Test: does `"SNAKE PLANT"` return the snake plant entry?**
```
Yes — normalization lowercases the input to "snake plant", which matches the
display name "Snake Plant".lower(). " pothos " (whitespace) and
"Dracaena trifasciata" (scientific name) also resolve correctly.
```

**One edge case you discovered while implementing:**
```
The "name" field in the not-found response should be the *normalized* input,
not the raw input. Returning the normalized value keeps the response consistent
with what was actually searched and avoids leaking stray casing/whitespace back
to the agent. Also worth noting: several plants share overlap-prone aliases
(e.g. "hen and chickens plant" vs. succulent's "hen and chicks") — exact-match
comparison avoids false positives that substring matching would cause.
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
    "name": "Spring",
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
Current month: June (6)
Expected season: Summer
Returned season: Summer  (detected_season: True)
```

**Test: does calling with `season="winter"` return winter data regardless of the current month?**
```
Yes — returns the Winter dict with detected_season: False, even though the
current month is June. An invalid season string (e.g. "monsoon") falls back to
auto-detection and returns Summer with detected_season: True.
```
