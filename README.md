# scrape-prj-sample
# Lead Scraping Workflow (n8n + Apify)

An n8n automation that takes a lead-search request from a form, runs it through
Apify scraper actors (Google Maps + Facebook), filters out leads with no
email, merges the two result sets, converts them to an Excel file, and emails
the file back to whoever submitted the form — via Gmail.

---

## How it works

```
                      On form submission
                             │
        ┌────────────────────┴────────────────────┐
        ▼                                          ▼
  Run an Actor                              Run an Actor2
  (Google Maps Scraper)                     (Facebook Search Scraper)
        │                                          │
  google maps data                          Get dataset items
  (pull Apify dataset, limit 25)            (pull Apify dataset, limit 25)
        │                                          │
  email filter with entries1                email filter with entries
  (keep rows only if "Your mail" was filled in the form)
        │                                          │
  Edit Fields2                              Edit Fields
  (map to: title, email)                    (map to: title, email)
        │                                          │
        └────────────────────┬────────────────────┘
                              ▼
                            Merge
                          (append)
                              │
                              ▼
                       Convert to File
                       (XLSX, output field: not named — default "data")
                              │
                              ▼
                          Send mail
                    (Gmail, subject: "Your leads are here,")
```

There is also a **disconnected branch** sitting in the canvas, not wired to
the form trigger at all: `Run an Actor 1` → `linkedIn Data` → `Edit Fields3`
→ `Convert to File3`. It's dead weight in the JSON but harmless — n8n just
won't execute it since nothing feeds into it. See **Known quirks** below for
why it was abandoned.

### Trigger: form fields

n8n Form Trigger ("On form submission"). Note the **exact internal field
names** below differ from their display labels — this matters if you ever
reference these fields in an expression (`$json['Country ']`, not
`$json.Country`):

| Display label | Internal field name (note exact spelling/spacing) | Required |
|---|---|---|
| Search Query | `Seacrh Query` *(typo, intentional — exists in the live workflow)* | Yes |
| City | `City` | Yes |
| Country | `Country ` *(trailing space)* | Yes |
| Mail | `Your mail ` *(trailing space)* | No (email type) |

### Apify actors used

- **Google Maps Scraper** — `console.apify.com/actors/WnMxbsRLNbPeYL6ge/input`
  — input: `searchStringsArray`, `locationQuery` (City, Country), `maxCrawledPlacesPerSearch: 25`
- **Facebook Search Scraper** — `console.apify.com/actors/Us34x9p7VgjCz99H6/input`
  — input: `categories` (search query), `locations` (City, Country), `resultsLimit: 25`
- **(Disconnected) LinkedIn Profile Search Scraper** — `console.apify.com/actors/M2FMdjRVeF1HPGFcc/input`

### Credentials referenced (not secrets — safe to commit)

The exported JSON only stores **credential names/IDs** that point to entries
in your own n8n credential store. The actual API token / OAuth secret is
never in this file:

- `apifyApi` → credential named **"Apify account"**
- `gmailOAuth2` → credential named **"Gmail account "**

When you import this into a different n8n instance, n8n will ask you to
re-map these to your own credentials.

---

## Setup

### Prerequisites
- An n8n instance (cloud or self-hosted)
- An [Apify](https://apify.com) account + API token
- A Gmail account connected to n8n via OAuth2

### Import the workflow
1. n8n → **Workflows** → **Import from File** → select `workflows/scrape-prj.json`
2. Open the **Run an Actor**, **Run an Actor2**, and **Send mail** nodes and
   assign your own Apify / Gmail credentials (they won't carry over).
3. Activate the workflow, open the form URL, and test it. While inactive
   you'll see a "This is a test version of your form" banner.

### Re-exporting after future edits
Workflow menu (`···`) → **Download** → overwrite `workflows/scrape-prj.json`
→ commit. Always do a quick search for `"token"` / `"apiKey"` / `"secret"`
in the diff before pushing, just in case a credential ever gets pasted
directly into a node parameter instead of the credentials system.


---

## Known quirks

- **Field name typos/spacing are load-bearing.** `Seacrh Query`, `City `,
  `Country `, and `Your mail ` (note spaces) are the literal keys every
  downstream node expression reads from. If you "fix" the typo or trim the
  spaces in the form node, every expression referencing the old key breaks
  silently (returns undefined, not an error).
- **The two branches use different limits**: Google Maps pulls up to 25
  places; the LinkedIn (disconnected) branch was capped at 5 — small leftover
  from testing.
- **Why the LinkedIn branch was removed**: most LinkedIn-scraping Apify
  actors return individual worker/profile details rather than clean
  company-level data — searching by company name, city, and country tends to
  surface similar/unrelated workers instead of the target companies. Most
  reliable ones also require an exact company name or LinkedIn URL rather
  than free-text search, and are paid (Apify's free tier gives roughly a
  3-day trial), which didn't fit this flow's free-text city/country pattern.
- **Output filename**: the `Convert to File` node doesn't set a custom binary
  field name, so the email always attaches a generically named XLSX.

📌 Note: This repo is a sample of a real-world automation project we
built — shared here to showcase the workflow design, tooling (n8n + Apify),
and integration patterns used in an actual production use case.
