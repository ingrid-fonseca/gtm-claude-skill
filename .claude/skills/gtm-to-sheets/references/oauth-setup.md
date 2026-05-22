# Getting a Google OAuth Access Token (Quick Method)

To write to Google Sheets, the skill needs a short-lived access token tied to
your Google account. The fastest way is via Google's OAuth 2.0 Playground —
no code required.

---

## Option A — OAuth 2.0 Playground (recommended, ~2 minutes)

1. Go to: https://developers.google.com/oauthplayground/
2. In the top-right gear icon → check **"Use your own OAuth credentials"**
   (or leave unchecked to use Google's default client — fine for personal use).
3. In the left panel, find and expand **"Google Sheets API v4"**.
   Select the scope: `https://www.googleapis.com/auth/spreadsheets`
   Also select from **"Drive API v3"**: `https://www.googleapis.com/auth/drive.file`
   (needed to move the sheet to a folder, if you want that).
4. Click **"Authorize APIs"** → sign in with your Google account.
5. Click **"Exchange authorization code for tokens"**.
6. Copy the **Access Token** value.
7. Paste it into the conversation.

> **Tokens expire after ~1 hour.** If you get a 401 error, just repeat
> steps 4–6 to get a fresh one.

---

## Option B — gcloud CLI (if you have it installed)

```bash
gcloud auth login
gcloud auth print-access-token
```

Copy the printed token and paste it into the conversation.

---

## Option C — Long-term setup (for repeated use)

If you want to run this regularly without re-authenticating each time, set up a
**Service Account** in Google Cloud Console with the Sheets API enabled and
share your Drive with the service account email. This is out of scope for the
skill but documented at:
https://developers.google.com/identity/protocols/oauth2/service-account
