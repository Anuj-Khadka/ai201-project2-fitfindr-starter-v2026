# FitFindr

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first** — setup, every
> command, and what to do when something breaks.
>
> Once `python test.py` passes:
>
> ```bash
> python app.py listings --full -n 6      # read the data (Milestone 1)
> python app.py fields                    # what you can filter on
> python app.py ask 'vintage graphic tee under $30'
> ```
>
> All three tools are stubs, so that last command will do nothing useful yet.
> That's the starting position.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     HOW TO USE THIS FILE

     This is your submission. Fill each section in as you finish the milestone
     it belongs to — don't leave it all to the end.

     Unit 3 asks for the first five sections. Unit 4 adds the five below them.
     Leave the unit 4 sections alone until then; they're here so you know
     what's coming.

     Everything is pasted as TEXT. No screenshots, no images, no video links.
     A typed block of output gets full credit; a picture of the same output
     gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 3 — THE BUILD ═══════════════════════ -->

## What This Does

<!-- Three or four sentences: what a user asks for, and what they get back. -->

This app takes a plain-language request for a thrifted item like a vintage graphic tee
under $30 and turns it into a styled recommendation. It searches 40 mock listings by keywords,
size, and a price ceiling, then picks the top match. It asks a model to build one or two outfits 
around that item using the user's existing wardrobe, then writes a short social-media caption 
about the find. If nothing matches, it stops and tells the user what to change instead of 
inventing an outfit for an item that does not exist.

---

## Tool Inventory

<!-- Four lines per tool. This is worth 2 points and it's the single most
     common place students lose them.

     "Returns a list" earns NOTHING. The description has to say what is IN
     the list.

     The empty case isn't optional either — it's the thing your loop branches
     on, and if you don't decide it here you'll discover it as a crash in
     Milestone 5. -->

### `search_listings`

- **What it does:** Searches the 40 listings for items whose text matches the
  description, optionally narrowing by size and a price ceiling, and returns the
  best matches ranked by keyword overlap.

- **Size match rule:** case-insensitive *token* match. The listing's `size` is
  split on `/`, spaces, and parentheses into tokens; an item matches if the
  requested size equals one of those tokens exactly. So "M" matches `"S/M"` and
  `"M/L"` but not `"XL"` or `"US 7"`. This avoids substring false-positives like
  `"l" in "xl"`.
  
- **Inputs:** <!-- name and type each: `max_price` (float), not "a price" -->
`description` (str) — keywords like "vintage graphic tee";
`size` (str or None) — e.g. "M", or None to skip size filtering;
`max_price` (float or None) — inclusive ceiling, or None to skip.

- **Returns:** A `list[dict]`, best match first, at most `SEARCH_RESULT_LIMIT`
  (10). Each dict is a full listing with `id, title, description, category,
  style_tags, size, condition, price (float), colors, brand (str or None),
  platform`.

- **When it has nothing:** Returns an empty list `[]` — not `None`, not an
  exception. This is the value the planning loop branches on.



### `suggest_outfit`

- **What it does:** Takes one thrifted item and the user's wardrobe and asks the
  model for one or two outfits built around that item.


- **Inputs:** 
`new_item` (dict) — a single listing dict from `search_listings`;
`wardrobe` (dict) — has an `items` key holding a list of wardrobe items, which
  may be empty.

- **Returns:** A non-empty `str` of outfit suggestions. With a non-empty
  wardrobe it names pieces the user already owns; with an empty wardrobe it
  returns general styling advice for the item.

- **When it has nothing:** On an empty wardrobe (`items == []`) it returns
  general advice rather than "" or an error.




### `create_fit_card`

- **What it does:** Writes a short social-post caption (2–4 sentences) about the
  item and the suggested outfit.

- **Inputs:** 
`outfit` (str) — the suggestion from `suggest_outfit`;
`new_item` (dict) — the same listing dict.

- **Returns:** A `str` of 2–4 sentences that names the item, its price, and its
  platform once each, and reads like a post rather than a product description.
  Varies run to run (TEMPERATURE 0.9, cache off during eval).

