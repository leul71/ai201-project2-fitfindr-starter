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
<!-- Describe what this tool does in 1–2 sentences -->

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `description` (str):  Natural language description of the item (e.g. "vintage graphic tee", "leather jacket")
- `size` (str): Clothing size to filter by (e.g. M or L )
- `max_price` (float): Maximum price the user will pay

**What it returns:**
A list of dicts, each containing: id, title, description, price, size, condition, platform, style_tags, colors, brand, category. Returns an empty list [] if no matches found.

**What happens if it fails or returns nothing:**
The agent sets session["error"] to a message like "No listings found for '[description]' in size [size] under $[max_price]. Try a broader description or higher price." It stops — does not call suggest_outfit or create_fit_card.

---

### Tool 2: suggest_outfit

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Given a specific thrifted item and the user's wardrobe, calls the LLM to suggest one or more complete outfit combinations and styling notes.


**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `new_item` (dict): ...A single listing dict returned by search_listings (the selected item)
- `wardrobe` (dict): ... The user's wardrobe object from the session, with an items list of clothing pieces

**What it returns:**
<!-- Describe the return value -->
A string — a paragraph of outfit suggestions with specific styling advice (e.g. "Pair with your wide-leg jeans and platform sneakers for a 90s look. Tuck the front corner slightly for shape.")

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the wardrobe is empty or no outfit can be suggested? -->
If wardrobe["items"] is empty, the LLM is still called but prompted to give general styling advice for the item without referencing a specific wardrobe. If the LLM call fails, returns the string: "Couldn't generate outfit suggestions right now. Try pairing this with basics in a similar color palette."

---

### Tool 3: create_fit_card

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Generates a short, casual, shareable caption describing the complete outfit the kind of thing someone would post on Instagram or TikTok with their fit pic.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `outfit` (...): ... The outfit suggestion string returned by suggest_outfit

**What it returns:**
<!-- Describe the return value -->
A single string, 1–3 sentences, casual social media tone, referencing the item, price, and platform. Different output each run due to LLM temperature.

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the outfit data is incomplete? -->
If outfit is an empty string, returns the error string: "Couldn't create a fit card — outfit description was missing." Does not crash.

---

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->

---

## Planning Loop

**How does your agent decide which tool to call next?**
<!-- Describe the logic your planning loop uses. What does it look at? What conditions change its behavior? How does it know when it's done? -->
Call search_listings(description, size, max_price) and immediately return early with an error if no items are found; otherwise, assign the first result to session["selected_item"]. Next, call suggest_outfit(session["selected_item"], session["wardrobe"]). If it returns an error string, set session["error"] and return early; otherwise, save the outfit to session["outfit_suggestion"]. Finally, generate a fit card using create_fit_card, store it in session["fit_card"], and return the complete session.

---

## State Management

**How does information from one tool get passed to the next?**
<!-- Describe how your agent stores and accesses state within a session. What data is tracked? How is it passed between tool calls? -->
A session dict is initialized at the start of each run_agent() call and passed through every step:
The data tracked include: "query": <original user input>, "wardrobe": <loaded wardrobe>, "selected_item": None, "outfit_suggestion": None,  "fit_card": None, "error": None              

---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | |   Sets session["error"] = "No listings found for '[description]' in size [size] under $[max_price]. Try a broader description, different size, or higher price." Returns early.
| suggest_outfit | Wardrobe is empty | | Calls LLM with a modified prompt: "The user has no saved wardrobe items. Give general styling advice for this item." Returns general advice string instead of crashing.
| create_fit_card | Outfit input is missing or incomplete | | Returns error string "Couldn't create a fit card — outfit description was missing." without calling LLM.

---

## Architecture

