# FitFindr — planning.md

> Complete this document before writing any implementation code.

---

## Tools

### Tool 1: search_listings

**What it does:**
Searches the mock listings dataset for secondhand items matching a text description, with optional filters for size and maximum price. Returns a ranked list of matches sorted by keyword relevance.

**Input parameters:**
- `description` (str): Keywords describing what the user is looking for (e.g., "vintage graphic tee")
- `size` (str | None): Size string to filter by, case-insensitive partial match (e.g., "M" matches "S/M"). None skips size filtering.
- `max_price` (float | None): Maximum price inclusive. None skips price filtering.

**What it returns:**
A list of listing dicts sorted by relevance score (highest first). Each dict contains: id, title, description, category, style_tags (list), size, condition, price (float), colors (list), brand, platform. Returns an empty list if nothing matches.

**What happens if it fails or returns nothing:**
If the result is an empty list, the agent sets session["error"] to a helpful message telling the user to try different keywords, a higher price, or a different size, then returns early without calling suggest_outfit.
---

### Tool 2: suggest_outfit

**What it does:**
Given a thrifted item and the user's wardrobe, calls the Groq LLM to suggest 1–2 complete outfit combinations using the new item paired with existing wardrobe pieces.

**Input parameters:**
- `new_item` (dict): A listing dict for the item the user is considering buying
- `wardrobe` (dict): A wardrobe dict with an 'items' key containing a list of wardrobe item dicts. May be empty.

**What it returns:**
A non-empty string with outfit suggestions. If the wardrobe is empty, returns general styling advice for the item instead of specific combinations.

**What happens if it fails or returns nothing:**
If the LLM call fails, returns a hardcoded fallback string with basic styling advice rather than raising an exception or returning an empty string.

---

### Tool 3: create_fit_card

**What it does:**
Generates a short, shareable Instagram/TikTok-style caption for the thrifted outfit, calling the Groq LLM with a higher temperature for varied output.

**Input parameters:**
- `outfit` (str): The outfit suggestion string returned by suggest_outfit
- `new_item` (dict): The listing dict for the thrifted item

**What it returns:**
A 2–4 sentence casual caption mentioning the item name, price, and platform naturally. Returns a descriptive error string if outfit is empty — never raises an exception.

**What happens if it fails or returns nothing:**
If outfit is empty or whitespace, returns "Unable to create fit card: no outfit suggestion provided." If the LLM call fails, returns a hardcoded fallback caption.

---

## Planning Loop

The agent parses the user's query using regex to extract a description, size, and max_price. It then calls search_listings with those parameters. If the result is empty, it sets an error message and stops — it does not call the remaining tools with empty input. If results exist, it selects the top result (results[0]) and passes it to suggest_outfit along with the user's wardrobe. The outfit suggestion is then passed to create_fit_card. The loop ends after create_fit_card returns and all results are stored in the session dict.

---

## State Management

All state is stored in a session dict initialized at the start of each run. The session tracks: the original query, parsed parameters (description, size, max_price), search results, the selected item, the wardrobe, the outfit suggestion, the fit card, and any error. Each tool's output is stored in the session before the next tool is called, so no tool needs to re-fetch or re-compute data from a previous step. For example, session["selected_item"] is set after search_listings and read directly by suggest_outfit — the user never has to re-enter it.

---

## Error Handling

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Sets session["error"] = "No listings found for '...'. Try different keywords, a higher price, or a different size." Returns early without calling suggest_outfit. |
| suggest_outfit | Wardrobe is empty | Calls LLM with a general styling prompt instead of a wardrobe-specific one. Returns general advice rather than crashing or returning empty string. |
| create_fit_card | Outfit input is empty string | Returns "Unable to create fit card: no outfit suggestion provided." without calling the LLM or raising an exception. |
---

## Architecture

```
User Query
    │
    ▼
Parse Query (regex)
extract: description, size, max_price
    │
    ▼
search_listings(description, size, max_price)
    │
    ├─── empty list ──► set session["error"] ──► return session (early exit)
    │
    ▼
session["selected_item"] = results[0]
    │
    ▼
suggest_outfit(selected_item, wardrobe)
    │
    ├─── wardrobe empty ──► general styling advice (no crash)
    │
    ▼
session["outfit_suggestion"] = result
    │
    ▼
create_fit_card(outfit_suggestion, selected_item)
    │
    ├─── outfit empty ──► return error string (no crash)
    │
    ▼
session["fit_card"] = result
    │
    ▼
Return completed session dict
```

---

## AI Tool Plan

**Milestone 3 — Individual tool implementations:**
I gave Claude Code the spec for each tool one at a time — the input parameters, return value, and failure mode from this planning.md. For search_listings I asked it to implement keyword scoring using title, description, and style_tags fields. I verified each function by calling it directly with test inputs in the terminal before moving to the next tool. For suggest_outfit and create_fit_card I checked that failure modes returned strings rather than raising exceptions.

**Milestone 4 — Planning loop and state management:**
I gave Claude Code the full planning loop description and state management section from this doc, plus the agent diagram above. I asked it to implement run_agent() following the exact conditional logic described. I verified by running python agent.py and checking that session["selected_item"] matched what was passed into suggest_outfit, and that the no-results path returned an error without calling the LLM tools.

---

## A Complete Interaction (Step by Step)

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers."

**Step 1:**
The agent parses the query and extracts description = "I'm looking for a vintage graphic tee", max_price = 30.0, size = None. It calls search_listings("I'm looking for a vintage graphic tee", size=None, max_price=30.0). This returns a list of matching listings — the top result is the Y2K Baby Tee — Butterfly Print at $18.

**Step 2:**
Since results is not empty, the agent sets session["selected_item"] = results[0]. It calls suggest_outfit(selected_item, wardrobe). The wardrobe has items, so the LLM receives the item details and wardrobe contents and returns 1–2 specific outfit combinations referencing named wardrobe pieces.

**Step 3:**
The agent calls create_fit_card(outfit_suggestion, selected_item). The LLM generates a casual 2–4 sentence Instagram caption mentioning the Y2K Baby Tee, the $18 price, and depop naturally.

**Final output to user:**
The Gradio interface displays three panels: the top listing details (title, size, price, condition, platform, description), the outfit suggestion with specific combinations, and the fit card caption ready to share.