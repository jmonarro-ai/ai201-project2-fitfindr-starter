# FitFindr 🛍️

FitFindr is a multi-tool AI agent that helps users find secondhand clothing pieces and figure out how to wear them. It takes a natural language query, searches a mock listings dataset, suggests outfit combinations based on the user's wardrobe, and generates a shareable fit card caption — all in one interaction.

---

## What's Included
ai201-project2-fitfindr-starter/

├── data/

│   ├── listings.json          # 40 mock secondhand listings

│   └── wardrobe_schema.json   # Wardrobe format + example wardrobe

├── utils/

│   └── data_loader.py         # Helper functions for loading the data

├── tests/

│   └── test_tools.py          # pytest tests for all three tools

├── planning.md                # Planning spec — filled out before implementation

├── agent.py                   # Planning loop and session state

├── app.py                     # Gradio UI

├── tools.py                   # The three required tools

└── requirements.txt           # Python dependencies

---

## Setup

1. Clone the repository and activate your virtual environment:
```bash
python -m venv .venv
.venv\Scripts\activate  # Windows
source .venv/bin/activate  # Mac/Linux
```

2. Install dependencies:
```bash
pip install -r requirements.txt
```

3. Set your Groq API key in a `.env` file (get a free key at [console.groq.com](https://console.groq.com)):
GROQ_API_KEY=your_key_here

4. Run the app:
```bash
python app.py
```

Then open http://127.0.0.1:7860 in your browser.

---

## The Mock Listings Dataset

`data/listings.json` contains 40 mock secondhand listings across categories (tops, bottoms, outerwear, shoes, accessories) and styles (vintage, y2k, grunge, cottagecore, streetwear, and more).

Each listing has: `id`, `title`, `description`, `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`, and `platform`.

Load it with:
```python
from utils.data_loader import load_listings
listings = load_listings()
```

---

## The Wardrobe Schema

`data/wardrobe_schema.json` defines the format your agent uses to represent a user's existing wardrobe. It includes:

- `schema`: field definitions for a wardrobe item
- `example_wardrobe`: a sample wardrobe with 10 items you can use for testing
- `empty_wardrobe`: a starting template for a new user

Load an example wardrobe with:
```python
from utils.data_loader import get_example_wardrobe
wardrobe = get_example_wardrobe()
```

---

## Tool Inventory

### `search_listings(description, size, max_price)`
- **Inputs:** `description` (str) — keywords describing the item; `size` (str or None) — size filter, case-insensitive partial match; `max_price` (float or None) — maximum price inclusive
- **Output:** A list of listing dicts sorted by relevance score (highest first). Each dict contains: `id`, `title`, `description`, `category`, `style_tags` (list), `size`, `condition`, `price` (float), `colors` (list), `brand`, `platform`. Returns an empty list if nothing matches — never raises an exception.
- **Purpose:** Searches the mock listings dataset by filtering on price and size, then scoring each listing by keyword overlap with the description across title, description, and style_tags fields.

### `suggest_outfit(new_item, wardrobe)`
- **Inputs:** `new_item` (dict) — a listing dict for the item being considered; `wardrobe` (dict) — a wardrobe dict with an `items` key containing a list of wardrobe item dicts (may be empty)
- **Output:** A non-empty string with 1–2 outfit suggestions. If the wardrobe is empty, returns general styling advice instead of wardrobe-specific combinations.
- **Purpose:** Calls the Groq LLM (llama-3.3-70b-versatile) to suggest complete outfit combinations using the new item and specific pieces from the user's wardrobe.

### `create_fit_card(outfit, new_item)`
- **Inputs:** `outfit` (str) — the outfit suggestion string from `suggest_outfit()`; `new_item` (dict) — the listing dict for the thrifted item
- **Output:** A 2–4 sentence casual Instagram/TikTok-style caption mentioning the item name, price, and platform naturally. Returns a descriptive error string if `outfit` is empty — never raises an exception.
- **Purpose:** Calls the Groq LLM at temperature=1.2 to generate a shareable, authentic-sounding outfit caption that varies with each call.

---

## How the Planning Loop Works

The planning loop in `run_agent()` makes decisions based on what each tool returns — it does not call all three tools unconditionally.

**Step 1 — Parse the query:** The agent sends the user's natural language query to the Groq LLM and extracts three parameters: `description` (str), `size` (str or None), and `max_price` (float or None). If parsing fails, it falls back to using the full query as the description with no filters.

**Step 2 — Search:** The agent calls `search_listings()` with the parsed parameters and stores the results in `session["search_results"]`.

**Step 3 — Branch on results:**
- **If `search_results` is empty:** The agent sets `session["error"]` to a specific, actionable message and returns immediately. `suggest_outfit` and `create_fit_card` are never called. This is the key decision point.
- **If results exist:** The agent sets `session["selected_item"] = session["search_results"][0]` and continues.

**Step 4 — Suggest outfit:** The agent calls `suggest_outfit()` with the selected item and the user's wardrobe.

**Step 5 — Create fit card:** The agent calls `create_fit_card()` with the outfit suggestion and selected item.

**Step 6 — Return session:** The completed session dict is returned with all results populated.

---

## State Management

All state is stored in a single `session` dict initialized at the start of each `run_agent()` call. No tool receives the session directly — each tool receives only the specific values it needs, extracted from the session by the planning loop.

| Key | Set when | Passed to |
|-----|----------|-----------|
| `session["query"]` | Start of run | Not passed — kept for reference |
| `session["parsed"]` | After LLM query parse | Values extracted and passed to `search_listings` |
| `session["search_results"]` | After `search_listings` | Used to set `selected_item` |
| `session["selected_item"]` | After search, if results exist | Passed into `suggest_outfit` and `create_fit_card` |
| `session["wardrobe"]` | Start of run | Passed into `suggest_outfit` |
| `session["outfit_suggestion"]` | After `suggest_outfit` | Passed into `create_fit_card` |
| `session["fit_card"]` | After `create_fit_card` | Returned to UI |
| `session["error"]` | If any step fails | Returned to UI instead of results |

The item found by `search_listings` flows directly into `suggest_outfit` without the user re-entering it, and the outfit from `suggest_outfit` flows directly into `create_fit_card`.

---

## Error Handling

### `search_listings`
**Failure mode:** No listings match the query (returns empty list).
**Agent response:** Sets `session["error"]` to a specific, actionable message and returns immediately — `suggest_outfit` and `create_fit_card` are never called.
**Concrete example from testing:** Query "designer ballgown size XXS under $5" returned `[]`. The UI displayed the error message in the Top Listing panel while the Outfit Idea and Fit Card panels remained empty, confirming the agent stopped early.

### `suggest_outfit`
**Failure mode:** `wardrobe['items']` is empty (new user with no wardrobe entered).
**Agent response:** Calls the LLM with a general styling prompt instead of a wardrobe-specific one. Returns useful general advice regardless of wardrobe state — never crashes or returns an empty string.
**Concrete example from testing:** Running `suggest_outfit(results[0], get_empty_wardrobe())` returned general styling advice with specific suggestions for bottoms, shoes, and layers — no exception raised.

### `create_fit_card`
**Failure mode:** `outfit` argument is empty or whitespace-only.
**Agent response:** Returns the string "Cannot generate a fit card without an outfit suggestion. Please try your search again." without calling the LLM — never raises an exception.
**Concrete example from testing:** Running `create_fit_card('', results[0])` returned the exact error string above with no exception.

---

## Spec Reflection

**One way the spec helped:** The planning.md architecture diagram was the most useful artifact in the project. Having the exact conditional logic written out — specifically the early return on empty search results — made the `run_agent()` implementation straightforward. Without it, it would have been easy to accidentally call `suggest_outfit` with empty input.

**One way implementation diverged from the spec:** The spec described query parsing using "regex, string splitting, or LLM." I initially planned to use regex for simplicity, but switched to LLM-based parsing because natural language queries like "looking for a vintage graphic tee under $30" are hard to parse reliably with regex. The LLM approach handles varied phrasing much more robustly, with a fallback if the LLM call fails.

---

## AI Usage

### Instance 1: Implementing `search_listings`
I gave Claude the Tool 1 spec block from `planning.md` (inputs with types, return value description, failure mode) and the `listings.json` field list. I asked it to implement the function using `load_listings()` from `utils/data_loader.py`, filtering by price and size, and scoring by keyword overlap. I reviewed the generated code and verified it handled all three parameters, used case-insensitive size matching, dropped zero-score listings, and returned an empty list rather than raising on no results. I tested it with three queries before trusting it.

### Instance 2: Implementing the planning loop in `agent.py`
I gave Claude the full Architecture diagram from `planning.md` and the Planning Loop and State Management sections. I asked it to implement `run_agent()` following the numbered TODO steps already in the file. Before running the generated code, I reviewed it to confirm it branched on the `search_results` being empty, stored each result in the correct session key, and did not call all three tools unconditionally. I overrode the query parsing approach — the generated code used regex, but I directed Claude to switch to LLM-based parsing with a fallback, which matched my planning.md spec better.

---

## Running Tests

```bash
pytest tests/
```

All 10 tests should pass, covering all three tools and each failure mode.

---

## Where to Start (for contributors)

1. **Read `planning.md` and fill it out before writing any code.**
2. Verify the data loads correctly by running `python utils/data_loader.py`.
3. Build and test each tool individually before connecting them through your planning loop.

Your implementation files go in this same directory. There's no required file structure for your agent code — organize it however makes sense for your design.