- **When it has nothing:** If `outfit` is empty or whitespace, returns a short
  descriptive message rather than raising.

---

## Planning Loop

<!-- Your branch rule, stated as a rule — the condition AND both paths — plus
     the file and function that holds it.

     Like this:
       "If search_listings returns an empty list, put a message in the session
        and stop. Otherwise take the first result and go to suggest_outfit."
        — agent.py::run_agent

     The grader checks your code against what you claim here, so the file and
     function have to be real. -->


**Branch rule:** If `search_listings` returns an empty list, put a message in
`session["error"]` naming what the user could change (price, size, or keywords)
and return immediately — do **not** call `suggest_outfit`. Otherwise, take the
first result as `session["selected_item"]` and continue to `suggest_outfit`,
then `create_fit_card`.

**Where it lives:** `agent.py::run_agent`

**How the query is parsed:** String/regex parsing (no model call). A `$NN` or
"under NN" sets `max_price`; "size X" sets `size`; the remaining words are the
description. Deterministic so the same query parses the same way every run.

**What moves through the session:** `query` → `parsed` (description/size/
max_price) → `search_results` → `selected_item` → `outfit_suggestion` →
`fit_card`, each written to the session before the next tool reads it back out.

---

## Sample Run

<!-- Two things go here.

     1. One FULL query and its output, pasted as text.
     2. Your three per-tool terminal tests — the command and what it printed. -->

**One full query**

```
$ python app.py ask '...'

$ python agent.py
=== A query the data can match ===
  found:    Y2K Baby Tee — Butterfly Print — $18.0 on depop
  outfit:   Hey bestie! That Y2K butterfly baby tee is such a find—the pink and purple print gives major early-2000s pop star off-duty energy. Since the tee is a fitted crop, it's super easy to balance out with pieces you already have. 

Here are two ways to style your new baby tee using items straight from your closet:

### Outfit 1: Bratz Doll Casual (Y2K Streetwear)
*This look plays on the contrast between the tight, girly top and ultra-baggy bottoms, which is the ultimate Y2K silhouette.*

* **Top:** The new Y2K Butterfly Baby Tee
* **Bottoms:** Baggy straight-leg jeans (dark wash)
* **Outerwear:** Black cropped zip hoodie (wear it unzipped or slung off the shoulders so the butterfly graphic still pops)
* **Shoes:** Chunky white sneakers
* **Accessories:** Black crossbody bag 

**Why it works:** The dark indigo wash of the jeans grounds the soft pink and purple tones of the tee, while the chunky sneakers and cropped hoodie lock in that authentic turn-of-the-millennium look.


---


### Outfit 2: Sweet & Edgy Mix (Cottagecore Meets Y2K)
*Leaning into the cottagecore tag, this outfit pairs the sweet butterfly graphic with tougher, earthy pieces for a cool thrifted vibe.*

* **Top:** The new Y2K Butterfly Baby Tee
* **Bottoms:** Wide-leg khaki trousers 
* **Outerwear:** Vintage black denim jacket
* **Shoes:** Black combat boots
* **Accessories:** Brown leather belt (loop this through the trousers to pull the khaki and earthy tones together)

**Why it works:** The khaki trousers and brown belt soften the graphic tee, giving it a subtle vintage-cottagecore feel, while the black combat boots and distressed denim jacket add that signature thrift-store edge. 

Which vibe are you wearing out first?
  fit card: Still can’t believe I scored this Y2K butterfly baby tee for just $18. The pink and purple print is literally giving off-duty pop star. Definitely listing it on Depop before I change my mind and keep it for myself!

=== A query it can't ===
  stopped: No listings matched. Try raising your price ceiling above $5, or removing the size 'XXS' filter, or trying different keywords.
  fit_card is None — it should still be None here

The second one should stop before the fit card. If both paths look the same,
the branch isn't doing anything yet.

```

