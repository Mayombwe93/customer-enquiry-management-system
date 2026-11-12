# Customer Enquiry Management System: Setup, Templates, Testing

All names, emails and phone numbers in this guide are fake.

---

## 1. Workflow overview

Google Form → Google Sheet (new row) → n8n polls every minute → generates `ENQ-YYYYMMDD-001` → validates → writes ID and `New` status → emails customer → messages owner on Telegram → marks Sheet `Yes / Yes`. Any failure → Telegram alert (step + enquiry ID) and row marked `Needs Review`.

Timezone: all dates and IDs use **Africa/Nairobi (EAT)**.

### Node list (21 nodes)

| # | Node | Type | Purpose |
|---|---|---|---|
| 1 | New Enquiry Row | Google Sheets Trigger | Fires on a new row (polls every minute) |
| 1b | Config | Set | Holds your Sheet ID, tab name, chat IDs, business name and owner name |
| 2 | Get Existing IDs | Google Sheets (Get Row(s)) | Reads the sheet to find today's highest ID and each row's number |
| 3 | Generate Enquiry ID | Code | Builds the ID, cleans fields, validates, flags `valid` true/false, skips rows that already have an ID |
| 4 | Is Data Valid? | IF | True → continue. False → error branch |
| 5 | Write ID and Status | Google Sheets (Update Row) | ID, Date Received, Status = New, Ack = No, Notified = No |
| 6 | Send Acknowledgement | Gmail | Customer email |
| 7 | Mark Acknowledged | Google Sheets (Update Row) | Acknowledgement Sent = Yes |
| 8 | Notify Owner | Telegram | Owner message |
| 9 | Mark Owner Notified | Google Sheets (Update Row) | Owner Notified = Yes |
| 10-16 | Err: … (7 nodes) | Set | Name the failed step and enquiry ID |
| 17 | Alert Admin | Telegram | Failure alert |
| 18 | Mark Needs Review | Google Sheets (Update Row) | Status = Needs Review |
| 19-20 | Sticky notes | Sticky Note | On-canvas documentation |

---

## 2. Setup checklist (beginner)

### A. Create the Google Form
1. Go to forms.google.com → **Blank form**. Title: *Contact Us / Enquiry Form*.
2. Add these questions, all marked **Required**:
   - **Full Name** (Short answer)
   - **Email** (Short answer; Response validation → Text → Email)
   - **Phone** (Short answer)
   - **Service Needed** (Dropdown: add your real services, e.g. *Deep Cleaning, Regular Cleaning, Office Cleaning, Other*)
   - **Message** (Paragraph)
3. Keep the question titles **exactly** as written. n8n reads them as column names.

