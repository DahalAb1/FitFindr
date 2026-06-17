# FitFindr 🛍️

FitFindr is a small AI **agent** that helps you thrift secondhand clothes and figure out how to wear them. You type something casual like *"I want a vintage graphic tee under $30, I usually wear baggy jeans"* and it does three things for you in one go: it finds a matching item from a catalog of listings, it suggests how to style that item using clothes you already own, and it writes a caption you could actually post under an outfit photo.

Before going further, it's worth defining the word **agent**, because it's the whole point of this project. A normal program runs a fixed list of steps in the same order every time. An agent is a program that **makes decisions about what to do next based on what just happened**. FitFindr is an agent because it doesn't blindly run all its steps — if the search comes up empty, it notices, stops, and tells you why, instead of plowing ahead and trying to style an item that doesn't exist. That "notice and decide" behavior is what separates an agent from an ordinary script, and it's the thing this README spends the most time explaining.

---

## How to run it

First, some setup. The project uses a **virtual environment** (a `.venv` folder) — that's just an isolated copy of Python and its packages that belongs only to this project, so installing things here can't break Python anywhere else on your machine.

**macOS / Linux:**
```bash
python -m venv .venv          # create the isolated environment
source .venv/bin/activate     # switch into it (your prompt shows (.venv))
pip install -r requirements.txt   # install the packages this project needs
```

**Windows:**
```bash
python -m venv .venv
source .venv/Scripts/activate
pip install -r requirements.txt
```

Two of the three tools need to talk to a large language model (an **LLM** — the kind of AI that writes text, like the one behind ChatGPT). FitFindr uses a free one called Llama 3.3, served by a company called Groq. To use it you need an API key, which is basically a password that proves you're allowed to make requests. You put it in a file called `.env` (the leading dot just means "hidden config file"), and the code reads it from there so the key never has to be written directly into the source:

```
GROQ_API_KEY=your_key_here
```

