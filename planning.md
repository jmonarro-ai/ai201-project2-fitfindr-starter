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
Searches the mock listings dataset for secondhand items that match the user's description, optional size, and optional price ceiling. Returns a ranked list of matching items sorted by keyword relevance.

**Input parameters:**
- `description` (str): Keywords describing what the user is looking for (e.g., "vintage graphic tee"). Used to score each listing by keyword overlap against title, description, and style_tags.
- `size` (str | None): Size string to filter by, or None to skip size filtering. Matching is case-insensitive (e.g., "M" will match "S/M").
- `max_price` (float | None): Maximum price (inclusive), or None to skip price filtering.

**What it returns:**
A list of listing dicts sorted by relevance score (highest first). Each dict contains: id (str), title (str), description (str), category (str), style_tags (list[str]), size (str), condition (str), price (float), colors (list[str]), brand (str or None), platform (str). Returns an empty list if nothing matches — never raises an exception.

**What happens if it fails or returns nothing:**
If the list is empty, the agent sets session["error"] to: "No listings found for '[query]'. Try broadening your search — remove the size filter, raise your price limit, or use different keywords." The agent returns the session immediately and does NOT call suggest_outfit or create_fit_card.

---

### Tool 2: suggest_outfit

**What it does:**
Given a thrifted item the user is considering and their current wardrobe, calls the Groq LLM to suggest 1–2 complete outfit combinations using specific pieces from the wardrobe.

**Input parameters:**
- `new_item` (dict): A listing dict representing the item the user is considering buying. Contains title, description, category, style_tags, colors, price, platform, and condition.
- `wardrobe` (dict): A wardrobe dict with an 'items' key containing a list of wardrobe item dicts. Each wardrobe item has: id, name, category, colors, style_tags, and optional notes. May be empty.

**What it returns:**
A non-empty string with 1–2 outfit suggestions. If the wardrobe is empty, returns general styling advice for the item (what types of pieces pair well, what vibe it suits). Never raises an exception or returns an empty string.

**What happens if it fails or returns nothing:**
If wardrobe['items'] is empty, the LLM is prompted for general styling advice instead of specific combinations. If the LLM call fails, returns: "Unable to generate outfit suggestions right now. The item is a [category] with [style_tags] — try pairing it with complementary basics."

---

### Tool 3: create_fit_card

**What it does:**
Generates a short, casual, shareable outfit caption (2–4 sentences) for the thrifted find — the kind of thing someone would post as an Instagram or TikTok OOTD caption.

**Input parameters:**
- `outfit` (str): The outfit suggestion string returned by suggest_outfit(). Used as context for the caption tone and specific pieces mentioned.
- `new_item` (dict): The listing dict for the thrifted item. Used to pull title, price, and platform for the caption.

**What it returns:**
A 2–4 sentence string written in a casual, authentic voice. Mentions the item name, price, and platform naturally once each. Sounds different for different inputs (higher LLM temperature). If outfit is empty or whitespace-only, returns the error string: "Cannot generate a fit card without an outfit suggestion. Please try your search again."

**What happens if it fails or returns nothing:**
If outfit is empty or whitespace-only, returns the descriptive error string above — does NOT raise an exception. If the LLM call fails, returns: "Fit card unavailable — but this [item title] for $[price] from [platform] is worth it."

---

### Additional Tools

### Tool 4: compare_price (Stretch Feature)

**What it does:**
Given a listing, finds comparable items in the dataset (same category, similar style tags) and returns a price assessment: whether the item is a good deal, average, or overpriced relative to similar listings.

**Input parameters:**
- `item` (dict): A listing dict for the item being evaluated.

**What it returns:**
A string summarizing the price assessment, e.g.: "This [title] is priced at $X. Comparable [category] items in the dataset range from $Y to $Z (avg $W). This is [below average / about average / above average] for its category and condition."

**What happens if it fails or returns nothing:**
If no comparable items are found, returns: "Not enough comparable listings to assess price. Check sold listings on Depop or Poshmark for a better estimate."

---

### Tool 5: style_profile_memory (Stretch Feature)

**What it does:**
Saves and loads a user's style profile (wardrobe + style preferences) to/from a local JSON file so they don't have to re-enter their wardrobe each session.

