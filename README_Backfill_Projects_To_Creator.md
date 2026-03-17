# 📦 Zoho Projects → Zoho Creator Bulk Backfill (Deluge)

**Function:** `Backfill_Projects_To_Creator`
**Type:** Manual / One-Time Bulk Sync | Paginated Batch Runner

A Zoho Deluge function that **bulk-syncs all existing Zoho Projects records into Zoho Creator** in paginated batches of 300 per run (3 pages × 100). Designed to be run manually and repeatedly with an incrementing start index until all historical records are migrated. Safe to re-run — it upserts records (update if exists, create if not), so no duplicates are created.

> **When to use this:** You already have hundreds or thousands of projects in Zoho Projects and need to populate a Zoho Creator database for the first time, or re-sync all records after a schema change. For ongoing real-time sync, use [`Update_Project_task_RR`](../zoho-projects-creator-task-sync/) instead.

---

## 📌 Table of Contents

- [Use Case](#-use-case)
- [How It Works](#-how-it-works)
- [Prerequisites](#-prerequisites)
- [Setup Instructions](#-setup-instructions)
- [Configuration](#-configuration)
- [How to Run — Step by Step](#-how-to-run--step-by-step)
- [Run Schedule Reference](#-run-schedule-reference)
- [Field Mapping Reference](#-field-mapping-reference)
- [Client Lookup Logic](#-client-lookup-logic)
- [Summary Log Output](#-summary-log-output)
- [Extending the Script](#-extending-the-script)
- [Notes & Limitations](#-notes--limitations)

---

## 🎯 Use Case

This script solves the **historical data gap** problem: you set up a real-time sync automation going forward, but you still have months or years of existing project records in Zoho Projects that need to be backfilled into Zoho Creator.

**Typical scenarios:**

| Scenario | What this script does |
|---|---|
| First-time Creator portal setup | Migrates all existing Zoho Projects records into Creator |
| Schema change in Creator | Re-syncs all records to populate new fields |
| Data cleanup run | Refreshes Engine, Location, Invoice data across all records |
| Testing the sync logic | Validates field mapping before going live with real-time sync |

**Why paginated batching?** The Zoho Projects API caps results at 100 per request, and Deluge functions have execution time limits. This script fetches 3 pages (300 records) per run, logs the next start index, and lets you re-run with minimal setup.

---

## ⚙️ How It Works

Each run processes exactly **300 records across 3 pages**:

```
Run starts at start_index (e.g. 1)
        │
        ├─ Page 1: index=1,   range=100  → fetch projects  1–100
        ├─ Page 2: index=101, range=100  → fetch projects 101–200
        └─ Page 3: index=201, range=100  → fetch projects 201–300

For each project on each page:
  1. Extract project fields + custom fields
  2. Strip HTML from rich-text fields (Test Type)
  3. Convert dates: MM-dd-yyyy → dd-MMM-yyyy
  4. Build Creator update map
  5. Client lookup: Exact → Contains → Starts With → Auto-Create
  6. Upsert: UPDATE if Project_Id exists in Creator, CREATE if not

At end of run:
  → Logs: Total fetched | Updated | Created | Skipped | Errors
  → Logs: NEXT RUN → start_index = X
```

---

## ✅ Prerequisites

1. **Zoho Projects** account with projects and custom fields configured
2. **Zoho Creator** app with:
   - Projects form (report link: `Projects_for_Admins`)
   - Contacts form (report link: `Contacts_for_Admins`)
   - Add Contact form (link: `Add_Contact`)
   - Schedule Project form (link: `Schedule_A_Project_Release`)
3. **Zoho Connection** named `project_connection` with scopes:
   - `ZohoProjects.portals.READ`
   - `ZohoProjects.projects.READ`
4. **Zoho Creator OAuth connection** (built-in) named `zoho_oauth_connection`
5. Your **Zoho Projects Portal ID** (numeric)
6. Your **Zoho Creator App Owner username** and **App Link Name**

---

## 🚀 Setup Instructions

### Step 1 — Create the Zoho Connection (`project_connection`)

1. Go to **Zoho Developer Console** → **Connections** → **Create Connection**
2. Select **Zoho OAuth**, name it: `project_connection`
3. Add scopes: `ZohoProjects.portals.READ`, `ZohoProjects.projects.READ`
4. Authorize with your Zoho account

### Step 2 — Find Your Portal ID

```
GET https://projectsapi.zoho.com/restapi/portals/
```
The response `"id"` field is your numeric `portalId`.

### Step 3 — Find Your Creator App Details

- **App Owner:** your Zoho account username (e.g. `john.doe`)
- **App Link Name:** visible in the Creator URL — `creator.zoho.com/OWNER/APP_LINK/`

### Step 4 — Deploy the Function

1. In Zoho Creator → **Functions** → create a new standalone function
2. Name it: `Backfill_Projects_To_Creator`
3. No arguments needed (all config is inside the function)
4. Paste the script from `Backfill_Projects_To_Creator.dg`
5. Update all placeholder values (see [Configuration](#-configuration))
6. Save

---

## 🔧 Configuration

Update these values at the top of the script before the first run:

```javascript
start_index  = 1;                        // ← Start at 1 for first run
portalId     = "YOUR_PORTAL_ID";         // ← Numeric Zoho Projects Portal ID
creatorOwner = "YOUR_CREATOR_APP_OWNER"; // ← Zoho username of Creator app owner
creatorApp   = "YOUR_APP_LINK_NAME";     // ← Link name of your Creator app
creatorConn  = "zoho_oauth_connection";  // ← Update if your connection name differs
```

Also verify your Creator **form and report link names** match these used in the script:

| Purpose | Link Name in Script |
|---|---|
| Projects report | `Projects_for_Admins` |
| Contacts report | `Contacts_for_Admins` |
| Add contact form | `Add_Contact` |
| Create project form | `Schedule_A_Project_Release` |

---

## 🏃 How to Run — Step by Step

### First Run
1. Set `start_index = 1` in the script
2. Save and execute the function manually
3. Watch the Deluge log — at the end you will see:
   ```
   ▶️ NEXT RUN → start_index = 301
   ```

### Subsequent Runs
4. Update `start_index` to the value shown in the log (e.g. `301`)
5. Save and execute again
6. Repeat, incrementing by 300 each time

### When to Stop
7. When the log shows:
   ```
   ✅ No projects at index X — DONE!
   ```
   for all 3 pages — your backfill is complete.

### ⏱️ Wait Between Runs
Allow **5–10 minutes** between runs to avoid hitting Zoho API rate limits.

---

## 📅 Run Schedule Reference

| Run # | `start_index` | Records Processed |
|---|---|---|
| 1 | `1` | 1 – 300 |
| 2 | `301` | 301 – 600 |
| 3 | `601` | 601 – 900 |
| 4 | `901` | 901 – 1,200 |
| 5 | `1201` | 1,201 – 1,500 |
| … | … | … |
| N | `(N-1)×300 + 1` | Continues until "DONE!" |

> **Tip:** Keep a simple note of which runs you've completed. The log always tells you the exact next `start_index` to use.

---

## 🗂️ Field Mapping Reference

### Zoho Projects → Zoho Creator

| Creator Field Link | Zoho Projects Source | Notes |
|---|---|---|
| `Project_Name` | `name` | Project title |
| `Project_Id` | `id_string` | Used as unique key for upsert lookup |
| `Status` | `custom_status_name` | Custom status label |
| `Start_Date` | `start_date` | Converted to dd-MMM-yyyy |
| `End_Date` | `end_date` | Converted to dd-MMM-yyyy |
| `Owner` | `owner_name` | Project owner display name |
| `Owner_Email` | `owner_email` | Project owner email |
| `Test_Type` | Custom field: `Test Type` | HTML stripped before sync |
| `Invoice_Status1` | Custom field: `Invoice Status` | |
| `Total_Invoice_amount` | Custom field: `Sale Order Total` | |
| `Invoiced_date` | Custom field: `Invoiced Date` | Converted to dd-MMM-yyyy |
| `Engine` | Custom field: `Engine Name` | |
| `Location` | Custom field: `Location Name` | |
| `Client` | Creator Contacts lookup | Matched by `Customer` custom field value |

### Date Format Conversion

```javascript
// All dates converted from API format to display format:
proj_start = proj_start_raw.toDate("MM-dd-yyyy").toString("dd-MMM-yyyy");
// "03-17-2026" → "17-Mar-2026"
```

### HTML Stripping

```javascript
// Test Type field may contain HTML wrapper tags from Zoho:
test_type = test_type_raw
    .replaceAll("<div>","").replaceAll("</div>","")
    .replaceAll("<br>","").replaceAll("</br>","")
    .replaceAll("<br/>","").trim();
```

---

## 🔍 Client Lookup Logic

For each project, the script tries to find the matching client in the Creator Contacts report using 3 progressive strategies before auto-creating:

```
Customer value from project custom field
         │
         ▼
Strategy 1: Exact match
            Name == "Acme Corp"
         │
         ▼ not found
Strategy 2: Contains match
            Name contains "Acme Corp"
         │
         ▼ not found
Strategy 3: Starts-with match (first 10 characters)
            Name starts with "Acme Co"
         │
         ▼ still not found
Auto-Create: New Contact created with customer name
             Project record linked to new contact ID
```

---

## 📋 Summary Log Output

At the end of each run, the function prints a full summary:

```
=== BACKFILL STARTED === From index: 1
Pages: 1 | 101 | 201
── Fetching page 1 | Index: 1
   Fetched: 100 projects
✅ UPDATED [123456789] Engine: CAT-3516B | Location: North Plant
✅ CREATED [987654321] New Site Test Job
⚠️ Client not linked: Unknown Corp
── Fetching page 2 | Index: 101
   Fetched: 100 projects
...
── Fetching page 3 | Index: 201
   Fetched: 100 projects
...
=== BACKFILL BATCH COMPLETE ===
Started at index  : 1
Total fetched     : 300
✅ Success        : 296 (Updates: 210 | Creates: 86)
⏭️ Skipped        : 2
❌ Errors         : 2
▶️ NEXT RUN → start_index = 301
```

---

## 🔨 Extending the Script

| Extension | How to Implement |
|---|---|
| **Add more pages per run** | Add `page4_index`, `page5_index` blocks — but stay within Deluge execution limits |
| **Filter by project status** | Add a criteria check inside the `for each` loop: `if(proj_status != "Completed") { ... }` |
| **Sync task data too** | Add a nested API call for each project to fetch its first task |
| **Track sync progress** | Write `start_index` progress to a Creator settings record after each page |
| **Add email report** | Add a `sendmail` at the end with the summary stats |
| **Notify on errors** | Add a `sendmail` inside the error blocks to alert on specific failures |

---

## ⚠️ Notes & Limitations

- **300 records per run only:** Deluge functions have execution time limits. 3 pages × 100 is the recommended safe batch size. Adjust if you experience timeouts.
- **Wait between runs:** Zoho Projects API has rate limits. Allow 5–10 minutes between runs to avoid `429 Too Many Requests` errors.
- **Safe to re-run:** The upsert logic uses `Project_Id` as the unique key, so re-running a batch won't create duplicates — it will update existing records.
- **Custom field names are case-sensitive:** Strings in `.get("Field Name")` must exactly match your Zoho Projects custom field labels.
- **HTML stripping covers common tags only:** Extend the `replaceAll` chain if your rich-text fields contain other HTML tags.
- **Auto-created contacts are minimal:** Only `Name` is set. Extend `new_client_map` to include additional fields if your Contacts form requires them.
- **Deluge `while` loops are not supported:** This script uses `for each` as required by Deluge syntax.
- **No task data:** This function syncs project-level data only. For task field sync, combine with `Update_Project_task_RR`.

---

## 🔗 Related Scripts in This Series

| Script | Purpose |
|---|---|
| [`Get_Project_And_Task_Details`](../zoho-projects-get-project-task-details/) | Fetch and log all fields for a single project + task |
| [`Update_Project_task_RR`](../zoho-projects-creator-task-sync/) | Real-time sync triggered on task update |
| **`Backfill_Projects_To_Creator`** ← *this script* | One-time bulk migration of all historical records |

---

## 🧑‍💻 Author

**Rafiullah Nikzad**
Senior Zoho Developer | Certified Zoho Developer
[LinkedIn](https://www.linkedin.com/in/rafiullahnikzad/) | [Portfolio](https://rafiullahnikzad.netlify.app)

> Part of the **Zoho Afghanistan** community — sharing open-source Zoho Deluge scripts to help Afghan businesses automate and grow.

---

## 📄 License

MIT License — free to use, modify, and distribute with attribution.
