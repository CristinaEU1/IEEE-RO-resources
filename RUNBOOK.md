# IEEE Romania Resources: Runbook

How the PDF download page is built, how to set it up from scratch, how to maintain it, and how to fix common problems.

- **Live site:** https://cristinaeu1.github.io/IEEE-RO-resources/
- **Repository:** https://github.com/CristinaEU1/IEEE-RO-resources
- **Cost:** free GitHub account, no third-party services

---

## 1. What the page does

| Requirement | How it is done |
|---|---|
| 1. Offer PDFs for download | PDFs are attached to a GitHub **Release** (tag `v1`). The page lists them with a **Download** button. |
| 2. Count unique visitors | The first time a browser opens the page, it silently downloads a tiny file, `pageviews.dat`, from the same release. That file's download count is the visitor count. A marker saved in the browser stops repeat visits from counting. |
| 3. Count downloads per PDF | GitHub counts every download of a release file on its own. |
| Showing the numbers | A **GitHub Actions** job reads all counts about every 15 minutes and publishes them as `stats.json` next to the page. Visitors read that file, so any number of visitors can use the page at once. |

### How it fits together

```
Every ~15 min, GitHub Actions:
   reads the release files and their download counts (authenticated)
   → writes stats.json → publishes index.html + stats.json to GitHub Pages

Visitor opens the page
   ├─► First visit in this browser?  yes → download pageviews.dat in the background (+1 visitor)
   │                                  no  → do nothing
   └─► Read stats.json from the site
          → "Unique visitors" = downloads of pageviews.dat
          → list every .pdf with its download count and a Download button
          → (if stats.json is missing: ask GitHub's public API directly, the backup route)
```

### What is in the repository

| File | Purpose |
|---|---|
| `index.html` | The whole website: layout, visitor counter, and PDF list |
| `.github/workflows/publish.yml` | The GitHub Actions job that refreshes `stats.json` and publishes the site |
| `README.md` | Short description of the repository |
| `RUNBOOK.md` | This manual |
| `LICENSE` | MIT licence |

**Important:** the PDFs are **not** stored as normal files in the repository. GitHub only counts downloads for files attached to a **Release**, so the PDFs must live there. `stats.json` is not in the repository either. The Actions job creates it fresh on every run.

---

## 2. Initial setup (one time)

### Step 1: Turn on GitHub Pages

1. Open the repository's settings directly:
   **https://github.com/CristinaEU1/IEEE-RO-resources/settings/pages**
2. Under **Build and deployment**, set **Source** to **GitHub Actions**.
   - During the first setup this was `Deploy from a branch` / `main` / `/ (root)`. Step 5 explains the switch.
3. The site address is **https://cristinaeu1.github.io/IEEE-RO-resources/**

> **Workaround:** if you don't see **Build and deployment**, you are in your **account** settings, which only show "Verified & approved domains". Use the direct link in point 1, or go to the repository first and then click its own **Settings** tab.
>
> Ignore the **Visibility / GitHub Enterprise** section. It is for paid accounts and not needed.

### Step 2: Create release `v1` with the counter file and the PDFs

1. Prepare the files:
   - **`pageviews.dat`:** any tiny file with exactly this name. Its content doesn't matter.
   - **Your PDFs**, renamed to how they should look on the page. The page turns `-` and `_` into spaces and drops `.pdf`. For example, `IEEE_Membership_Guide_2026.pdf` appears as **IEEE Membership Guide 2026**.
2. Open **https://github.com/CristinaEU1/IEEE-RO-resources/releases/new**
3. Click **Choose a tag**, type `v1`, then click **+ Create new tag: v1 on publish**.
4. **Release title:** `IEEE Romania Resources`. The description can stay empty.
5. Drag `pageviews.dat` and the PDFs into **Attach binaries by dropping them here or selecting them**. Wait until each file shows its size.
6. Leave **Set as the latest release** ticked and **Set as a pre-release** unticked.
7. Click **Publish release**. Don't use **Save draft**, because drafts are invisible to visitors.

> **Workarounds:**
> - The tag must be exactly `v1` (lowercase) and the file exactly `pageviews.dat`. These names are set in `index.html` (see section 5).
> - On a Mac, check that the counter file didn't get a hidden extension such as `pageviews.dat.txt`. In Finder, choose **Get Info** to see the full name.

### Step 3: Count unique visitors instead of page views

This is already in `index.html`. The visitor block only registers a visit when the browser has no "already counted" marker:

