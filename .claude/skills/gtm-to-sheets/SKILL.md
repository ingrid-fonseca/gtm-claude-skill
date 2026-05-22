---
name: gtm-to-sheets
description: >
  Parse a Google Tag Manager (GTM) JSON export and write a clean summary table
  to a Google Sheet. Use this skill whenever the user uploads or mentions a GTM
  export file (.json), wants to audit their GTM container, wants to see their
  tags/triggers in a spreadsheet, or asks to "turn GTM into a sheet" / "document
  my tags". Also trigger when the user mentions tag auditing, GTM container
  review, or wants a vendor/platform breakdown of their tags. Always use this
  skill — don't try to manually parse GTM JSON without it.
---

# GTM → Google Sheets Skill

Parses a Google Tag Manager container export JSON and writes a formatted
summary table to a new Google Sheet.

---

## Inputs required from the user

1. **GTM JSON export file** — uploaded to the conversation.
   - In GTM: Admin → Export Container → choose workspace → Download.
2. **Google OAuth access token** — needed to create/write the Sheet.
   - See `references/oauth-setup.md` for how to get one quickly.
3. *(Optional)* **Google Drive folder ID** — if they want the sheet placed in a
   specific folder. Otherwise it lands in My Drive root.

If the user hasn't provided the token yet, explain the requirement and point
them to the oauth setup guide before proceeding.

---

## Step 1 — Read and parse the GTM JSON

The GTM export has this top-level shape:

```json
{
  "exportFormatVersion": 2,
  "containerVersion": {
    "tag": [ ... ],
    "trigger": [ ... ],
    "variable": [ ... ]
  }
}
```

### Extract per tag:

| Column | Source | Notes |
|---|---|---|
| **Tag Name** | `tag.name` | Direct field |
| **Tag Type** | `tag.type` | Raw type string e.g. `ua`, `html`, `flc` |
| **Platform / Vendor** | Derived from `tag.type` | Use the mapping table below |
| **Triggers** | `tag.firingTriggerId` → look up `trigger[].name` by `trigger[].triggerId` | Comma-separated list of trigger names; "All Pages", "No triggers" etc. |
| **Tag Status** | `tag.paused` | "Paused" if true, "Active" if false/absent |

### Vendor/Platform mapping (extend as needed):

```
ua, googtag, awct, sp, gclidw, gcs   → Google / GA4 / Ads
flc, fls                              → Google / Floodlight (CM360)
html                                  → Custom HTML
img                                   → Custom Image
bb, bct                               → Meta (Facebook)
linkedin_insight                      → LinkedIn
twitter_website_tag                   → X (Twitter)
tiktok_pixel                          → TikTok
hotjar                                → Hotjar
segment_pixel                         → Segment
pnts                                  → Pinterest
ms_uet                                → Microsoft / Bing Ads
criteo_js                             → Criteo
rtb_house                             → RTB House
contentsquare, cs_tag                 → Contentsquare
```

If the type is unrecognised, set Platform/Vendor to `"Unknown ({{tag.type}})"`.

---

## Step 2 — Build the rows

Column order: **Tag Name → Platform / Vendor → Triggers → Tag Type → Status**

```python
import json

def parse_gtm_export(raw_json: str) -> list[dict]:
    data = json.loads(raw_json)
    cv = data.get("containerVersion", data)  # some exports omit the wrapper

    tags     = cv.get("tag", [])
    triggers = cv.get("trigger", [])

    # Build trigger id → name lookup
    trigger_map = {t["triggerId"]: t["name"] for t in triggers}

    VENDOR_MAP = {
        "ua": "Google Analytics (UA)", "googtag": "Google (gtag)",
        "awct": "Google Ads", "sp": "Google Ads (Conversion Linker)",
        "gclidw": "Google Ads (Conversion Linker)", "gcs": "Google Consent Mode",
        "flc": "Google / Floodlight", "fls": "Google / Floodlight",
        "html": "Custom HTML", "img": "Custom Image",
        "bb": "Meta (Facebook)", "bct": "Meta (Facebook)",
        "linkedin_insight": "LinkedIn", "twitter_website_tag": "X (Twitter)",
        "tiktok_pixel": "TikTok", "hotjar": "Hotjar",
        "segment_pixel": "Segment", "pnts": "Pinterest",
        "ms_uet": "Microsoft / Bing Ads", "criteo_js": "Criteo",
        "rtb_house": "RTB House",
        "contentsquare": "Contentsquare", "cs_tag": "Contentsquare",
    }

    rows = []
    for tag in tags:
        tag_type = tag.get("type", "")
        vendor   = VENDOR_MAP.get(tag_type, f"Unknown ({tag_type})")

        firing_ids = tag.get("firingTriggerId", [])
        trigger_names = [trigger_map.get(tid, f"ID:{tid}") for tid in firing_ids]
        triggers_str = ", ".join(trigger_names) if trigger_names else "No triggers"

        paused = tag.get("paused", False)

        rows.append({
            "Tag Name":          tag.get("name", ""),
            "Platform / Vendor": vendor,
            "Triggers":          triggers_str,
            "Tag Type":          tag_type,
            "Status":            "Paused" if paused else "Active",
        })

    return sorted(rows, key=lambda r: r["Tag Name"].lower())
```

