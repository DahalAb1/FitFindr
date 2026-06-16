# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:**
Searches the local `listings.json` dataset for items matching the user's description keywords, filtered by size and max price, and returns matches ranked by keyword relevance. It's plain filtering/ranking — no LLM — because matching text against a fixed dataset is deterministic and doesn't need reasoning. It exists as the pipeline's entry point: the other two tools need a real, concrete item to act on, and this is the only tool that produces one.

**Input parameters:**
- `description` (str): Keywords describing what the user wants (e.g., "vintage graphic tee"). Each listing's `title`, `description`, and `style_tags` are checked for overlap with these keywords to compute a relevance score.
- `size` (str or None): Size to filter by, case-insensitive (e.g., "M"). If `None`, no size filtering is applied.
- `max_price` (float or None): Maximum price allowed, inclusive. If `None`, no price filtering is applied.

**What it returns:**
A list of listing dicts, sorted by relevance score (highest first). Each dict contains: `id`, `title`, `description`, `category`, `style_tags` (list), `size`, `condition`, `price` (float), `colors` (list), `brand`, `platform`. Returns an empty list (`[]`) if no listings match the filters/keywords — never raises an exception.

**What happens if it fails or returns nothing:**
If the list is empty, the agent does not call `suggest_outfit`. It sets `session["error"]` to a message naming what was searched for (description, size, price) and suggesting the user loosen one constraint (e.g., raise the budget or drop the size filter), then returns the session immediately. Stopping here matters: there's no point spending an LLM call styling an item that doesn't exist.

---

### Tool 2: suggest_outfit

**What it does:**
Given a specific thrifted item and the user's wardrobe, calls the Groq LLM to suggest 1–2 complete outfit combinations that pair the new item with pieces the user already owns. It needs an LLM, not a lookup table, because matching colors/silhouettes/vibe across arbitrary wardrobe items is a reasoning task, not a fixed rule. It exists to answer the question the user actually has after finding an item: "okay, but what do I do with it?"

