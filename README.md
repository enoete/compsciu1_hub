# CS Unit 1 — course hub, topics, and the sync pipeline

Three repos, one pipeline, no API keys in any published file.

```
compsciu1_hub/   the hub students land on
compsciu1_1/     Topic 1 — Storage Devices & Media
compsciu1_2/     Topic 2 — Logic Gates & Truth Tables
setup.sh         creates/updates all three on GitHub
seed-bin.json    the starting contents for your JSONBin
```

---

## 1. Run it

```bash
chmod +x setup.sh
./setup.sh
```

You need the [GitHub CLI](https://cli.github.com) logged in (`gh auth login`). The script asks for the
JSONBin key once, with hidden input, and hands it straight to GitHub as an encrypted secret. It never
writes the key to a file.

What it does, per repo: creates it if missing, pushes the contents, turns on Pages. Then for the hub
only: grants Actions write permission, stores `JSONBIN_KEY` and `JSONBIN_BIN_ID` as secrets, optionally
seeds the bin, and triggers the first sync.

> **It force-pushes.** `compsciu1_1` and `compsciu1_2` already exist and their current contents will be
> replaced. The script makes you type `yes` first. If you want the old versions kept, rename those repos
> on GitHub before running it.

When it finishes:

| | |
|---|---|
| Hub | `https://enoete.github.io/compsciu1_hub/` |
| Topic 1 | `https://enoete.github.io/compsciu1_1/` |
| Topic 2 | `https://enoete.github.io/compsciu1_2/` |

First publish takes a minute or two.

---

## 2. Rotate the key

The master key was pasted into a chat, and a JSONBin master key grants full read/write/delete across
every bin on your account. Once the pipeline runs, generate a fresh key in the JSONBin dashboard and
update the secret:

```bash
gh secret set JSONBIN_KEY --repo enoete/compsciu1_hub
```

Better still, create a scoped **Access Key** with read permission on this bin only, then:

```bash
gh variable set JSONBIN_KEY_TYPE --repo enoete/compsciu1_hub --body access
```

The sync script switches to the `X-Access-Key` header automatically.

---

## 3. How the metadata gets there

This is the part that removes the API-key problem entirely.

```
JSONBin ──► GitHub Action ──► fetches each topic URL ──► data/content.json ──► the hub
  (list      (has the key,      (reads og: and              (committed,          (plain fetch,
   of URLs)   runs on a          lesson: meta tags)          public, static)      no key)
              schedule)
```

Each lesson page carries its own manifest in its `<head>`:

```html
<meta property="og:title"       content="Logic Gates &amp; Truth Tables">
<meta property="og:description" content="The five-step drill for turning any truth table…">
<meta name="lesson:duration"    content="75 min">
<meta name="lesson:difficulty"  content="Core">
<meta name="lesson:questions"   content="30">
<meta name="lesson:sections"    content="9">
<meta name="lesson:topics"      content="AND,OR,NOT,NAND,NOR,XOR,XNOR,Combinational circuits">
```

The sync script reads those, so the hub card for a topic describes itself correctly with no extra work
from you. Add a topic URL to the bin and the card writes itself.

Anything you set explicitly in the bin (`title`, `blurb`, `tags`, `accent`) overrides what was scraped.
If a page is temporarily unreachable, the previous metadata is kept rather than blanked.

The Action runs every six hours, on every push, and whenever you click **Run workflow** in the Actions tab.

---

## 4. Adding a topic

Open `compsciu1_hub/admin.html` **on your own machine** — it is in `.gitignore` and is never published.
Load the bin, add an entry, save:

```json
{
  "id": "number-systems",
  "number": 3,
  "url": "https://enoete.github.io/compsciu1_3/",
  "status": "live",
  "accent": "#5A4B9C"
}
```

`status` accepts `live`, `soon` or `locked`. Non-live topics render greyed out and unclickable, which
is handy for showing students what is coming without opening it yet.

Then either wait for the schedule or run the workflow manually.

---

## 5. Reusing this for your other classes

Copy `compsciu1_hub/` to a new folder and change two things:

1. `config.json` — `code`, `title`, `subtitle`, `tagline`, `accent`
2. The `JSONBIN_BIN_ID` secret in the new repo — point it at a new bin

Then `HUB_REPO=appliedmaths_hub BIN_ID=<new-bin-id> ./setup.sh`.

The lesson pages are self-contained single files. Copy either one as a starting point for a new topic;
the only things that must change are the `<head>` manifest and the content arrays near the top of the
`<script>` block.

---

## 6. What is in each lesson

**Topic 1 — Storage Devices & Media** (55 min, 25 test questions)

Opens with a latency ladder: if a register read took one second, tape would take four thousand years.
Then memory types, a volatile/non-volatile drag-and-drop sort, the storage hierarchy explorer, a live
seek race comparing serial, direct and random access, a unit converter with a capacity calculator,
an input/output/storage sort, and computer classifications.

**Topic 2 — Logic Gates & Truth Tables** (75 min, 30 test questions)

The centrepiece is the circuit builder. Students work through four exercises using one repeatable drill:

> **SCAN** the output column for 1s → **MARK** each row's literals (1 = plain, 0 = primed) →
> **WRITE** each row as an AND term → **SUM** the terms with OR → **DRAW** inverters, AND gates, one OR gate.

Each step is locked until the previous one is correct, and every wrong answer gets a targeted
explanation rather than a red cross. At the final step the page builds the schematic from their wiring,
evaluates it across all rows, and shows it beside the target table so they can see for themselves that
it matches.

Also included: a live gate bench, all seven gates with clickable truth tables, a truth-table dojo
(seven expressions to fill in), circuit-reading practice, and the Spot the Bug panel below.

---

## 7. About the textbook error

Page 22, Example 1 gives `R = 'A.C + D` with a truth table beside it. The table is wrong — it is the
table for `A'·C` alone, with `D` ignored. Rows `001`, `101` and `111` all have D = 1 and should output 1;
the book prints 0 for all three.

Rather than reproduce that, the lesson has a **Spot the Bug** section that shows the printed table and
asks students to find the bad rows, then explains what happened. Builder exercise 4 is the corrected
version of the same problem, so they finish by building the circuit the book was reaching for.

If you would rather match the book exactly, delete the `<section id="bug">` block and its `BUG` object
in the script.

---

## 8. If something misbehaves

**Hub says content hasn't been generated** — the Action hasn't run. Actions tab → *Sync course content*
→ Run workflow. If it fails on a 401, the key or bin ID secret is wrong.

**Action fails at the commit step** — repo Settings → Actions → General → Workflow permissions →
*Read and write permissions*. The script tries to set this, but organisation policy can override it.

**A topic card shows the right title but no duration or tag chips** — that page is missing its
`lesson:*` meta tags. Copy the manifest block from either lesson's `<head>`.

**Pages shows a 404 after setup** — give it two minutes, then check Settings → Pages is set to
branch `main`, folder `/ (root)`.
