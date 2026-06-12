# FitFindr

A multi-tool AI agent that helps users find secondhand clothing and figure out how to wear it. Given a natural language query, FitFindr searches thrift listings, suggests outfit combinations using the user's wardrobe, and generates a shareable fit card caption.

---

## Tool Inventory

### search_listings(description, size, max_price)
- **Purpose:** Search the mock listings dataset for items matching a description with optional size and price filters
- **Inputs:** description (str), size (str or None), max_price (float or None)
- **Output:** List of listing dicts sorted by keyword relevance score, or empty list if no matches
- **Each listing contains:** id, title, description, category, style_tags, size, condition, price, colors, brand, platform

### suggest_outfit(new_item, wardrobe)
- **Purpose:** Suggest 1–2 complete outfit combinations using the thrifted item and the user's existing wardrobe
- **Inputs:** new_item (dict — a listing), wardrobe (dict with 'items' key)
- **Output:** Non-empty string with outfit suggestions, or general styling advice if wardrobe is empty

### create_fit_card(outfit, new_item)
- **Purpose:** Generate a casual Instagram/TikTok-style caption for the outfit
- **Inputs:** outfit (str — suggestion from suggest_outfit), new_item (dict — the listing)
- **Output:** 2–4 sentence caption mentioning item name, price, and platform naturally

---

## How the Planning Loop Works

The agent parses the user's query using regex to extract three things: a description, an optional size, and an optional max price. It then calls search_listings with those parameters. If the result is empty, it sets an error message and stops immediately — it never calls suggest_outfit or create_fit_card with empty input. If results exist, it picks the top result and passes it to suggest_outfit along with the user's wardrobe. The outfit suggestion is then passed to create_fit_card. The agent's behavior differs depending on what search_listings returns — it does not call all three tools unconditionally.

---

## State Management

All state lives in a single session dict created at the start of each run. It tracks: the original query, parsed parameters, search results, the selected item, the wardrobe, the outfit suggestion, the fit card, and any error. Each tool writes its output into the session before the next tool reads from it. For example, session["selected_item"] is set after search_listings runs and passed directly into suggest_outfit — the user never has to re-enter it between steps.

---

## Error Handling

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Sets an error message and returns early. suggest_outfit is never called with empty input. |
| suggest_outfit | Wardrobe is empty | Calls LLM with a general styling prompt instead of wardrobe-specific one. Returns useful advice rather than crashing. Tested by passing get_empty_wardrobe() directly. |
| create_fit_card | Outfit string is empty | Returns "Unable to create fit card: no outfit suggestion provided." without calling the LLM. Tested by calling create_fit_card("", item) directly. |

---

## Spec Reflection

The planning.md spec was most useful when implementing the planning loop — having the exact conditional logic written out made it straightforward to prompt Claude Code and verify the output matched. The one place implementation diverged from the spec was query parsing: the spec described extracting a clean description by cutting the string before size/price mentions, but in practice the regex sometimes left filler words like "I'm looking for" in the description. This didn't break anything since keyword scoring in search_listings handles noise, but a cleaner approach would strip common filler phrases before scoring.

---

## AI Usage

**Instance 1 — Implementing tools.py:**
I gave Claude Code the spec for each tool from planning.md one at a time — inputs, return value, and failure mode. For search_listings I asked it to implement keyword scoring against title, description, and style_tags. I reviewed the generated code against my spec before running it to confirm the size filter used case-insensitive partial matching as described. For create_fit_card I verified the empty-outfit guard returned the exact error string from my spec before accepting the output.

**Instance 2 — Implementing agent.py:**
I gave Claude Code the Planning Loop and State Management sections from planning.md plus the architecture diagram. I asked it to implement run_agent() following the exact conditional logic. The generated code correctly branched on empty search results and stored values in the session dict. I overrode one thing: the generated code placed import re at the top of the file, but since the spec said not to modify anything outside run_agent(), I moved the import inside the function.