**Input parameters:**
- `new_item` (dict): A listing dict (one item from `search_listings`'s results) — the item the user is considering buying. Uses fields like `title`, `category`, `style_tags`, `colors`.
- `wardrobe` (dict): A wardrobe dict with an `items` key containing a list of wardrobe item dicts (each with fields like `name`, `category`, `colors`). May have an empty `items` list.

**What it returns:**
A non-empty string containing the outfit suggestion(s) — natural-language styling advice naming specific wardrobe pieces alongside the new item, plus a styling tip (e.g., how to wear/style it).

**What happens if it fails or returns nothing:**
If `wardrobe["items"]` is empty, the tool does not fail — it calls the LLM with a different prompt asking for general styling advice for the new item alone (what it pairs well with, what vibe it suits), and still returns a non-empty string. It never returns `""` and never raises an exception, because general advice is still more useful to the user than silence, and a new user will always start with an empty wardrobe.

---

### Tool 3: create_fit_card

**What it does:**
Calls the Groq LLM to turn an outfit suggestion and the new item's details into a short, casual, shareable caption — the kind of text someone would post alongside an OOTD photo. It's a separate tool from `suggest_outfit` because the two outputs serve different audiences: styling advice is written for the user, a caption is written for the user's followers — different tone, different length, different job. Higher temperature is used here specifically so the same item doesn't produce the same caption twice.

**Input parameters:**
- `outfit` (str): The outfit suggestion string returned by `suggest_outfit()`.
- `new_item` (dict): The listing dict for the thrifted item — supplies `title`, `price`, and `platform` for the caption.

**What it returns:**
A 2–4 sentence string suitable as an Instagram/TikTok caption: casual tone, mentions the item name, price, and platform once each, and captures the outfit vibe in specific terms. Generated with a higher LLM temperature so repeated calls on the same input produce varied phrasing.

**What happens if it fails or returns nothing:**
If `outfit` is empty or whitespace-only, the tool does not call the LLM. It returns a descriptive error message string (e.g., `"Can't generate a fit card without an outfit suggestion."`) instead of raising an exception or returning `""`. Skipping the LLM call here also avoids paying for a request that would have nothing real to caption.


---

## Planning Loop

**How does your agent decide which tool to call next?**

The loop is a fixed sequence with one conditional branch that can end it early — it does not call all three tools unconditionally.

1. Parse the user's query into `description`, `size`, and `max_price`.
2. Call `search_listings(description, size, max_price)`. Check `len(results)`.
   - If `results == []`: set `session["error"]` to a message naming what was searched and what to loosen, and **return immediately**. Do not call `suggest_outfit` or `create_fit_card`.
   - If `results` is non-empty: set `session["selected_item"] = results[0]` and continue.
3. Call `suggest_outfit(session["selected_item"], wardrobe)`. This call never produces an empty result (Tool 2 always returns a non-empty string, even for an empty wardrobe), so there is no branch here — store the result in `session["outfit_suggestion"]` and continue.
4. Call `create_fit_card(session["outfit_suggestion"], session["selected_item"])`. Since `outfit_suggestion` is guaranteed non-empty by step 3, this call won't hit its own empty-outfit failure mode in normal operation — store the result in `session["fit_card"]`.
5. The loop is "done" when either: (a) `session["error"]` is set (early exit after step 2), or (b) `session["fit_card"]` is set (full success after step 4). Return the session.

The only condition that changes the agent's behavior is the emptiness check on `search_listings`'s output — that's the single decision point that determines whether the agent proceeds through the full pipeline or stops early.

---

## State Management

**How does information from one tool get passed to the next?**

A single `session` dict (defined in `_new_session()` in `agent.py`) is created once per user interaction and passed through every step of the loop — no tool re-asks the user for information already captured.

Tracked fields and when they're set:
- `query` — set once at session creation, the raw user input.
- `parsed` — set after parsing the query (description/size/max_price).
- `search_results` — set immediately after `search_listings()` returns.
- `selected_item` — set to `search_results[0]` if results are non-empty; this exact dict is what gets passed as `new_item` into both `suggest_outfit()` and `create_fit_card()`, so the item is never re-described.
- `outfit_suggestion` — set after `suggest_outfit()` returns; this exact string is what gets passed as `outfit` into `create_fit_card()`.
- `fit_card` — set after `create_fit_card()` returns; this is the final output shown to the user.
- `error` — `None` unless the loop exits early; checked first by the caller (`app.py`) to decide whether to show an error message instead of the three output panels.

Because each tool reads its inputs from the session dict (rather than from fresh user input), state flows forward automatically: `selected_item` produced by step 2 is consumed by steps 3 and 4 without any re-entry.

---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Agent does not call `suggest_outfit`. Sets `session["error"]` to a message stating what was searched for (description, size, max_price) and suggests one constraint to loosen (e.g., "No listings found for 'designer ballgown' in size XXS under $5 — try raising your budget or removing the size filter."). Returns the session immediately with `fit_card` left as `None`. |
| suggest_outfit | Wardrobe is empty (`wardrobe["items"] == []`) | Tool does not fail or return an empty string — it calls the LLM with a general-styling prompt (no wardrobe pieces to reference) and returns advice on what the item pairs well with and what vibe it suits. The loop proceeds normally to `create_fit_card`. |
| create_fit_card | Outfit input is missing or incomplete (empty/whitespace string) | Tool does not call the LLM. Returns a descriptive error string (e.g., "Can't generate a fit card without an outfit suggestion.") instead of raising an exception. The agent surfaces this string to the user as the fit card output rather than crashing. |

---

## Architecture


```
User query: "vintage graphic tee under $30, I wear baggy jeans + chunky sneakers"
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  run_agent()  —  PLANNING LOOP                                           │
│  (owns the session dict; every tool reads/writes through it)            │
│                                                                         │
│  1. parse query ──► session["parsed"] = {description, size, max_price}  │
│         │                                                               │
│         ▼                                                               │
│  2. search_listings(description, size, max_price)                       │
│         │                                                               │
│         ├── results == []  ──►  session["error"] = "No listings for..." │
│         │                       return session   ──────────────────────┼──► [ERROR EXIT]
│         │                       (suggest_outfit / create_fit_card        │     fit_card stays None;
│         │                        are NEVER called)                       │     app.py shows the error
│         │                                                               │
│         └── results = [item, ...]                                       │
│                 │                                                       │
│                 ▼                                                       │
│             session["search_results"] = results                        │
│             session["selected_item"]  = results[0]                     │
│                 │                                                       │
│                 ▼                                                       │
│  3. suggest_outfit(selected_item, wardrobe)   ◄── reads selected_item   │
│         │   (empty wardrobe ► general advice, still non-empty)          │
│         ▼                                                               │
│     session["outfit_suggestion"] = "..."                               │
│                 │                                                       │
│                 ▼                                                       │
│  4. create_fit_card(outfit_suggestion, selected_item)  ◄── reads both   │
│         │                                                               │
│         ▼                                                               │
│     session["fit_card"] = "..."                                        │
│                 │                                                       │
│                 ▼                                                       │
│  5. return session   ──────────────────────────────────────────────────┼──► [SUCCESS EXIT]
└─────────────────────────────────────────────────────────────────────────┘     app.py shows
                                                                                  item + outfit + card
```

The single branch point is step 2's emptiness check — the only place the agent's path changes. Everything downstream reads its inputs from the session dict, so the found item flows into both LLM tools without the user re-entering anything.

---

## AI Tool Plan

**Tool used for all milestones:** Claude (Claude Code), because it can read the actual stub files and data loader in this repo and match my spec to them directly, rather than me copy-pasting code back and forth.

**Milestone 3 — Individual tool implementations:**
- **search_listings:** Give Claude the Tool 1 block above (inputs, return value, failure mode) and tell it to use `load_listings()` from `utils/data_loader.py` rather than re-reading the JSON. I expect a function that filters by `max_price` and `size` first, then scores remaining listings by keyword overlap of `description` against `title`/`description`/`style_tags`, drops zero-score items, and returns the rest sorted high-to-low. *Verify before trusting:* (a) it filters on all three params, (b) `search_listings("designer ballgown", "XXS", 5)` returns `[]` not a crash, (c) `search_listings("jacket", None, 10)` returns only items with `price <= 10`. These become my pytest cases.
- **suggest_outfit:** Give Claude the Tool 2 block and tell it to call Groq `llama-3.3-70b-versatile` with `GROQ_API_KEY` from `.env`. I expect a function that branches on whether `wardrobe["items"]` is empty — empty → general styling prompt, non-empty → prompt that names specific wardrobe pieces. *Verify:* run it once with `get_example_wardrobe()` (output should name real wardrobe pieces like the baggy jeans) and once with `get_empty_wardrobe()` (output should still be a non-empty styling string, no crash).
- **create_fit_card:** Give Claude the Tool 3 block. I expect an empty-`outfit` guard that returns an error string before any LLM call, and otherwise a Groq call at higher temperature. *Verify:* `create_fit_card("", item)` returns the error string (no exception); calling it 3× on the same input produces 3 different captions, and each mentions the item title, price, and platform once.

**Milestone 4 — Planning loop and state management:**
- Give Claude the **Architecture diagram**, the **Planning Loop** section, and the **State Management** section together, and tell it to implement `run_agent()` to match them exactly. I expect parse → search → (branch on empty) → suggest → card, with every result written into the `session` dict. *Verify before trusting:* (a) the no-results query sets `session["error"]` and leaves `fit_card` as `None` with `suggest_outfit` never called, (b) on the happy path the *same* `selected_item` dict that went into `suggest_outfit` is the one passed to `create_fit_card` (print and compare), (c) it does not call all three tools unconditionally — I'll confirm by running the second test case already in `agent.py`.

---

## A Complete Interaction (Step by Step)

Write out what a full user interaction looks like from start to finish — tool call by tool call. Use a specific example query.

**Whole-app summary:** FitFindr takes one natural-language request, finds a real secondhand item from the local `listings.json` that matches it, suggests how to wear that item with the user's existing wardrobe, and writes a shareable caption for the result — searching when the user asks for an item, styling once an item is found, and captioning once an outfit exists, and stopping with a helpful message if the search comes up empty.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1 — Parse + search.**
`run_agent()` parses the query into `description="vintage graphic tee"`, `size=None` (no size stated), `max_price=30.0`, stored in `session["parsed"]`. It calls `search_listings("vintage graphic tee", None, 30.0)`. The tool filters out anything over $30, then scores by keyword overlap. `lst_006` scores highest — title *"Graphic Tee — 2003 Tour Bootleg Style"*, description *"Vintage-style bootleg tee with faded graphic"*, tags `["graphic tee", "vintage", "grunge", "streetwear", "band tee"]` — all three keywords hit. It returns a sorted list; `session["search_results"]` gets the list and `session["selected_item"]` is set to `results[0]` (`lst_006`, $24, depop).

**Step 2 — Suggest outfit.**
Because results are non-empty, the agent calls `suggest_outfit(selected_item=lst_006, wardrobe=get_example_wardrobe())`. The wardrobe has items, so the LLM is prompted with the tee plus named pieces — the baggy dark-wash jeans (`w_001`) and chunky white sneakers (`w_007`). It returns something like: *"Wear the faded graphic tee with your baggy dark-wash jeans and chunky white sneakers for an easy streetwear look. Half-tuck the front of the tee so the boxy fit doesn't swallow the high waist of the jeans."* This is stored in `session["outfit_suggestion"]`.

**Step 3 — Create fit card.**
The agent calls `create_fit_card(outfit=session["outfit_suggestion"], new_item=lst_006)`. `outfit` is non-empty, so it calls the LLM at higher temperature and returns a caption like: *"found this faded 2003 bootleg tee on depop for $24 and it was made for my baggy jeans 🖤 boxy fit + chunky sneakers = the easiest fit. full look loading."* Stored in `session["fit_card"]`. The loop returns the session.

**Final output to user:**
`app.py` checks `session["error"]` (it's `None`), then populates three panels: the found item (*Graphic Tee — 2003 Tour Bootleg Style — $24, depop, good condition*), the outfit suggestion, and the fit card caption — all from one query, with the user never re-entering the item.

**Error variant:** If the query had been *"designer ballgown size XXS under $5"*, `search_listings` returns `[]`. The agent sets `session["error"]` to *"No listings found for 'designer ballgown' in size XXS under $5 — try raising your budget or removing the size filter."*, leaves `fit_card` as `None`, and returns immediately. `suggest_outfit` and `create_fit_card` are never called, and `app.py` shows the error instead of the three panels.
