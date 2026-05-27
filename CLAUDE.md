# Tea Websites — Project Context for Claude

## What This Project Is

Two personal tea websites maintained by Jon (jondsamuelsai@gmail.com). This folder is the working directory for one of them: a tea inventory page hosted on GitHub Pages.

- **Live site:** https://jdsai-code.github.io/cedartea/
- **GitHub repo:** https://github.com/jdsai-code/cedartea (branch: `main`)
- **Original reference page:** https://sites.google.com/view/cedarteasdc/tea

---

## The Workflow

New tea orders arrive as forwarded emails from jonandyana@gmail.com (via Gmail MCP connector). The workflow is:

1. Check Gmail for forwarded order confirmation emails (search "forwarded" or "Tea: Fwd:")
2. Parse the order — identify tea names, quantities, prices
3. Look up each tea on the vendor's website to get weight/size, description, tasting notes
4. Add entries to `tea_database.json`
5. Regenerate `index.html` from the database using the Python generator (see below)
6. Push both files to GitHub via `git clone` + `git push` over HTTPS with PAT

When tea is consumed and runs out, update `status` to `"none"` and move/mark accordingly.

---

## Key Files

### `tea_database.json`
The source of truth for all tea inventory. Structure:
```
{
  "meta": { "lastUpdated": "YYYY-MM-DD", ... },
  "categories": {
    "traditional_black": { "label": "Traditional Black", "teas": [...] },
    "traditional_oolong": { "label": "Traditional Oolong", "teas": [...] },
    "green":              { "label": "Green", "teas": [...] },
    "flavored":           { "label": "Flavored Tea", "teas": [...] },
    "white":              { "label": "White Tea", "teas": [] },
    "decaf":              { "label": "Decaf Tea", "teas": [...] },
    "tea_bags":           { "label": "Tea Bags (if you must)", "teas": [...] },
    "archived":           { "label": "Archived Teas (previously available)", "teas": [...] }
  }
}
```

Each tea entry has these fields:
```json
{
  "id": "snake_case_id",
  "name": "Full Tea Name",
  "origin": "Country/Region",
  "vendor": "Vendor Name",
  "price_per_oz": 7.50,
  "price_per_cup": 0.60,
  "description": "...",
  "notes": "Personal tasting notes",
  "worth_ordering_again": false,
  "status": "in_stock",
  "quantity_notes": "2 oz ordered March 2026",
  "history": [
    { "date": "YYYY-MM-DD", "change": "added to inventory", "note": "..." }
  ]
}
```

**Tea bags** use `"price_per_bag"` instead of `"price_per_oz"` / `"price_per_cup"`.

**Status values:** `in_stock`, `low`, `none`, `archived`

### `index.html`
Generated from `tea_database.json` by running the Python generator (inline in bash). Do NOT hand-edit — always regenerate from the database.

**Page structure:**
- Hero banner: `images/topbanner.png` with `rgba(0,0,0,0.22)` dark overlay
- Header text (DO NOT CHANGE): **"Tea Selection"** / **"See What You're Feeling, What You're Not"**
- Legend bar: ★ Worth Ordering Again | Running low tag
- One section per category, each with a float-left image (340×220px) and tea list
- Archived section at the bottom — full entry format, NO strikethrough

**Section images** (in `images/`):
`blacktea.png`, `oolong.png`, `greentea.png`, `flavored.png`, `whitetea.png`, `decaf.png`, `teabags.png`, `archivedtea.png`, `topbanner.png`

**Each tea entry displays:** Name ★ (if worth ordering) → Origin (italic) → Vendor → Price → Description → Notes

**Empty sections** (e.g. White Tea) show: *"No current offerings — check back soon"*

---

## Python HTML Generator

Run this inline in bash to regenerate `index.html`. Key logic:

```python
import json
with open('/sessions/.../mnt/Tea Websites/tea_database.json') as f:
    db = json.load(f)

def tea_item(tea):
    star = ' <span class="star">&#9733;</span>' if tea.get('worth_ordering_again') else ''
    if 'price_per_oz' in tea:
        price = f"${tea['price_per_oz']:.2f}/oz (about ${tea['price_per_cup']:.2f} per cup)"
    elif 'price_per_bag' in tea and tea['price_per_bag']:
        price = f"${tea['price_per_bag']:.2f}/bag"
    else:
        price = ''
    qty_tag = ' <span class="tag low">Running low</span>' if tea.get('status') == 'low' else ''
    origin = f'\n        <span class="origin">{tea["origin"]}</span>' if tea.get('origin') else ''
    vendor = f'\n        <span class="vendor">{tea["vendor"]}</span>' if tea.get('vendor') else ''
    price_html = f'\n        <span class="price">{price}</span>' if price else ''
    desc = f'\n        <span class="desc">{tea["description"]}</span>' if tea.get('description') else ''
    notes = f'\n        <span class="notes">Notes: {tea["notes"]}</span>' if tea.get('notes') else ''
    return f'<li>\n        <span class="tea-name">{tea["name"]}{star}{qty_tag}</span>{origin}{vendor}{price_html}{desc}{notes}\n      </li>'
```

Archived teas use the same `tea_item()` function — no strikethrough, full info shown.

---

## Git Push Workflow

The bash sandbox loses `/tmp` between sessions. Re-clone every time:

```bash
cd /tmp && rm -rf cedartea_clone
git clone https://YOUR_GITHUB_PAT@github.com/jdsai-code/cedartea.git cedartea_clone
cp "/sessions/.../mnt/Tea Websites/index.html" /tmp/cedartea_clone/index.html
cp "/sessions/.../mnt/Tea Websites/tea_database.json" /tmp/cedartea_clone/tea_database.json
cd /tmp/cedartea_clone
git config user.email "jondsamuelsai@gmail.com"
git config user.name "JonforAI"
git add index.html tea_database.json
git commit -m "..."
git push https://YOUR_GITHUB_PAT@github.com/jdsai-code/cedartea.git main
```

**Note:** GitHub REST API calls are blocked in the bash sandbox (403). Browser `fetch()` also hangs. Only `git clone/push` over HTTPS works.

---

## Price Calculations

- **Loose leaf:** `price_per_oz = price / weight_oz`, `price_per_cup = price / (weight_g / 2)` (approx 2g per cup)
- **100g = 3.53oz**, **50g = 1.76oz** (use 3.5oz / 1.7oz as Teabox lists them)
- Use the price the customer actually paid (from the order email), not the current website price

---

## Known Issues / Rules

- Never change the header text — it is always "Tea Selection" / "See What You're Feeling, What You're Not"
- Always regenerate `index.html` from `tea_database.json`; never hand-edit the HTML
- The JSON can truncate silently on large edits — always verify with `python3 -c "import json; json.load(open('tea_database.json'))"` after editing
- If the file is truncated, use the Read tool (which tracks in-memory state) to recover the missing tail and append it via bash

---

## Gmail MCP Connector

Tool prefix: `mcp__931fb4bb-d3b1-4025-90c0-449e0e4582aa__*`

Search for forwarded order emails with queries like:
- `"forwarded" subject:Tea in:inbox`
- `subject:"Fwd:" from:jonandyana@gmail.com`