**Input parameters:**
- `action` (str): Either "save" or "load".
- `profile_data` (dict | None): The wardrobe dict to save. Only required when action="save".
- `user_id` (str): Identifier for the user's profile file (default: "default").

**What it returns:**
On "save": confirmation string. On "load": the saved wardrobe dict, or an empty wardrobe if no profile exists yet.

**What happens if it fails or returns nothing:**
If loading fails (file missing or corrupted), returns get_empty_wardrobe() with a note that no saved profile was found.

---

## Planning Loop

**How does your agent decide which tool to call next?**

The planning loop in `run_agent()` follows this conditional logic:

1. Parse the query using the Groq LLM to extract `description` (str), `size` (str or None), and `max_price` (float or None). Store in `session["parsed"]`.

2. Call `search_listings(description, size, max_price)`. Store result in `session["search_results"]`.
   - **IF** `session["search_results"]` is empty → set `session["error"]` to a helpful message and **return session immediately**. Do NOT proceed.
   - **IF** results exist → set `session["selected_item"] = session["search_results"][0]` (top result) and continue.

3. Call `suggest_outfit(session["selected_item"], session["wardrobe"])`. Store result in `session["outfit_suggestion"]`.
   - This always runs if we reach this step — the empty wardrobe case is handled inside the tool itself.

4. Call `create_fit_card(session["outfit_suggestion"], session["selected_item"])`. Store result in `session["fit_card"]`.

5. Return the completed session.

The agent never calls `suggest_outfit` or `create_fit_card` unconditionally — they only run if `search_listings` returned at least one result.

---

## State Management

**How does information from one tool get passed to the next?**

All state is stored in a single `session` dict initialized by `_new_session()` at the start of each `run_agent()` call. The session contains:

- `session["query"]`: the original user query string
- `session["parsed"]`: dict with keys `description`, `size`, `max_price` — extracted from the query and passed directly to `search_listings`
- `session["search_results"]`: the full list returned by `search_listings` — stored so the agent can access any result, not just the top one
- `session["selected_item"]`: `session["search_results"][0]` — the single listing dict passed into both `suggest_outfit` and `create_fit_card`
- `session["wardrobe"]`: passed in at the start of `run_agent()` and forwarded unchanged to `suggest_outfit`
- `session["outfit_suggestion"]`: the string returned by `suggest_outfit` — passed directly as the `outfit` argument to `create_fit_card`
- `session["fit_card"]`: the final caption string returned by `create_fit_card`
- `session["error"]`: set to a string if any step fails; None on a successful run

No tool receives the session dict directly — each tool receives only the specific values it needs, extracted from the session by the planning loop.

---

## Error Handling

| search_listings | No listings match the query (empty list returned) | Sets session["error"] to "No listings found for that description. Try removing the size filter, raising your price limit, or using different keywords." Returns session immediately — suggest_outfit and create_fit_card are never called. |
| suggest_outfit | wardrobe items list is empty (new user with no wardrobe) | Calls LLM with a general styling prompt instead of a wardrobe-specific one. Returns general advice like "This piece pairs well with high-waisted bottoms and chunky sneakers for a 90s streetwear look." Never crashes. |

---

## Architecture
User query (natural language)

│

▼

run_agent(query, wardrobe)

│

▼

Step 1: Parse query with LLM

│  → session["parsed"] = {description, size, max_price}

│

▼

Step 2: search_listings(description, size, max_price)

│  → session["search_results"] = [list of listing dicts]

│

├─── results == [] ──► session["error"] = "No listings found..."

│                               │

│                               ▼

│                          return session  ◄─── EARLY EXIT

│

│ results not empty

│  → session["selected_item"] = results[0]

│

▼

Step 3: suggest_outfit(selected_item, wardrobe)

│  → session["outfit_suggestion"] = "Pair this with..."

│

│  (if wardrobe empty → general styling advice, no crash)

│

▼

Step 4: create_fit_card(outfit_suggestion, selected_item)

│  → session["fit_card"] = "thrifted this... 🖤"

│

│  (if outfit empty → returns error string, no crash)

│

▼

return session

{

selected_item: {...},

outfit_suggestion: "...",

fit_card: "...",

error: None

}

