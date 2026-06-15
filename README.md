# FitFindr

A multi-tool AI agent that helps users find secondhand clothing and figure out how to wear it. Given a natural language query, FitFindr searches mock thrift listings, suggests outfit combinations based on your wardrobe, and generates a shareable fit card caption.

## Setup

```bash
pip install -r requirements.txt
```

Add your Groq API key to a `.env` file in the project root:

Get a free key at [console.groq.com](https://console.groq.com). Then run:
```bash
python app.py
```

---

## Tool Inventory

### `search_listings(description, size, max_price)`
Searches the mock listings dataset for secondhand items matching the user's query.
- `description` (str): Natural language keywords (e.g. "vintage graphic tee")
- `size` (str or None): Size to filter by — skipped if None
- `max_price` (float or None): Price ceiling — skipped if None
- **Returns:** List of matching listing dicts sorted by keyword relevance. Empty list if nothing matches.

### `suggest_outfit(new_item, wardrobe)`
Calls the Groq LLM to suggest 1–2 complete outfit combinations using the thrifted item and the user's wardrobe.
- `new_item` (dict): A listing dict returned by search_listings
- `wardrobe` (dict): User wardrobe with an `items` list
- **Returns:** A string with specific outfit suggestions and styling tips. If wardrobe is empty, returns general styling advice instead.

### `create_fit_card(outfit, new_item)`
Calls the Groq LLM to generate a short, casual social media caption for the outfit.
- `outfit` (str): The outfit suggestion string from suggest_outfit
- `new_item` (dict): The listing dict (used for item name, price, platform)
- **Returns:** A 2–3 sentence caption in OOTD post style. Different output each run due to high LLM temperature (0.95).

---

## How the Planning Loop Works

`run_agent()` in `agent.py` runs these steps in order, branching on results:

1. Parse the user query with regex to extract `description`, `size`, and `max_price`
2. Call `search_listings()` — if it returns an empty list, set an error message and return early. Do NOT proceed.
3. Set `session["selected_item"]` to the top result
4. Call `suggest_outfit()` with the selected item and wardrobe
5. Call `create_fit_card()` with the outfit suggestion and selected item
6. Return the completed session

The agent never calls `suggest_outfit` if `search_listings` returned nothing, and never calls `create_fit_card` if there is no outfit suggestion.

---

## State Management

A `session` dict is initialized at the start of each `run_agent()` call and passed through every step:

```python
session = {
    "query": <original user input>,
    "parsed": {"description": ..., "size": ..., "max_price": ...},
    "search_results": [...],
    "selected_item": None,       # set after search_listings
    "wardrobe": <wardrobe dict>,
    "outfit_suggestion": None,   # set after suggest_outfit
    "fit_card": None,            # set after create_fit_card
    "error": None                # set if any step fails
}
```

Each tool receives its inputs directly from the session — no tool re-fetches data another tool already returned.

---

## Error Handling

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| `search_listings` | No listings match the query | Sets `session["error"]` with a message like "No listings found for 'designer ballgown' in size XXS under $5. Try a broader description, different size, or higher price." Returns early — outfit and fit card are never called. |
| `suggest_outfit` | Empty wardrobe | Calls LLM with a modified prompt asking for general styling advice rather than wardrobe-specific combinations. Never crashes. |
| `create_fit_card` | Empty outfit string | Returns the string "Couldn't create a fit card — outfit description was missing." without calling the LLM. |

**Concrete example:** Running the query "designer ballgown size XXS under $5" returns no results from `search_listings`. The agent sets `session["error"] = "No listings found..."` and returns immediately — `suggest_outfit` and `create_fit_card` are never called, and the Gradio UI displays the error in the first panel with the other two panels empty.

---

## Spec Reflection

**One way planning.md helped:** Writing out the exact conditional logic for the planning loop before coding made it straightforward to implement — the session dict structure and early-return behavior were already decided, so `run_agent()` was mostly transcribing the plan into Python.

**One way implementation diverged:** The spec described parsing the query with the LLM, but regex turned out to be simpler and faster for extracting size and price. The LLM approach would have added latency and an extra API call for something rule-based patterns handle well enough.

---

## AI Usage

**Instance 1 — Tool implementations:** I gave Claude the Tool 1–3 spec blocks from `planning.md` (inputs, return values, failure modes) one at a time and asked it to implement each function. For `search_listings`, I verified the generated code filtered by all three parameters independently and returned `[]` on no match before using it. I added `or []` and `or ""` guards after discovering null values in the listings JSON caused a crash the generated code didn't anticipate.

**Instance 2 — Planning loop:** I gave Claude the full architecture diagram and Planning Loop + State Management sections from `planning.md` and asked it to implement `run_agent()`. The generated code matched the spec structure but I switched query parsing from an LLM call to regex, since the spec left the parsing method open and regex was faster and more reliable for extracting size and price patterns.