```
$ python app.py ask 'looking for a vintage graphic tee under $30'

  Found:    Y2K Baby Tee — Butterfly Print — $18.0 on depop

  Outfit:   Hey bestie! That Y2K butterfly baby tee is such a find—the pink and purple print gives major early-2000s pop star off-duty energy. Since the tee is a fitted crop, it's super easy to balance out with pieces you already have. 

Here are two ways to style your new baby tee using items straight from your closet:

### Outfit 1: Bratz Doll Casual (Y2K Streetwear)
*This look plays on the contrast between the tight, girly top and ultra-baggy bottoms, which is the ultimate Y2K silhouette.*

* **Top:** The new Y2K Butterfly Baby Tee
* **Bottoms:** Baggy straight-leg jeans (dark wash)
* **Outerwear:** Black cropped zip hoodie (wear it unzipped or slung off the shoulders so the butterfly graphic still pops)
* **Shoes:** Chunky white sneakers
* **Accessories:** Black crossbody bag 

**Why it works:** The dark indigo wash of the jeans grounds the soft pink and purple tones of the tee, while the chunky sneakers and cropped hoodie lock in that authentic turn-of-the-millennium look.

---

### Outfit 2: Sweet & Edgy Mix (Cottagecore Meets Y2K)
*Leaning into the cottagecore tag, this outfit pairs the sweet butterfly graphic with tougher, earthy pieces for a cool thrifted vibe.*

* **Top:** The new Y2K Butterfly Baby Tee
* **Bottoms:** Wide-leg khaki trousers 
* **Outerwear:** Vintage black denim jacket
* **Shoes:** Black combat boots
* **Accessories:** Brown leather belt (loop this through the trousers to pull the khaki and earthy tones together)

**Why it works:** The khaki trousers and brown belt soften the graphic tee, giving it a subtle vintage-cottagecore feel, while the black combat boots and distressed denim jacket add that signature thrift-store edge. 

Which vibe are you wearing out first?

  Fit card: Still can’t believe I scored this Y2K butterfly baby tee for just $18. The pink and purple print is literally giving off-duty pop star. Definitely listing it on Depop before I change my mind and keep it for myself!


```

**The three tools, tested one at a time**

```
$ python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"

$ python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"
[{'id': 'lst_002', 'title': 'Y2K Baby Tee — Butterfly Print', 'description': 'Super cute early 2000s baby tee with butterfly graphic. Fitted crop length. Tag says medium but fits like a small.', 'category': 'tops', 'style_tags': ['y2k', 'vintage', 'graphic tee', 'cottagecore'], 'size': 'S/M', 'condition': 'excellent', 'price': 18.0, 'colors': ['white', 'pink', 'purple'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_006', 'title': 'Graphic Tee — 2003 Tour Bootleg Style', 'description': 'Vintage-style bootleg tee with faded graphic. Slightly boxy fit. 100% cotton, soft and worn-in.', 'category': 'tops', 'style_tags': ['graphic tee', 'vintage', 'grunge', 'streetwear', 'band tee'], 'size': 'L', 'condition': 'good', 'price': 24.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_017', 'title': 'Mesh Long-Sleeve Top — Black', 'description': 'Sheer black mesh long-sleeve. Great for layering under a graphic tee or over a bralette. Stretchy material, fits true to size.', 'category': 'tops', 'style_tags': ['y2k', 'grunge', 'goth', 'layering'], 'size': 'S/M', 'condition': 'excellent', 'price': 15.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_033', 'title': 'Vintage Band Tee — Faded Grey', 'description': 'Faded grey band-style tee with distressed graphic. Crew neck. Fits boxy. Well-loved but no holes or major damage.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'band tee', 'graphic tee', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 19.0, 'colors': ['grey', 'charcoal'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_011', 'title': 'Low-Rise Cargo Pants — Khaki', 'description': 'Y2K era low-rise cargo pants. Lots of pockets. Khaki color, slightly distressed at the hems. Great for layering with a long tee.', 'category': 'bottoms', 'style_tags': ['y2k', 'cargo', '2000s', 'streetwear'], 'size': 'W29', 'condition': 'fair', 'price': 27.0, 'colors': ['khaki', 'tan'], 'brand': None, 'platform': 'poshmark'}, {'id': 'lst_012', 'title': 'Oversized Crewneck Sweatshirt — Vintage Navy', 'description': 'Perfectly faded navy crewneck. Genuinely vintage — not manufactured distressed. Ribbed cuffs and hem. No graphics, clean.', 'category': 'tops', 'style_tags': ['vintage', 'basics', 'oversized', 'classic'], 'size': 'XL (fits oversized)', 'condition': 'good', 'price': 20.0, 'colors': ['navy'], 'brand': None, 'platform': 'thredUp'}, {'id': 'lst_015', 'title': 'Vintage Graphic Hoodie — Faded Black', 'description': 'Faded black pullover hoodie with barely-visible vintage graphic on the chest. Cozy interior. Some pilling but adds to the worn-in look.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'graphic', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 26.0, 'colors': ['black', 'charcoal'], 'brand': None, 'platform': 'depop'}]

```