---

## AI Tool Plan

**Milestone 3 — Individual tool implementations:**

- **search_listings**: I will give Claude the Tool 1 spec block from planning.md (inputs with types, return value description, failure mode) and the listings.json field list. I will ask it to implement the function using `load_listings()` from `utils/data_loader.py`, filtering by price and size, scoring by keyword overlap across title, description, and style_tags, and returning an empty list (not raising) when nothing matches. Before running, I will verify: (1) all three parameters are used, (2) size matching is case-insensitive, (3) listings with score 0 are dropped, (4) result is sorted descending by score. I will test with 3 queries: one that returns results, one that returns empty, one that tests the price filter.

- **suggest_outfit**: I will give Claude the Tool 2 spec block and the wardrobe_schema.json structure. I will ask it to implement the function calling Groq's `llama-3.3-70b-versatile`, with two prompt branches: one for empty wardrobe (general styling advice) and one for populated wardrobe (specific combinations using named wardrobe pieces). Before running, I will verify: (1) it checks `wardrobe['items']` before building the prompt, (2) it uses `_get_groq_client()`, (3) it never returns an empty string.

- **create_fit_card**: I will give Claude the Tool 3 spec block. I will ask it to guard against empty outfit input first, then build a prompt asking for a casual Instagram-style caption mentioning item name, price, and platform. I will ask for temperature=1.2 or higher. Before running, I will verify: (1) the empty-string guard comes before the LLM call, (2) the caption sounds casual not like a product description, (3) running it twice on the same input gives different output.

**Milestone 4 — Planning loop and state management:**

I will give Claude the full Architecture diagram and the Planning Loop + State Management sections from planning.md. I will ask it to implement `run_agent()` in agent.py, using the session dict already defined in `_new_session()`. Before running, I will verify: (1) the function returns early if search_results is empty, (2) selected_item is set to results[0] before calling suggest_outfit, (3) outfit_suggestion is passed directly into create_fit_card, (4) no tool is called unconditionally.

---


## A Complete Interaction (Step by Step)

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
The agent calls the Groq LLM to parse the query. The LLM extracts: `description = "vintage graphic tee"`, `size = None` (no size specified), `max_price = 30.0`. These are stored in `session["parsed"]`.

**Step 2:**
The agent calls `search_listings("vintage graphic tee", size=None, max_price=30.0)`. The function loads all listings, filters to those priced ≤ $30, then scores each by keyword overlap with "vintage graphic tee" against title, description, and style_tags. It returns 3 matches sorted by score. The top result is: `{"title": "Vintage Band Tee — Faded Grey", "price": 19.0, "platform": "depop", ...}`. This is stored as `session["selected_item"]`.

**Step 3:**
The agent calls `suggest_outfit(selected_item, wardrobe)`. The wardrobe has 10 items including baggy straight-leg jeans and chunky white sneakers. The LLM is prompted with the item details and all 10 wardrobe pieces. It returns: "Pair this faded grey band tee with your baggy straight-leg jeans and chunky white sneakers for a classic 90s streetwear look. Tuck the front of the tee slightly and add your black crossbody bag to finish it off." Stored in `session["outfit_suggestion"]`.

**Step 4:**
The agent calls `create_fit_card(outfit_suggestion, selected_item)`. The LLM receives the item details (title, $19, depop) and the outfit suggestion, and generates: "found this faded band tee on depop for $19 and it was literally made for my baggy jeans era 🖤 chunky sneakers and a crossbody and we're done here." Stored in `session["fit_card"]`.

**Final output to user:**
- **Top listing panel**: "Vintage Band Tee — Faded Grey | $19.00 | depop | Size: L | Condition: fair | Style: vintage, grunge, band tee"
- **Outfit idea panel**: The suggest_outfit string above
- **Fit card panel**: The create_fit_card caption above

**Error path (no results):**
If the query were "designer ballgown size XXS under $5", search_listings returns []. The agent sets session["error"] = "No listings found for 'designer ballgown'. Try removing the size filter, raising your price limit, or using different keywords." The agent returns immediately. suggest_outfit and create_fit_card are never called. The UI shows the error message in the top listing panel and empty strings in the other two panels.