### B. Create the Google Sheet
1. In the Form, open **Responses** → **Link to Sheets** → create a new spreadsheet. Name it *Customer Enquiries*.
2. Keep the tab name **Form Responses 1** (if your Google account is in another language, the name may differ; the workflow's tab name must match it).
3. Columns A to F are created by Google: `Timestamp, Full Name, Email, Phone, Service Needed, Message`. Do not move them.
4. Add these headers in G1 to L1, **spelled exactly**:

   `Enquiry ID | Date Received | Status | Acknowledgement Sent | Owner Notified | Last Updated`

   (Google Forms controls columns A to F, so your extra columns sit to the right. The order differs slightly from the original spec, but all columns exist.)
5. Add dropdowns: select column I (Status) → **Data → Data validation → Dropdown**: `New, Contacted, Quoted, Won, Lost, Needs Review`. Select columns J and K → dropdown: `Yes, No`.
6. Select column D (Phone) → **Format → Number → Plain text**, so leading zeros and "+" are kept.
7. Copy the **Sheet ID** from the URL: `docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit`.

### C. Google credentials in n8n (Sheets and Gmail)
- **n8n Cloud:** in a Google node, click *Create new credential* → **Sign in with Google**. Done.
- **Self-hosted:**
  1. console.cloud.google.com → create a project.
  2. **APIs & Services → Library**: enable **Gmail API** and **Google Sheets API**.
  3. **OAuth consent screen**: External; add your Gmail address as a test user. Tip: click **Publish app**, otherwise your login expires every 7 days.
  4. **Credentials → Create credentials → OAuth client ID → Web application**. Paste the *OAuth Redirect URL* shown in the n8n credential screen into *Authorized redirect URIs*.
  5. Copy the Client ID and Client Secret into n8n, then click **Sign in with Google**.
- You need three credential entries in n8n (the Trigger node uses its own type): *Google Sheets OAuth2*, *Google Sheets Trigger OAuth2*, *Gmail OAuth2*. All use the same Google account.

### D. Telegram bot
1. In Telegram, open **@BotFather** → send `/newbot` → follow prompts → copy the **bot token**.
2. Open your new bot and press **Start** (a bot cannot message you until you do).
3. Get your **chat ID**: message **@userinfobot**, which replies with your numeric ID.
4. In n8n: **Credentials → New → Telegram API** → paste the token.
5. If the owner is a different person, they must also press **Start** on the bot, and you use their chat ID in *Notify Owner*.

### E. Import and configure
1. n8n -> **Workflows -> menu (three dots) -> Import from file** -> choose `customer-enquiry-management-system.json`.
2. Open each Google Sheets, Gmail and Telegram node and pick your saved credential from the **Credential to connect with** dropdown (each node has a note on the canvas telling you what to do).
3. Open **New Enquiry Row** and paste your Sheet ID into *Document* (By ID). Check the sheet tab is **Form Responses 1**.
4. Open the **Config** node and fill the 6 values: Sheet ID (same as step 3), tab name, owner chat ID, admin chat ID, business name, your name. This is the only other place you edit.
5. Click **Execute workflow**, add a test row, and watch it run. When happy, switch the workflow to **Published/Active** (top right). The trigger only runs while the workflow is active.

---

## 3. Message templates

### Acknowledgement email (plain text, already in the workflow)

**Subject:** We have received your enquiry (ENQ-20261103-001)

```
Dear Amina Test,

Thank you for contacting Your Business Name. We have received your enquiry about Deep Cleaning and recorded it under reference ENQ-20261103-001.

A member of our team will respond to you within 24 hours.

If your request is urgent, please reply to this email and include your phone number so we can call you.

Kind regards,
Your Name
Your Business Name
```

### Telegram message to owner

```
🆕 NEW ENQUIRY: ENQ-20261103-001

Name: Amina Test
Phone: +254 700 000 001
Service: Deep Cleaning

Message:
Hello, I need a quote for cleaning a 3-bedroom house.

Status: New. Customer has been emailed.
```

(Messages over 500 characters are cut with "… (full text in Sheet)". The Sheet always keeps the full text.)

### Telegram failure alert

```
⚠️ ENQUIRY WORKFLOW ALERT

Failed step: Send Acknowledgement (Gmail)
Enquiry ID: ENQ-20261103-004
Customer: Amina Test | Phone: +254 700 000 001
Reason: <error message>

The row is marked "Needs Review" where possible. Please check this enquiry manually.
```

---

## 4. Test plan (10 fake enquiries)

**How to run:** Use the Form for tests 1, 2, 3, 8, 9. The Form blocks blank fields, so for tests 4 to 7, **paste a complete row into the Sheet in one action** (copy a row of 6 cells and paste into A:F of the next empty row). Typing cell by cell can trigger n8n before the row is complete. Wait about 1 to 2 minutes after each test. Replace `DATE` with today's date as `YYYYMMDD` (Nairobi time).

| # | Test | Input (fake) | Expected result | Actual result | Pass / Fail |
|---|---|---|---|---|---|
| 1 | Valid enquiry | Amina Test / amina.test@example.com / +254 700 000 001 / Deep Cleaning / "I need a quote for a 3-bedroom house." | ID `ENQ-DATE-001`; Status New; Ack = Yes; Notified = Yes; email received; Telegram received | | |
| 2 | Second valid enquiry, same day | Brian Sample / brian.sample@example.com / +254 700 000 002 / Regular Cleaning / "Weekly cleaning for my flat." | ID `ENQ-DATE-002`; all steps succeed | | |
| 3 | Valid, Phone blank (only possible if Phone is not Required) | Carol Demo / carol.demo@example.com / (blank) / Office Cleaning / "Please send prices." | Passes; Telegram shows "Phone: Not provided"; Ack = Yes; Notified = Yes | | |
| 4 | Blank Full Name | (blank) / dan.fake@example.com / +254 700 000 004 / Deep Cleaning / "Need help." | Fails validation; Status = Needs Review; Telegram alert "Validation", reason "Full Name is empty"; no email; no owner message | | |
| 5 | Blank Message | Eve Placeholder / eve.placeholder@example.com / +254 700 000 005 / Other / (blank) | Fails validation; Needs Review; alert reason "Message is empty" | | |
| 6 | Blank Email | Frank Mock / (blank) / +254 700 000 006 / Regular Cleaning / "Call me please." | Fails validation; Needs Review; alert reason "Email is empty"; no email sent | | |
| 7 | Invalid email | Grace Invalid / grace.invalid@ / +254 700 000 007 / Deep Cleaning / "Quote please." | Fails validation; Needs Review; alert reason "Email is not valid" | | |
| 8 | Very long message (about 3,000 characters; paste any dummy text) | Henry Longtext / henry.longtext@example.com / +254 700 000 008 / Other / (long text) | Succeeds; Sheet holds full text; Telegram shows first 500 characters then "… (full text in Sheet)"; email sent; no Telegram length error | | |
| 9 | Special characters | Iris O'Neil-Test / iris.test@example.com / +254 700 000 009 / Deep Cleaning / `Tom & Jerry <b>bold</b> "quotes" 😀 100%` | Succeeds; text appears as typed in email and Telegram; no formatting errors | | |
| 10 | Two submissions in the same minute | Submit two valid forms back to back (Jack Rapid / Kim Rapid, fake emails) | Both processed; two different consecutive IDs (e.g. `-009`, `-010` if earlier tests ran); no duplicate IDs | | |

**Optional failure test (recommended for the demo):** disconnect the Gmail credential (or temporarily set a wrong credential), submit a valid enquiry. Expected: Telegram alert "Send Acknowledgement (Gmail)" with the enquiry ID, Status = Needs Review, and the alert still shows the customer's name and phone so the owner can follow up manually.

---

## 5. Common errors and fixes

| Error / symptom | Likely cause | Fix |
|---|---|---|
| Workflow never fires | Workflow not Published/Active; or row added while it was off | Switch it to Active. Rows added while inactive are skipped; submit a new test |
| Fires on a half-filled row | Row typed cell by cell | Paste the full row at once; Forms always writes complete rows |
| "Node does not have any credentials set" | Credentials not selected after import | Open the node, choose or create the credential |
| Google "Access blocked / app not verified / access_denied" | Your Google account is not a test user | Cloud Console → OAuth consent screen → add your email as test user |
| Works, then fails a week later ("invalid_grant", "token expired") | OAuth app is in *Testing* mode; tokens expire in 7 days | Publish the OAuth app (set to In production), then reconnect the credential |
| "Column to match on not found" / "Column names were updated after the node's setup" | Headers differ from the node's saved list | Check headers are spelled exactly; open the node and click the refresh icon in column mapping |
| "Could not get parameter / Sheet not found" | Tab name mismatch | Tab must be named exactly like the node setting (default `Form Responses 1`) |
| Updates go to the wrong row, or "Could not locate the sheet row" alert | Two rows with identical Timestamp, Email and Message; or rows deleted/sorted mid-run | Avoid sorting or deleting rows in the Sheet while workflow is live; retry with a new submission |
| Telegram `400 chat not found` | Wrong chat ID, or you never pressed Start on the bot | Re-check the ID; open the bot and press Start |
| Telegram `403 bot was blocked` | The user blocked the bot | Unblock the bot in Telegram |
| Telegram `400 can't parse entities` | A parse mode (Markdown/HTML) was set and the message has special characters | Leave Parse Mode empty (the default in this workflow) |
| Telegram `message is too long` | Over 4096 characters | Already handled: message is cut at 500 characters |
| Gmail `Invalid To header` | Bad email passed validation | Rare; the failure goes to the alert path. Fix the email in the Sheet and email the customer manually |
| Gmail `Daily user sending quota exceeded` | Free Gmail sends about 500 emails/day | Wait 24 hours, or use Google Workspace |
| "Referenced node is unexecuted" when testing one node | A node references earlier nodes that haven't run | Run the whole workflow with **Execute workflow** instead of a single node |
| "Can't get data for expression" / paired item error | Item link between nodes lost (often after editing nodes) | Re-run from the trigger. Do not insert merge/aggregate nodes in the middle without relinking |
| IDs show yesterday's date | Workflow timezone not Nairobi | Workflow ⋯ → Settings → Timezone → *Africa/Nairobi* (already set in the import) |
| Phone number loses the leading 0 or "+" | Column formatted as number | Format column D as Plain text (and set it before collecting data) |
| Duplicate IDs | Two executions ran at the same moment | Rare at small volume; in n8n settings, limit concurrent executions to 1 if self-hosted |
| Import fails or nodes show "?" | n8n version too old for a node version | Update n8n (this workflow targets a recent 1.x release) |

---