```
$ python -c "from tools import search_listings; print(search_listings('designer ballgown', max_price=5))"
[]
```


```
$ python -c "from tools import suggest_outfit; ..."

$ python  -c "from tools import suggest_outfit; from utils.data_loader import get_example_wardrobe, load_listings; print(suggest_outfit(load_listings()[0], get_example_wardrobe()))"
Hey there! Those Vintage Levi's 501 Jeans are an absolute holy grail find—that medium wash with the natural knee fading is timeless and so easy to style. 

Since the 501s have that classic, slightly structured straight-leg fit, we can play with proportions using what you already have in your closet. Here are two effortless thrift-inspired looks featuring your new Levi's:

### Outfit 1: Casual Streetwear Layering
*This look leans into the vintage-meets-streetwear vibe, balancing fitted and oversized pieces for an effortless everyday fit.*

* **Top 1:** **White ribbed tank top** (tucked into the jeans to define the waist)
* **Outerwear:** **Oversized grey crewneck sweatshirt** layered right over the tank, or draped loosely over your shoulders
* **Shoes:** **Chunky white sneakers** to keep it grounded and sporty
* **Accessories:** **Black crossbody bag** and a **Brown leather belt** to tie the denim and hardware together

### Outfit 2: Edgy Denim-on-Denim
*Double denim is having a massive moment, and contrasting the medium wash 501s with black denim creates a cool, moody contrast.*

* **Top:** **Black cropped zip hoodie** worn zipped up partway
* **Outerwear:** **Vintage black denim jacket** layered on top for a textured, monochromatic top-half
* **Shoes:** **Black combat boots** to add some attitude and toughen up the medium wash denim
* **Accessories:** **Black crossbody bag** 

Which vibe are you leaning toward first?
```

```
$ python -c "from tools import suggest_outfit; from utils.data_loader import get_empty_wardrobe, load_listings; print(suggest_outfit(load_listings()[0], get_empty_wardrobe()))"
For an effortless, off-duty streetwear look, pair those vintage 501s with an oversized graphic tee tucked in loosely, a black leather belt, and retro white sneakers. Alternatively, lean into a classic vintage aesthetic by styling them with a tucked-in ribbed white tank top, an unbuttoned oversized flannel or corduroy overshirt, and well-worn leather boots.

```






```
$ python -c "from tools import create_fit_card; ..."

$ python -c "from tools import create_fit_card; from utils.data_loader import load_listings; print(create_fit_card('baggy jeans and white sneakers', load_listings()[0]))"
Found my holy grail medium wash vintage Levi's 501s and I’m literally never taking them off. Snagged them for just $38.0 on depop. Can't wait to style them oversized with some fresh white sneakers for that effortless 90s look.
```

```
$ python -c "from tools import create_fit_card; from utils.data_loader import load_listings; print(create_fit_card('   ', load_listings()[0]))"
No outfit suggestion was provided, so there's nothing to caption yet.
```

---

## How I Used AI

<!-- Two specific moments. What you asked, what came back, what you changed.

     "I used Claude to help me code" is not enough.

     "I gave Claude my search_listings spec. It returned None on no match
     instead of an empty list, so I changed it" is the level we want. -->

**Moment 1**

- *What I asked for:* Help filtering listings by size.