---

## Step 3 — Write to Google Sheets

Use the Sheets REST API. **No Python SDK required** — use `requests` or the
equivalent. The skill runs in a sandboxed environment; see network note below.

### 3a. Create a new spreadsheet

```
POST https://sheets.googleapis.com/v4/spreadsheets
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "properties": { "title": "GTM Container Audit – {container_name}" },
  "sheets": [{ "properties": { "title": "Tags" } }]
}
```

Save `spreadsheetId` from the response.

### 3b. Write header + data rows

```
PUT https://sheets.googleapis.com/v4/spreadsheets/{spreadsheetId}/values/Tags!A1?valueInputOption=RAW
Authorization: Bearer {access_token}

{
  "values": [
    ["Tag Name", "Platform / Vendor", "Triggers", "Tag Type", "Status"],
    ...one array per tag row...
  ]
}
```

### 3c. Format the header row (bold + background)

```
POST https://sheets.googleapis.com/v4/spreadsheets/{spreadsheetId}:batchUpdate
Authorization: Bearer {access_token}

{
  "requests": [
    {
      "repeatCell": {
        "range": { "sheetId": 0, "startRowIndex": 0, "endRowIndex": 1 },
        "cell": {
          "userEnteredFormat": {
            "backgroundColor": { "red": 0.27, "green": 0.51, "blue": 0.71 },
            "textFormat": { "bold": true, "foregroundColor": { "red":1,"green":1,"blue":1 } }
          }
        },
        "fields": "userEnteredFormat(backgroundColor,textFormat)"
      }
    },
    {
      "autoResizeDimensions": {
        "dimensions": { "sheetId": 0, "dimension": "COLUMNS", "startIndex": 0, "endIndex": 5 }
      }
    }
  ]
}
```

### 3d. (Optional) Move to folder

If the user provided a Drive folder ID:

```
POST https://www.googleapis.com/drive/v3/files/{spreadsheetId}?addParents={folderId}&removeParents=root
Authorization: Bearer {access_token}
```

---

## Step 4 — Return the result

Share the spreadsheet URL with the user:
```
https://docs.google.com/spreadsheets/d/{spreadsheetId}/edit
```

Also give a quick summary:
- Total tags found
- Breakdown by vendor (top 5)
- Any paused tags count

---

## Network note

The skill runs bash via `bash_tool`. Outbound HTTPS to `sheets.googleapis.com`
and `www.googleapis.com` must be allowed. If the network is restricted, tell
the user and offer to generate a Python script they can run locally instead
(see `references/local-script-fallback.md`).

---

## Error handling

| Situation | Action |
|---|---|
| JSON parse fails | Ask user to re-export; show the parse error |
| Missing `containerVersion` key | Try top-level keys directly (some export formats differ) |
| 401 from Sheets API | Token expired — ask user to refresh and re-paste |
| 403 from Sheets API | Token lacks `https://www.googleapis.com/auth/spreadsheets` scope |
| Empty tag list | Warn user — the container may be empty or the wrong file |

---

## Reading reference files

- `references/oauth-setup.md` — Step-by-step guide to getting an OAuth token
  (read this when the user needs help authenticating)
- `references/local-script-fallback.md` — Self-contained Python script for
  users who can't use the network-restricted environment