```js
const SEEN_KEY = "ieee-ro-resources-visited";
let firstVisit = true;
try {
  firstVisit = !localStorage.getItem(SEEN_KEY);
  if (firstVisit) localStorage.setItem(SEEN_KEY, new Date().toISOString());
} catch (e) { /* storage blocked: count the visit anyway */ }
if (firstVisit) {
  new Image().src =
    `https://github.com/${GH_OWNER}/${GH_REPO}/releases/download/${GH_TAG}/${VIEW_ASSET}?t=${Date.now()}`;
}
```

### Step 4: Reset the counters before going live

Testing inflates the numbers. Deleting a file from the release and uploading it again starts its count at **0**.

1. Open **https://github.com/CristinaEU1/IEEE-RO-resources/releases/tag/v1**
2. Click the **pencil icon** (Edit release).
3. Click the **trash icon** next to `pageviews.dat` and next to any test PDFs.
4. Drag in a fresh `pageviews.dat` and the final PDFs.
5. Click **Update release**.
6. Check the site after about 15–30 minutes. Visitors and downloads should be at **0**.

### Step 5: Publish the numbers with GitHub Actions (avoids the visitor limit)

**Why:** without this, every visitor's browser asks GitHub's public API for the numbers. That API allows only **60 requests per hour per internet connection**. On shared Wi-Fi, such as a university or conference room, the limit runs out and the page shows *"Could not load the list."*

**How:** a GitHub Actions job fetches the numbers (authenticated, with a much higher limit) and publishes them as `stats.json` next to the page.

1. **Switch Pages to Actions:** open https://github.com/CristinaEU1/IEEE-RO-resources/settings/pages and set **Source** to **GitHub Actions**. There is nothing to save; it applies immediately. Ignore GitHub's suggested workflow templates.
2. **Add the job file:**
   1. On the repository's **Code** tab, click **Add file**, then **Create new file**.
   2. In the name box, type exactly: `.github/workflows/publish.yml`. Typing each `/` creates a folder.
   3. Paste the contents of `publish.yml` into the editor.
   4. Click **Commit changes…**, then **Commit changes**.
3. **Upload the updated `index.html`:** click **Add file**, then **Upload files**, drag in `index.html`, and click **Commit changes**.
4. **Check the job ran:** open the **Actions** tab. The run **"Publish site and refresh stats"** should get a green tick within 1–2 minutes.
5. **Check the site:** hard-reload with **Cmd+Shift+R** (Mac) or **Ctrl+Shift+R** (Windows). At the bottom, under the table, you should see *"Numbers last updated: …"*. That line confirms the page is reading `stats.json`.

**The job runs:**
- about every **15 minutes**. GitHub sometimes starts scheduled jobs later than planned when it is busy.
- whenever anything is committed to the repository.
- whenever a release is published or edited.
- on demand: go to **Actions**, choose **Publish site and refresh stats**, then click **Run workflow**.

> **Workaround: the 60-day pause.** GitHub pauses scheduled jobs in a repository after **60 days without any activity**. When this happens, GitHub shows a banner in the **Actions** tab. The page keeps working (it falls back to asking GitHub directly), but the shared-Wi-Fi protection is gone. **Fix:** commit any small change, such as editing the README or the change log below, or open **Actions → Publish site and refresh stats** and click **Enable workflow**.

---

## 3. Routine tasks

### Add a new PDF
1. Rename the PDF to its display name (see Step 2).
2. Open the `v1` release, click **Edit**, drag the PDF in, and click **Update release**.
3. It appears on the site at the next job run, within about 15 minutes. To show it straight away, go to **Actions → Publish site and refresh stats** and click **Run workflow**.

### Remove a PDF
Edit the `v1` release and click the trash icon next to the file. **Its download count is lost.** Note the number first if you need it for reporting.

### Replace a PDF with a new version
- **Same file name:** delete the old file and upload the new one. The count restarts at 0, so note the old count first.
- **Keep both versions:** give the new file a different name, for example `..._v2.pdf`. Both will be listed.

### Read the numbers for a report
- **On the site:** the numbers are as of the *"Numbers last updated"* time shown under the table.
- **Raw data:** open https://cristinaeu1.github.io/IEEE-RO-resources/stats.json. It shows one `download_count` per file. Opening it does **not** use up the visitor limit.
- **Live from GitHub:** https://api.github.com/repos/CristinaEU1/IEEE-RO-resources/releases. This one **does** count toward your connection's 60-per-hour limit, so use it sparingly.

### Change the page title, text or style
Edit `index.html` (**Add file → Upload files**, or the pencil icon on the file). The title is in the `<title>` tag and the `<h1>` heading. Colours and spacing are in the `<style>` block. The change is live 1–2 minutes after the commit, once the Actions run is green.

---

## 4. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Visitors show **"–"**, table shows **"No documents yet."** | No published release `v1`, or `pageviews.dat` is missing or misnamed | Do Step 2. Check that the tag is `v1`, the file is named `pageviews.dat`, and the release is published, not a draft. |
| Numbers don't change after downloads or visits | Two delays add up: GitHub updates counts about 10–15 minutes late, and the job publishes them about every 15 minutes | Wait up to about 30 minutes, then hard-reload. To speed things up, click **Run workflow** in the Actions tab. |
| **"Could not load the list. Please try again later."** | `stats.json` is unavailable **and** the backup route hit GitHub's limit of 60 requests per hour per connection, which is common after many refreshes or on shared Wi-Fi | Check the **Actions** tab. If the last run failed or the job is paused, see the next rows. Otherwise wait up to an hour. |
| No *"Numbers last updated"* line under the table | The page couldn't read `stats.json` and used the backup route | Check that Pages **Source** is **GitHub Actions** (Step 5, point 1) and that the latest Actions run is green. |
| Actions run has a **red cross** | Usually Pages **Source** is not set to **GitHub Actions**, or the job file has a typo | Open the failed run to read the error. Fix the Pages source, or re-paste `publish.yml` exactly, then click **Re-run jobs**. |
| Actions tab shows a banner that the scheduled workflow was **disabled** | The 60-day inactivity pause | Click **Enable workflow**, or commit any small change. See Step 5. |
| Visitor count doesn't go up when **I** open the page | Intended: your browser has already been counted | Test from another device, another browser, or a private window. |
| My own visits should count again | The "already counted" marker is stored in your browser | Clear site data for `cristinaeu1.github.io` in the browser settings. |
| Page changes don't appear after uploading `index.html` | Publishing takes 1–2 minutes, and the browser may show an old copy | Wait for the green tick in **Actions**, then hard-reload. |
| Visitor count seems lower than expected | Some ad-blockers or privacy extensions block the hidden counter request | Nothing to fix on the page. Treat the count as a minimum. |
| Download counts seem higher than expected | Link-preview bots (Slack, WhatsApp, Teams) and crawlers also download files | Expected. Treat the count as approximate. |
| A PDF is missing from the list | The file doesn't end in `.pdf`, or the job hasn't run since you added it | Rename it to end in `.pdf`, or click **Run workflow**. |
| No **Build and deployment** section in Settings → Pages | You are in account settings, not repository settings | Use https://github.com/CristinaEU1/IEEE-RO-resources/settings/pages |

---

## 5. Settings

### In `index.html` (top of the `<script>` block)

```js
const GH_OWNER   = "CristinaEU1";        // GitHub account
const GH_REPO    = "IEEE-RO-resources";  // repository name
const GH_TAG     = "v1";                 // release that holds pageviews.dat
const VIEW_ASSET = "pageviews.dat";      // the visitor-counter file
```

- **PDFs:** the page lists PDFs from **all** published releases, so a second release (for example `v2`) also works. `pageviews.dat` must stay in the release named in `GH_TAG`.
- **Moving or renaming the repository:** update `GH_OWNER` and `GH_REPO`. The Actions job adapts automatically.
- **Changing the visitor marker:** changing `SEEN_KEY` makes every browser count once more, which is effectively a reset of who counts as "seen".

### In `.github/workflows/publish.yml`

- **Refresh frequency:** the line `- cron: "*/15 * * * *"`. For example, `"*/30 * * * *"` means every 30 minutes. GitHub doesn't run scheduled jobs more often than every 5 minutes.
- **Extra site files**, such as a logo: add a line under **Build the site folder**, for example `cp logo.png _site/`. Only files copied there are published.

---

## 6. Known limits

- **"Unique" means unique per browser, not per person.** The same person on a phone and a laptop counts twice. Private-browsing sessions and cleared browser data also count again. Exact unique-person counting needs a server, which this free GitHub-only setup doesn't have.
- **Numbers on the page are typically 15–30 minutes behind.** That is GitHub's own counting delay plus the refresh schedule.
- **Counts can't be edited.** They can only be reset, by deleting and re-uploading the file.
- **Each file can be at most 2 GB.** That limit is far above any PDF.
- **There is no history over time.** GitHub keeps totals only. For monthly figures, read `stats.json` on a fixed date each month and record the numbers.
- **Scheduled jobs pause after 60 days without repository activity.** See Step 5.

---

## 7. Change log

| Date | Change |
|---|---|
| 2026-10-03 | Repository created; `index.html` with page-view and download counters |
| 2026-10-31 | GitHub Pages enabled (`main` / root); release `v1` created with `pageviews.dat` and test PDFs; visitor counter changed to count each browser once |
| 2026-10-31 | GitHub Actions job added: numbers published as `stats.json` about every 15 minutes, so visitors no longer hit GitHub's 60-per-hour limit; Pages source switched to GitHub Actions |
| *later* | Step 4: reset counters and add final PDFs |