- *What came back:* A warning that a plain substring check is wrong i.e `"l" in
  "xl"` and `"s" in "us 7"` are both True — plus a token-match approach: split
  the size on `/`, spaces, and parens and match exact tokens.

- *What I changed:* Adopted token matching instead of substring, and wrote the
  rule into my Tool Inventory so criterion 1 could rely on it.

**Moment 2**

- *What I asked for:* To build `run_agent` following my branch rule.

- *What came back:* A loop that writes each result into the session and reads it
  back out, with the empty-search branch returning before `suggest_outfit`.

- *What I changed:* I'd pasted the query parser nested inside `run_agent`; I kept
  it there and wired the loop to call it, then moved it to module level and ran
  both example paths to confirm the impossible query stopped with `fit_card`
  still None.





<!-- ═══════════════════════ UNIT 4 — THE TEST ═══════════════════════

     Don't fill these in during unit 3.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Run Log — Before

<!-- Five criteria, five tries each, in this exact format.

     Five, because your criteria are written out of five. Mark each try PASS
     or FAIL, count the passes, and read that count against your target — a
     row targeting 4 of 5 with three PASS cells is MISSED (3/5).

     `python run_eval.py --label before` runs everything and writes the table
     into results/. Paste it here and fill in the verdicts. -->

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Real output from one try**, pasted as text, naming the file and function
that produced it:

```

```

---

## Verdicts and Diagnoses

<!-- MET or MISSED per criterion against LAST UNIT's target, plus a sentence on
     how you decided.

     Then, for every miss: which of the four places it happened — a tool, the
     loop's branch, the session, or the model's output — AND the mechanism.

     Not a diagnosis:  "The fit card was bad."
     A diagnosis:      "The fit card criterion missed on 2 of 5 items. Both had
                        an empty brand field. My prompt puts the brand in the
                        first sentence, so the card opened with a blank and read
                        like a fragment. The tool worked; the prompt assumed a
                        field that isn't always there."

     Look for a pattern. Three misses on the same tool is one problem, not
     three. -->

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 |  |  |  |  |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |

**Diagnoses**



---

## Loop Trace

<!-- One full run, printed step by step, with the MCP call visible in it.

     `python app.py ask '...' --trace` once you've added the trace.step()
     calls in Milestone 2.

     Worth pasting BOTH the happy path and the empty-search path. The empty
     one should be visibly shorter, because it stops. If your two traces are
     the same length, your branch isn't working — and this is the fastest way
     anyone will ever find that out. -->

**Happy path**

```

```

**Empty search**

```

```

**On the MCP move:** <!-- what changed in your code, and whether anything
behaved differently afterwards. If the rewire didn't work, say exactly where it
broke — the error text and the last thing that worked. That earns the point in
full. -->



---

## The Improvement

<!-- What you changed, why your diagnosis pointed at it, and the after-run in
     the same table format. One change, measured properly.

     `python run_eval.py --label after` -->

**What I changed:**

**Which failure it was meant to fix:**

### Run Log — After

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it made things worse, say that. Honestly reported, that earns full
     credit and is more interesting than one that worked. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped where
     you did. "I ran out of time" is fine if it's true. Pretending nothing is
     left is not. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 3

       [ ] criteria.md has five numbered criteria, each with a target
       [ ] Each criterion has a reason underneath it
       [ ] All five unit 3 sections above have real content
       [ ] Tool Inventory: all three tools, inputs WITH TYPES, a specific
           return value, and the empty case
       [ ] Planning Loop names the branch rule and agent.py::run_agent
       [ ] Sample Run: one full query plus the three per-tool tests, as text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN, you submit the same one
           next unit

     SUBMISSION CHECKLIST — unit 4

       [ ] mcp_server.py exists with one tool registered
           (or a written record of exactly where the rewire broke)
       [ ] Run Log — Before, five criteria, five tries each
       [ ] Real output pasted underneath, naming file and function
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, naming a place AND a mechanism
       [ ] Loop Trace, with the MCP call visible in it
       [ ] All three failure modes triggered and handled
       [ ] One improvement, with Run Log — After in the same format
       [ ] What's Still Broken
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository. Your commit history is what
     shows your criteria existed before your results did.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
