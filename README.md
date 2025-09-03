# OneTrust Domain Group Manager (Google Sheets + Apps Script)

This Google Apps Script adds a custom menu to Google Sheets for managing [OneTrust Cookie Manager](https://onetrust.com/) domain groups.

You can:
- Create or update domain groups in OneTrust directly from a Google Sheet.
- Check existing mappings for a domain via the **CDN `domain-list.json` endpoint**.
- Report mapped domains into a dedicated sheet (`domainCheck`).

---

## 📋 Prerequisites

- A Google account with access to Google Sheets.
- Access to a OneTrust tenant and API credentials (`CLIENT_ID`, `CLIENT_SECRET`) **or** a static `BEARER_TOKEN`.
- Basic familiarity with [Apps Script](https://developers.google.com/apps-script).

---

## 🛠️ Setup Instructions

### 1. Create the Google Sheet
1. Go to [Google Sheets](https://sheets.google.com/).
2. Create a new sheet.
3. Rename the first worksheet to **`DomainGroups`** (case-sensitive).
   - Column A: Main Domain (hostname, optional)
   - Column B: Domain ID (UUID, optional if hostname is provided)
   - Column C: Group Domains (comma, semicolon, or newline-separated list)
   - Column D: Status (script will write results here)

Example:

| Main Domain   | Domain ID                              | Group Domains                | Status   |
|---------------|----------------------------------------|------------------------------|----------|
| example.com   |                                        | foo.com, bar.com, baz.com    |          |
|               | 019907fa-ba11-72b4-a8b3-83d0262bb8ae   | another.com                  |          |

---

### 2. Add the Script
1. In your sheet, click **Extensions → Apps Script**.
2. Paste the full script (`Code.gs`) from this repo into the editor.
3. Save the project with a name like `OneTrust Tools`.

---

### 3. Add Secrets
The script looks for your OneTrust credentials in **Script Properties**.

1. In the Apps Script editor, click **Project Settings (⚙️)** → **Script Properties**.
2. Add one of the following:

- **Option A (Static Bearer Token):**
  - Key: `OT_BEARER_TOKEN`
  - Value: *(your token string)*

- **Option B (OAuth Client Credentials):**
  - Key: `OT_CLIENT_ID`
  - Value: *(your client id)*
  - Key: `OT_CLIENT_SECRET`
  - Value: *(your client secret)*

If both are present, the script prefers `OT_BEARER_TOKEN`.

---

### 4. Authorize the Script
1. Back in the Apps Script editor, run any function once (e.g., `validateSetup`).
2. Grant the required Google permissions.

---

## 🚀 Usage

Once installed, your sheet will have a new menu:

**OneTrust Tools**
- **Validate Setup** → Check sheet structure and API credentials.
- **Create Domain Groups** → Push data from the `DomainGroups` sheet into OneTrust.
- **Check Domain Group (prompt)** → Enter a Domain ID (UUID) *or* hostname.  
  The script fetches `https://cdn.cookielaw.org/consent/<domainId>/domain-list.json` and writes results to the `domainCheck` sheet.

---

## 📝 Report Sheet

The script creates a second worksheet named **`domainCheck`** for reporting mapped domains.

- Column A: `domainID`
- Column B: `Mapped Domains` (comma-separated list)

Each run appends a new row.

---

## ⚙️ Config Options

At the top of the script:

```js
const SHEET_NAME = 'DomainGroups';    // Data sheet name
const STATUS_COL = 4;                 // Column for status messages
const REPLACE_EXISTING = true;        // true = overwrite groups, false = append
const REPORT_SHEET = 'domainCheck';   // Report output sheet