You can get a free key at [console.groq.com](https://console.groq.com). The `.env` file is listed in `.gitignore`, which means git will refuse to commit it — that keeps your key out of GitHub.

Once that's done, there are three ways to run things:

```bash
python app.py        # the web interface — open the localhost URL it prints
python agent.py      # a quick command-line test of the happy path + the error path
pytest tests/        # runs the automated tests
```

---

## The big picture: how the three pieces fit together

The project is deliberately split into three layers, each in its own file. Understanding why they're separate makes the rest of the README click.

- **`tools.py` — the tools.** A "tool" here just means a single Python function that does one well-defined job and can be called and tested on its own. There are three of them. They don't know about each other and they don't know about the agent — each one just takes some input and returns some output.
- **`agent.py` — the planning loop.** This is the brain that calls the tools in the right order, decides whether to keep going or stop, and remembers everything that happened along the way. It's the part that makes this an "agent" rather than three disconnected functions.
- **`app.py` — the interface.** This is the web page (built with a library called Gradio, which turns Python functions into a clickable UI with zero web-development work). It takes what you typed, hands it to the agent, and lays the results out in three panels.

The reason for splitting it this way is **testability**. Each tool can be proven correct by itself before any of them are connected. As the project hints put it: an agent built from three untested tools is three times harder to debug than three tools you tested one at a time. So the build order was tools first, then the brain that orchestrates them, then the UI on top.

---

## The three tools (in `tools.py`)

A quick note on how to read the function signatures below. Something like `size (str or None)` means the parameter named `size` is normally a piece of text (a "string", abbreviated `str`), but it's also allowed to be `None`, which is Python's way of saying "nothing / not provided." A `dict` is a collection of labeled values (like `{"price": 18.0, "title": "..."}`), and `list[dict]` means a list where every element is one of those dicts. These types matter because the whole agent depends on each tool handing the next one exactly the shape of data it expects.

### Tool 1 — `search_listings(description, size, max_price)`

**What problem it solves.** Everything downstream needs a *real, specific item* to work with. You can't suggest an outfit for "a vague idea of a tee." So this tool's job is to turn a fuzzy description into a concrete listing pulled from the catalog.

**Inputs:**
- `description` (str) — the keywords you're searching for, e.g. `"vintage graphic tee"`.
- `size` (str or None) — a size to filter by, matched case-insensitively as a substring so `"M"` will match a listing labeled `"S/M"`. If it's `None`, the tool simply doesn't filter by size.
- `max_price` (float) — the most you're willing to pay, inclusive. (A `float` is a number that can have decimals, like `30.0`.) If it's `None`, there's no price ceiling.

**What it returns.** A `list[dict]` — a list of matching listings, sorted so the best match is first. Each listing dict contains: `id`, `title`, `description`, `category`, `style_tags` (a list of style words like `["vintage", "grunge"]`), `size`, `condition`, `price` (a float), `colors` (a list), `brand`, and `platform` (which secondhand app it's from). If nothing matches, it returns an **empty list** `[]` — importantly, *not* an error.

**How it actually works, and why.** This is the only tool that does **not** use the LLM, and that's a deliberate choice. Searching a fixed catalog by keywords is a *deterministic* problem — the same query should always return the same results — and deterministic problems don't need an AI that reasons; they need plain, predictable logic that's faster, free, and easy to test. The logic has three stages:

1. **Hard filters first.** Walk through every listing and immediately discard any that cost more than `max_price` or whose size doesn't match. These are non-negotiable constraints — there's no point scoring an item you literally can't afford or fit.
2. **Relevance scoring.** For each surviving listing, glue its title, description, and style tags into one block of text and count how many of your search words appear in it. More overlap = higher score. This is a simple stand-in for "how relevant is this item to what I asked for."
3. **Drop the zeros and sort.** Any listing that scored 0 shares no words with your query, so it's dropped. The rest get sorted highest-score-first and returned.

**What happens when it fails.** "Failure" here means *no item matched* — and the tool handles it by returning `[]` rather than crashing. That empty list is a signal, not an error, and it's the agent's job (not the tool's) to react to it. We'll see exactly how in the planning-loop section.

### Tool 2 — `suggest_outfit(new_item, wardrobe)`

**What problem it solves.** Finding an item is only half the battle. The real question a thrifter has is *"okay, but what do I wear it with?"* This tool answers that by pairing the found item with clothes you already own.

**Inputs:**
- `new_item` (dict) — one listing dict, exactly the kind `search_listings` returns. This is the item you're considering buying.
- `wardrobe` (dict) — a dictionary with an `items` key holding a list of the clothes you own. Each owned piece has a `name`, `category`, `colors`, `style_tags`, and optional `notes`. Crucially, this list **can be empty** — a brand-new user hasn't entered any clothes yet.

**What it returns.** A `str` (a piece of text) containing the outfit advice. If you have a wardrobe, the advice names specific pieces you own ("pair it with your baggy dark-wash jeans and chunky white sneakers") plus a styling tip. If your wardrobe is empty, it gives general advice about what *kinds* of things go with the item instead.

**How it works, and why it uses the LLM.** Unlike searching, styling is **not** a deterministic lookup — there's no table that says "graphic tee + these exact jeans = good." Deciding whether colors, silhouettes, and vibes work together is a judgment call, the kind of open-ended reasoning that LLMs are actually good at. So this tool builds a text prompt describing the item and your wardrobe, sends it to Llama 3.3, and returns whatever styling advice the model writes back. (A "prompt" is just the instructions you give the LLM in plain English; the quality of the prompt largely determines the quality of the answer.)

**What happens when it fails.** The failure mode here is an **empty wardrobe**. A naive version might crash trying to loop over zero items, or return an empty string. Instead, the tool checks up front whether `wardrobe["items"]` is empty, and if so it switches to a different prompt that asks for general styling ideas for the item on its own. The reasoning: a new user with no clothes entered yet should still get *something* useful, because generic advice beats a blank screen — and every user starts out with an empty wardrobe, so this isn't a rare edge case, it's the first-run experience.

### Tool 3 — `create_fit_card(outfit, new_item)`

**What problem it solves.** The styling advice from Tool 2 is written *for you* — it's practical, instructional. But the project also wants a "fit card": a short, fun caption written *for your followers*, the kind of thing you'd actually type under an outfit post. Those are two genuinely different writing jobs (different tone, different length, different audience), which is exactly why this is a separate tool instead of being folded into Tool 2.

**Inputs:**
- `outfit` (str) — the styling advice string that `suggest_outfit` produced.
- `new_item` (dict) — the same listing dict, used to mention the item's name, price, and platform naturally in the caption.

**What it returns.** A `str` — a casual 2–3 sentence caption that name-drops the item, its price, and where it's from, exactly once each, in a voice that sounds like a real person rather than a product listing.

**How it works, and one specific detail.** Like Tool 2, this calls the LLM with a prompt. But there's a deliberate twist: it runs at a higher **temperature**. Temperature is a dial on the LLM between 0 and roughly 1 that controls randomness — low temperature makes the model pick the most likely next word every time (consistent but repetitive), high temperature lets it take more chances (more varied, more creative). A caption that came out word-for-word identical every time would feel robotic, and the project explicitly requires the output to vary, so this tool turns the temperature up to `1.0`.

**What happens when it fails.** The failure mode is being handed an **empty outfit string** (which would happen if something upstream went wrong). Before spending an LLM call on nothing, the tool checks whether `outfit` is empty or just whitespace, and if so it returns a plain error message — `"Can't generate a fit card without an outfit suggestion."` — rather than crashing or paying for a request that has nothing to caption.

---

## The planning loop (in `agent.py`)

This is the heart of the project. Everything above is just three functions sitting in a file; the planning loop is what turns them into an agent that *behaves*.

### What "state" means and why we need it

When you make several tool calls in a row, each one produces something the next one needs. The found item has to reach the styling tool; the styling advice has to reach the caption tool. If we made you re-type the item before each step, that would be a miserable experience — and it would also mean the agent has no memory. So we keep a single dictionary called `session` that travels through the whole interaction and accumulates everything. This idea — one shared place that holds everything known so far — is called **state**, and "state management" is just the discipline of putting results in there and reading them back out.

The `session` dict has a slot for every meaningful thing that happens:

| Field | When it's filled in | Who reads it later |
|---|---|---|
| `query` | the moment the session is created | the parser |
| `parsed` | after we pull size/price/keywords out of the query | the search tool |
| `search_results` | right after the search runs | the branch that decides whether to continue |
| `selected_item` | after search, set to the top result | handed to **both** the styling tool and the caption tool |
| `outfit_suggestion` | after styling | handed to the caption tool |
| `fit_card` | after the caption is written | shown to the user |
| `error` | only if something went wrong and we stopped early | the UI, which checks it first |

The thing to notice is `selected_item`. It's set **once**, from the top search result, and then the *exact same dictionary* is passed into `suggest_outfit` and later into `create_fit_card`. The user never re-enters the item — it flows forward on its own. That's state management working.

### The actual decision logic, step by step

The function `run_agent()` runs these steps in order. The whole reason it's an "agent" is hiding in step 2.

1. **Parse the query.** The user typed one freeform sentence, but `search_listings` needs three separate things: keywords, a size, and a price. So a small helper called `_parse_query` uses **regex** (regular expressions — a mini-language for finding patterns in text) to pull them apart. For example it spots the `$30` after the word "under" and turns it into `max_price = 30.0`, spots "size M" and extracts `"M"`, and treats the leftover words as the description. We chose regex here instead of asking the LLM to parse the query because parsing is deterministic and we'd rather not pay for an extra AI call (and an extra possible point of failure) just to read a number out of a sentence. The result is stored in `session["parsed"]`.

2. **Search — and this is the decision point.** We call `search_listings` with the parsed values and store the result in `session["search_results"]`. Then comes the single branch that makes this whole thing adaptive:
   - **If the search came back empty:** we do *not* continue. We write a specific, helpful message into `session["error"]` — something like *"No listings found for 'designer ballgown' in size XXS under $5 — try raising your budget or removing the size filter"* — and we `return` right there. The two LLM tools are never called. This matters for a concrete reason: calling `suggest_outfit` with no item would be meaningless (and would waste a paid API request on nonsense), so the agent refuses to.
   - **If the search found something:** we set `session["selected_item"]` to the first (best-scoring) result and carry on.

3. **Suggest the outfit.** We call `suggest_outfit(selected_item, wardrobe)` and store the text in `session["outfit_suggestion"]`. There's no branch here, because Tool 2 is guaranteed to return a non-empty string no matter what (remember, it handles the empty-wardrobe case internally).

4. **Write the fit card.** We call `create_fit_card(outfit_suggestion, selected_item)` and store the result in `session["fit_card"]`.

5. **Return the session.** Whoever called us (the UI) now has a dict containing either a filled-in `error`, or a complete set of `selected_item` + `outfit_suggestion` + `fit_card`.

### Why this counts as an adaptive planning loop

The key idea the project is testing is whether the agent **behaves differently depending on what it gets back** — as opposed to mechanically running all three tools every time no matter what. FitFindr passes this test at exactly one place: the emptiness check in step 2. Feed it a reasonable query and it runs the full three-tool pipeline. Feed it something impossible like *"designer ballgown size XXS under $5"* and it runs **one** tool, recognizes the dead end, and bails out with an explanation. Same code, two genuinely different execution paths, chosen at runtime based on data. That's the planning loop earning its name.

---

## Error handling, tool by tool

A core requirement of this project is that **every tool handles its own failure** — "crash the whole agent" and "fail silently and return nothing" are both unacceptable. Here's the failure each tool can hit and what it does instead of breaking:

| Tool | What can go wrong | What it does about it |
|------|-------------|----------------|
| `search_listings` | Nothing in the catalog matches the query | Returns an empty list `[]` instead of raising an error. The planning loop sees the empty list, writes a specific message into `session["error"]` telling the user what to loosen, and stops before calling the other tools. |
| `suggest_outfit` | The wardrobe is empty (new user, no clothes entered) | Detects the empty wardrobe up front and switches to a general-advice prompt, still returning useful non-empty text. The pipeline keeps going normally. |
| `create_fit_card` | It's handed an empty outfit string | Checks for that before doing anything else and returns a plain, readable error message instead of calling the LLM or crashing. |

### A concrete example from actually testing this

Here's the real output from running the agent on a deliberately impossible query:

```
$ python agent.py
...
=== No-results path ===
Error message: No listings found for 'designer ballgown' in size XXS under $5 — try raising your budget or removing the size filter.
```

What this proves: the search returned `[]`, the loop caught it, `session["fit_card"]` stayed empty (`None`), and the two LLM tools were never invoked. The user isn't left staring at a blank screen or a stack trace — they're told *exactly what failed* (no matches for that description at that size and price) and *exactly what to try next* (raise the budget, or drop the size filter). That "what failed + what to do about it" shape is the difference between an error message that helps and one that just frustrates.

---

## Spec reflection

I wrote a full spec in `planning.md` before writing any code, and it's worth being honest about how that played out.

**One way the spec helped.** Writing the planning loop's branching logic out in plain English *first* — literally the sentence "if results is empty, set an error and return immediately; otherwise set selected_item and continue" — meant that when I got to `agent.py`, the code was almost a direct transcription of that sentence. I never had to figure out the control flow and write the code at the same time, which is where bugs usually sneak in. The State Management table in the spec did the same favor: it nailed down in advance which `session` field each tool reads and writes, so connecting the tools together had zero ambiguity about where any piece of data lived.

**One place the implementation diverged from the spec, and why.** In the spec's walkthrough I predicted that searching "vintage graphic tee" would return a specific listing (`lst_006`, the 2003 bootleg tee) as the top match. The finished code actually returns a *different* item — the Y2K Baby Tee at $18 — because my keyword scorer counts word overlap across the title, description, and tags, and the Baby Tee happens to tie or beat the bootleg tee and sits earlier in the catalog. The result is still correct (it's a genuine vintage graphic tee under $30), so the divergence is only between *which item I guessed while planning* and *which item the scoring actually picks*. I deliberately left the scorer simple rather than over-engineering it just to match my prediction — the prediction was the thing that was wrong, not the code.

---

## How AI was used to build this

I built FitFindr with the help of an AI coding assistant (Claude), and the project asks me to be specific about that. Two representative instances:

**Implementing `search_listings`.** I gave the assistant the Tool 1 section of my `planning.md` — the inputs, the return shape, and the failure behavior — and told it to implement the function using the existing `load_listings()` helper instead of re-reading the data file itself. Before trusting the generated code I checked three specific things against my spec: that it filtered on all three parameters, that an impossible query returned `[]` rather than throwing an exception, and that the price filter genuinely excluded anything over the limit. I confirmed each one with a quick smoke test (`"designer ballgown"` at size XXS under $5 returned `[]`; a search for jackets under $10 returned only items at or below $10) and then froze those exact checks into the pytest suite so they can't silently break later.

**Implementing the planning loop.** I gave the assistant the Planning Loop section, the State Management section, and the architecture diagram together, and asked it to write `run_agent()` to match all three. The first version didn't include a way to turn the freeform query into search parameters, so I overrode that and had it add a self-contained regex parser rather than spending an LLM call on parsing — a choice I made deliberately for speed and predictability. Then I verified the two behaviors the whole project hinges on: that the no-results query sets `session["error"]` and leaves the fit card empty without ever calling the LLM tools, and that the *same* `selected_item` object flows into both `suggest_outfit` and `create_fit_card` so nothing is re-entered.

---

## Project structure

```
FitFindr/
├── data/
│   ├── listings.json          # 40 mock secondhand listings (the catalog we search)
│   └── wardrobe_schema.json    # the wardrobe format + a sample 10-item wardrobe
├── utils/
│   └── data_loader.py          # helpers: load_listings / get_example_wardrobe / get_empty_wardrobe
├── tools.py                    # the 3 tools — each independently callable and tested
├── agent.py                    # the planning loop + the session state dict
├── app.py                      # the Gradio web interface
├── tests/
│   └── test_tools.py           # one test per failure mode + happy-path checks
└── planning.md                 # the spec, written before any code
```