<!-- Draw a diagram of your agent showing how the components connect:
     User input → Planning Loop → Tools (search_listings, suggest_outfit, create_fit_card)
                                                                          ↕
                                                                   State / Session
     Show what triggers each tool, how state flows between them, and where error paths branch off.
     ASCII art, a Mermaid diagram (https://mermaid.js.org/syntax/flowchart.html), or an embedded
     sketch are all fine. You'll share this diagram with an AI tool when asking it to implement
     the planning loop and each individual tool. -->
     User query (description, size, max_price, wardrobe)
    │
    ▼
Planning Loop
    │
    ├─► search_listings(description, size, max_price)
    │       │
    │       ├── results == [] ──► session["error"] = "No listings found..." ──► RETURN ◄─┐
    │       │                                                                              │
    │       └── results = [item, ...] ──► session["selected_item"] = results[0]          │
    │                                           │                                         │
    ├─► suggest_outfit(selected_item, wardrobe) │                                         │
    │       │                                   │                                         │
    │       ├── LLM fails ──────────────────────────────────────────────────────────────►─┘
    │       │                                                                              
    │       └── success ──► session["outfit_suggestion"] = "..."                         
    │                                           │                                         
    └─► create_fit_card(outfit_suggestion, selected_item)                                
            │                                                                             
            ├── outfit == "" ──► session["error"] = "Couldn't create fit card..." ──► RETURN
            │                                                                             
            └── success ──► session["fit_card"] = "..."
                                                │
                                                ▼
                                        Return complete session
( I made this using AI)
---

## AI Tool Plan

<!-- For each part of the implementation below, describe:
     - Which AI tool you plan to use (Claude, Copilot, ChatGPT, etc.)
     - What you'll give it as input (which sections of this planning.md, your agent diagram)
     - What you expect it to produce
     - How you'll verify the output matches your spec before moving on

     "I'll use AI to help me code" is not a plan.
     "I'll give Claude my Tool 1 spec (inputs, return value, failure mode) and ask it to implement
     search_listings() using load_listings() from the data loader — then test it against 3 queries
     before trusting it" is a plan. -->

**Milestone 3 — Individual tool implementations:**

search_listings: Give Claude the Tool 1 spec (inputs, return value, failure mode) plus the note that load_listings() is available from utils/data_loader.py. Ask it to implement the function filtering by description (keyword match on title/description/style_tags), size, and max_price. Verify the output handles all three filter params independently (each optional), returns a list of dicts, and returns [] on no match. Test with 3 queries: one that matches, one with impossible price, one with nonexistent size.
suggest_outfit: Give Claude the Tool 2 spec plus the wardrobe schema. Ask it to implement a Groq LLM call using llama-3.3-70b-versatile with a prompt that includes the new item's title/style_tags/colors and the wardrobe items list. Verify it handles empty wardrobe (modified prompt path) and doesn't crash on LLM error. Test with example wardrobe and empty wardrobe.
create_fit_card: Give Claude the Tool 3 spec. Ask it to implement a Groq LLM call with a prompt asking for a casual social-media caption referencing the item name, price, and platform from new_item. Verify the empty-outfit guard is present and that temperature is set high enough (≥0.9) for varied output. Run 3 times on same input to confirm variation.

**Milestone 4 — Planning loop and state management:**
Give Claude the full Architecture diagram above and the Planning Loop + State Management sections. Ask it to implement run_agent() in agent.py using the session dict structure defined above. Verify before running: does it branch on empty results? Does it store values in session between steps? Does it NOT call all three tools unconditionally? Test the no-results branch explicitly with an impossible query.

---

## A Complete Interaction (Step by Step)

Write out what a full user interaction looks like from start to finish — tool call by tool call. Use a specific example query.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
<!-- What does the agent do first? Which tool is called? With what input? -->
Agent calls search_listings("vintage graphic tee", size=None, max_price=30.0). The function filters listings by keyword match on "vintage" and "graphic tee" in title/description/style_tags, and price ≤ 30. Returns: [{"title": "Faded Band Tee", "price": 22.0, "platform": "Depop", "condition": "Good", ...}, ...]. Agent sets session["selected_item"] = results[0]

**Step 2:**
<!-- What happens next? What was returned from step 1? What tool is called now? -->
Agent calls suggest_outfit(session["selected_item"], session["wardrobe"]) where wardrobe contains baggy jeans and chunky sneakers. LLM receives the item details and wardrobe items and returns: "Pair this faded band tee with your wide-leg jeans and chunky sneakers for a classic 90s grunge look. Roll the sleeves once and tuck the front corner slightly for shape." Agent sets session["outfit_suggestion"] to this string.

**Step 3:**
<!-- Continue until the full interaction is complete -->
 Agent calls create_fit_card(session["outfit_suggestion"], session["selected_item"]). LLM receives the outfit suggestion and item details (name: "Faded Band Tee", price: $22, platform: Depop) and returns a casual caption. Agent sets session["fit_card"] to the result.

**Final output to user:**
<!-- What does the user actually see at the end? -->
Search result panel:Faded Band Tee — $22, Depop, Good condition
Outfit suggestion panel: "Pair this faded band tee with your wide-leg jeans and chunky sneakers for a classic 90s grunge look. Roll the sleeves once and tuck the front corner slightly for shape."
Fit card panel: "thrifted this faded band tee off depop for $22 and it was literally made for my wide-legs 🖤 full look in my stories"
