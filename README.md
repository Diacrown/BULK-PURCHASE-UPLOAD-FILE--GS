# GS BULK PI

Gemstones Purchase Import — Bulk File Generator.

A single-file, no-backend HTML tool. Drop in a raw **WSL PL Import** export,
it corrects it in the browser, and you download the fixed `.xlsx` straight
back out. Nothing is uploaded anywhere — all parsing happens client-side via
SheetJS.

## What it does

On every real line-item row (rows with a `Pos` value — trailing grand-total
rows are dropped from the output):

1. **SBK** is forced to `A`.
2. **Forwarding PC** is forced to match **SSP** (the whole cell is copied so
   type/format match exactly).
3. **Item # renaming**, only for rows where `IT` (Item Type) = `PP`:
   - Skipped (left as the original Item#) if `Ref #` or `Lot#` contains a
     name — that marks the line as reserved for an overseas client.
   - A bare `STOCK` or `STK` in `Ref #`/`Lot#` does **not** count as a
     client name — those rows still get renamed as ordinary stock.
   - Otherwise, renamed to `{Type}{ShapeCode}{Col}{MeasurementCode}`:
     - **Shape code** — looked up from the standard shape table (see
       `SHAPE_CODES` in `index.html`). `OCT` (octagonal) is coded as
       **Emerald (15)**, not Octagonal (43) — that's a deliberate gemstone-
       industry convention, not a table lookup bug.
     - **Measurement code**, parsed from `Remarks`:
       - Round shapes: first number in `Remarks`, decimal point stripped,
         padded/truncated to exactly 2 digits (`2` → `20`, `2.5` → `25`,
         `6.35` → `63`).
       - Non-round shapes: integer part of the first two numbers in
         `Remarks`, concatenated (`7 X 5 MM` → `75`).
     - If the shape code or measurement can't be resolved, the row is left
       unrenamed and flagged **needs review** rather than guessed at.
4. **Format normalization.** Some raw exports carry a real number/date
   format on only one row per column and leave the rest as `General` — which
   is how a date cell round-trips as a bare serial number instead of a date.
   For every column, the tool finds whichever row already has a proper
   format and applies it to every other row in that column.
5. **2-decimal rounding.** Every non-date numeric value is rounded to 2
   decimal places (the stored value, not just the display), so nothing like
   a 12-decimal-place figure makes it into the ERP feed.
6. **Manual override.** Every row's Item# in the results table is editable
   before you download — click in, type a replacement, it's written straight
   into the workbook cell. Edited rows are marked **edited** and counted
   separately from auto-renamed ones.
7. **History (sign-in required).** Every downloaded output is saved
   automatically, searchable by invoice number (`Txn #`, e.g. `TSI00047`) —
   see below.

Cells the tool doesn't touch are left completely byte-identical — it edits
specific cells in place rather than rebuilding the sheet, so things like
column widths and every other column's data are untouched.

## Using it

Just open `index.html` in a browser (locally, or hosted anywhere static).
Drop in the raw file, hit **Process file**, review the per-row status table
(edit any Item# you want to override by hand), then **Download corrected
file**.

## History — setup (optional)

Off by default. The tool works fully without it — history just won't save
or be searchable until you wire up Firebase. Same pattern as the
customs-clearance composer: Firestore and Auth's REST APIs called directly
with `fetch`, no SDK, so the tool stays a single HTML file.

**Reuse the customs-composer Firebase project, or start fresh — either
works.** Reusing means one project's free-tier quota covers both tools
instead of splitting it across two; a fresh project keeps them fully
separate. Your call.

1. **Firebase console** → (existing project, or **Add project** → skip
   Analytics → Create).
2. **Build → Firestore Database** → Create database if this is a new
   project (test mode is fine to start).
3. **Build → Authentication → Get started → Sign-in method** → enable
   **Email/Password**.
4. **Authentication → Users → Add user** → `diacrownmedia@gmail.com` and a
   password of your choice. That's the one account this tool signs in as —
   the password lives only in your head and Firebase, never in the code.
5. **Firestore → Rules** — add this collection to whatever's already there
   (don't replace an existing rule block if reusing a project). This
   requires a signed-in request, not just anyone with the API key:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /outputs/{docId} {
         allow read, write: if request.auth != null;
       }
       // keep any other existing match blocks here too
     }
   }
   ```
   Publish.
6. **Project settings → General** → copy the **Project ID**, and the
   **Web API Key** from "Your apps" (add a Web app if there isn't one yet).
7. Paste both into `index.html`, near the top of the `<script>` block:
   ```js
   const FIREBASE_PROJECT_ID = "your-project-id";
   const FIREBASE_API_KEY = "your-web-api-key";
   ```
8. Push. The History panel now shows a sign-in form — sign in with
   `diacrownmedia@gmail.com` and the password from step 4.

### What History gives you

- **Auto-save on download** — every corrected file is stored (as base64,
  inside the same Firestore doc — well within the free tier's 1&nbsp;MiB
  document limit for files this size) alongside the invoice number, a
  timestamp, and the renamed/skipped/unchanged/review/edited counts. Saving
  never blocks or delays the actual file download — if Firestore is
  unreachable or you're signed out, you still get your file, just with a
  quiet "history save failed" note.
- **Search by invoice number** — type any part of a `TSI...` number to find
  every past run for it.
- **Re-download** any past output straight from history, byte-identical,
  no reprocessing needed.
- **Export CSV** — invoice number, date, filename, and stats for every
  entry currently listed (respects an active search), for record-keeping
  outside Firestore.
- **Delete** old entries you don't need kept.
- **Duplicate-invoice heads-up** — if you process a file whose invoice
  number already has a saved run, you'll see a banner before you download,
  so a reprocess doesn't happen silently.

### How the sign-in works

Firebase's Identity Toolkit REST API, same no-SDK approach as Firestore.
Signing in exchanges the email/password for a short-lived ID token, which
gets sent as an `Authorization: Bearer` header on every Firestore request —
that's what the `request.auth != null` rule above checks. The session is
kept in `sessionStorage` (cleared when the tab closes, never written to
disk) and refreshed automatically while it's open. Nobody without that
password can read, add, or delete anything in the `outputs` collection, even
though the Project ID and API key are visible in the page's source — those
were never the actual access control.

## Deploying

**GitHub Pages**
1. Push this repo to GitHub.
2. Repo Settings → Pages → Deploy from branch → `main` / root.
3. It'll be live at `https://<user>.github.io/<repo>/`.

## Not included (ask if you want any of these)

Kept out to avoid overbuilding a tool that's meant to stay a single file:

- **Multiple accounts / roles** — one shared sign-in
  (`diacrownmedia@gmail.com`), not per-person accounts or permission levels.
- **Separate file storage** (e.g. Firebase Storage / S3) — output files
  live inside the Firestore doc itself. Fine at this file size; would need
  revisiting if outputs ever got much larger (tens of MB).

## Notes

- Only processes the first sheet in the workbook.
- Expects the standard WSL PL Import headers (`Pos`, `IT`, `SBK`, `Type`,
  `Shape`, `Ref #`, `Remarks`, `Col`, `Item#`, `Lot#`, `SSP`,
  `Forwarding PC`) — if any are missing it'll show an error naming which
  ones, rather than guessing. `Txn #` is used for History but isn't
  required if History is off.
- With History off (the default), no data leaves the browser at all.
