# IEEE Romania Resources: Runbook

How the PDF download page is built, how to set it up from scratch, how to maintain it, and how to fix common problems.

- **Live site:** https://cristinaeu1.github.io/IEEE-RO-resources/
- **Repository:** https://github.com/CristinaEU1/IEEE-RO-resources
- **Cost:** free GitHub account, no third-party services

---

## 1. What the page does

| Requirement | How it is done |
|---|---|
| 1. Offer PDFs for download | PDFs are attached to GitHub **Releases**. The page lists them with a **Download** button. |
| 2. Count unique visitors | The first time a browser opens the page, it silently downloads a tiny file, `pageviews.dat`, from release `v1`. That file's download count is the visitor count. A marker saved in the browser stops repeat visits from counting. |
| 3. Count downloads per PDF | GitHub counts every download of a release file on its own. |
| Showing the numbers | A **GitHub Actions** job reads all counts about every 15 minutes and publishes them as `stats.json` next to the page. Visitors read that file, so any number of visitors can use the page at once. |
| Folders | **Each release is one folder** on the page, named after the release title. |
| Logos | Three logo spaces at the top of the page. Empty ones show a dashed placeholder. |

### How it fits together

```
Every ~15 min (and on every change), GitHub Actions:
   reads all releases, their files and download counts (authenticated)
   → writes stats.json (each file tagged with its folder = release title)
   → publishes index.html + logos/ + stats.json to GitHub Pages

Visitor opens the page
   ├─► First visit in this browser?  yes → download pageviews.dat in the background (+1 visitor)
   │                                  no  → do nothing
   └─► Read stats.json from the site
          → "Unique visitors" = downloads of pageviews.dat
          → one folder box per release, listing its PDFs with download counts and Download buttons
          → (if stats.json is missing: ask GitHub's public API directly, the backup route)
```

### What is in the repository

| File | Purpose |
|---|---|
| `index.html` | The whole website: logos, visitor counter, folders, and PDF lists |
| `.github/workflows/publish.yml` | The GitHub Actions job that refreshes `stats.json` and publishes the site |
| `logos/` | *(created when the first logo is added)* Logo image files |
| `README.md` | Short description of the repository |
| `RUNBOOK.md` | This manual |
| `LICENSE` | MIT licence |

**Important:** the PDFs are **not** stored as normal files in the repository. GitHub only counts downloads for files attached to a **Release**, so the PDFs must live there. `stats.json` is not in the repository either. The Actions job creates it fresh on every run.

---

## 2. Initial setup (one time)

> **Status (31 Oct 2026):** Steps 1, 2, 3, 5 and 6 are done. **Step 4 is still open.** It will be done after the MVP is validated with the customer.

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
4. **Release title:** `IEEE Romania Resources`. The title becomes the folder name on the page (see Step 6), so it can also be something like `General`. The description can stay empty.
5. Drag `pageviews.dat` and the PDFs into **Attach binaries by dropping them here or selecting them**. Wait until each file shows its size.
6. Leave **Set as the latest release** ticked and **Set as a pre-release** unticked.
7. Click **Publish release**. Don't use **Save draft**, because drafts are invisible to visitors.

> **Workarounds:**
> - The tag must be exactly `v1` (lowercase) and the file exactly `pageviews.dat`. These names are set in `index.html` (see section 5).
> - On a Mac, check that the counter file didn't get a hidden extension such as `pageviews.dat.txt`. In Finder, choose **Get Info** to see the full name.
> - If `v1` contains only `pageviews.dat` and no PDFs, it doesn't appear as a folder on the page. That's fine.

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

### Step 4: Reset the counters before going live (still open)

Testing inflates the numbers. Deleting a file from a release and uploading it again starts its count at **0**.

1. Open **https://github.com/CristinaEU1/IEEE-RO-resources/releases/tag/v1**
2. Click the **pencil icon** (Edit release).
3. Click the **trash icon** next to `pageviews.dat` and next to the test PDFs (`test1.pdf`, `Test2.pdf`).
4. Drag in a fresh `pageviews.dat`. Then either drag in the final PDFs here, or put them into folder releases (see Step 6).
5. Click **Update release**.
6. To publish straight away, go to **Actions → Publish site and refresh stats** and click **Run workflow**.
7. Check the site after about 15–30 minutes. Visitors and downloads should be at **0**.

> **Note:** your own browser is already marked as counted, so your visits won't raise the new counter. That's intended. Test from another device.

### Step 5: Publish the numbers with GitHub Actions (avoids the visitor limit)

**Why:** without this, every visitor's browser asks GitHub's public API for the numbers. That API allows only **60 requests per hour per internet connection**. On shared Wi-Fi, such as a university or conference room, the limit runs out and the page shows *"Could not load the list."* This happened during testing:

```
API rate limit exceeded for <your IP address>. (But here's the good news: Authenticated requests get a higher rate limit. …)
```

**How:** a GitHub Actions job fetches the numbers (authenticated, with a much higher limit) and publishes them as `stats.json` next to the page.

1. **Switch Pages to Actions:** open https://github.com/CristinaEU1/IEEE-RO-resources/settings/pages and set **Source** to **GitHub Actions**. There is nothing to save; it applies immediately. Ignore GitHub's suggested workflow templates.
2. **Add the job file:**
   1. On the repository's **Code** tab, click **Add file**, then **Create new file**.
   2. In the name box, type exactly: `.github/workflows/publish.yml`. Typing each `/` creates a folder.
   3. Paste the contents of `publish.yml` into the editor.
   4. Click **Commit changes…**, then **Commit changes**.
3. **Upload the updated `index.html`:** click **Add file**, then **Upload files**, drag in `index.html`, and click **Commit changes**.
4. **Check the job ran:** open the **Actions** tab. The run **"Publish site and refresh stats"** should get a green tick within 1–2 minutes.
5. **Check the site:** hard-reload with **Cmd+Shift+R** (Mac) or **Ctrl+Shift+R** (Windows). At the bottom of the page you should see *"Numbers last updated: …"*. That line confirms the page is reading `stats.json`.

**The job runs:**
- about every **15 minutes**. GitHub sometimes starts scheduled jobs later than planned when it is busy.
- whenever anything is committed to the repository.
- whenever a release is published or edited.
- on demand: go to **Actions**, choose **Publish site and refresh stats**, then click **Run workflow**.

> **Workaround: the 60-day pause.** GitHub pauses scheduled jobs in a repository after **60 days without any activity**. When this happens, GitHub shows a banner in the **Actions** tab. The page keeps working (it falls back to asking GitHub directly), but the shared-Wi-Fi protection is gone. **Fix:** commit any small change, such as editing the README or the change log below, or open **Actions → Publish site and refresh stats** and click **Enable workflow**.

### Step 6: Logos and folders

#### Logos

The page shows **three logo spaces** at the top, left to right. Empty ones show a dashed placeholder labelled *Logo 1*, *Logo 2*, *Logo 3*.

To add a logo:
1. Prepare the image: **PNG with a transparent background, or SVG**, roughly 3:1 width to height. Logos are scaled to fit a box up to 80 px tall (56 px on phones).
2. Upload it into a `logos` folder: on the repository's **Code** tab, click **Add file**, then **Upload files**. Drag in the image, and before committing, put `logos/` in front of its name, for example `logos/ieee-romania.png`. Alternatively, drag a whole local folder called `logos` that contains the image.
3. Edit `index.html` (open the file, then click the pencil icon). Fill in the logo's line in the `LOGOS` setting:
   ```js
   const LOGOS = [
     { src: "logos/ieee-romania.png", alt: "IEEE Romania", link: "https://ieee.ro" },
     { src: "", alt: "Logo 2", link: "" },
     { src: "", alt: "Logo 3", link: "" },
   ];
   ```
   - `src`: the image path. Leave it empty for a placeholder.
   - `alt`: a short description, used by screen readers and shown if the image fails to load.
   - `link`: optional web address the logo opens when clicked. Leave it empty for no link.
4. Commit, wait for the green tick in **Actions**, then hard-reload the site.

> **Workaround:** the image path is case-sensitive, so `Logo.PNG` and `logo.png` are different. If a logo shows only its `alt` text, check that the name in `src` matches the uploaded file exactly.

#### Folders

**Each GitHub Release is one folder** on the page:
- **Folder name** = the release **title**. If the title is empty, the tag is used.
- Folders are sorted by name with numbers in order (D1, D2 … D10), and the documents inside each folder are sorted by name.
- A folder with no PDFs is hidden. That's why `v1`, if it holds only `pageviews.dat`, doesn't show.
- Visitors click a folder name to collapse or expand it. Folders open by default (see `FOLDERS_OPEN` in section 5).

**To create a folder:**
1. Open **https://github.com/CristinaEU1/IEEE-RO-resources/releases/new**
2. **Choose a tag:** a short, lowercase name without spaces, for example `d1`, then click **Create new tag … on publish**. The tag only has to be unique; visitors don't see it.
3. **Release title:** the folder name visitors will see, for example `D1`.
4. Drag in the folder's PDFs and click **Publish release**.
5. The folder appears on the site at the next job run, or click **Actions → Publish site and refresh stats → Run workflow**.

> **Workarounds:**
> - Don't put `pageviews.dat` into a folder release. It must stay in `v1`.
> - To make a folder appear first, start its title with a number, for example `1. Guides`, `2. Templates`.
> - Don't tick **Set as the latest release** for folder releases; leave `v1` as the latest. This only affects GitHub's own Releases page, not your site.

---

## 3. Routine tasks

### Add a new PDF
1. Rename the PDF to its display name (see Step 2).
2. Open the folder's release (**Releases**, then click the folder name), click **Edit** (pencil icon), drag the PDF in, and click **Update release**.
3. It appears on the site at the next job run, within about 15 minutes. To show it straight away, go to **Actions → Publish site and refresh stats** and click **Run workflow**.

### Move a PDF to another folder
GitHub can't move a file between releases, so:
1. Note its download count if you need it, because **the count restarts at 0**.
2. Upload the PDF to the new folder's release.
3. Delete it from the old folder's release (Edit, then the trash icon).

### Rename a folder
Edit the release and change its **title**. The PDFs and their counts stay as they are.

### Delete a folder
Delete its PDFs, or delete the whole release (the release page's **Delete** button). The download counts of its PDFs are lost. **Never delete release `v1`**, because it holds the visitor counter.

### Remove a PDF
Edit its release and click the trash icon next to the file. **Its download count is lost.** Note the number first if you need it for reporting.

### Replace a PDF with a new version
- **Same file name:** delete the old file and upload the new one. The count restarts at 0, so note the old count first.
- **Keep both versions:** give the new file a different name, for example `..._v2.pdf`. Both will be listed.

### Add or change a logo
See Step 6 → **Logos**.

### Read the numbers for a report
- **On the site:** the numbers are as of the *"Numbers last updated"* time shown at the bottom.
- **Raw data:** open https://cristinaeu1.github.io/IEEE-RO-resources/stats.json. It shows each file with its `download_count` and `folder`. Opening it does **not** use up the visitor limit.
- **Live from GitHub:** https://api.github.com/repos/CristinaEU1/IEEE-RO-resources/releases. This one **does** count toward your connection's 60-per-hour limit, so use it sparingly.

### Change the page title, text or style
Edit `index.html` (**Add file → Upload files**, or the pencil icon on the file). The title is in the `<title>` tag and the `<h1>` heading. Colours and spacing are in the `<style>` block. The change is live 1–2 minutes after the commit, once the Actions run is green.

---

## 4. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Visitors show **"–"** | No published release `v1`, or `pageviews.dat` is missing or misnamed | Do Step 2. Check that the tag is `v1`, the file is named `pageviews.dat`, and the release is published, not a draft. |
| **"No documents yet."** | No published release contains a file ending in `.pdf` | Upload PDFs to a release (Step 2 or Step 6). |
| Numbers don't change after downloads or visits | Two delays add up: GitHub updates counts about 10–15 minutes late, and the job publishes them about every 15 minutes | Wait up to about 30 minutes, then hard-reload. To speed things up, click **Run workflow** in the Actions tab. |
| **"Could not load the list. Please try again later."** | `stats.json` is unavailable **and** the backup route hit GitHub's limit of 60 requests per hour per connection, which is common after many refreshes or on shared Wi-Fi | Check the **Actions** tab. If the last run failed or the job is paused, see the next rows. Otherwise wait up to an hour. |
| No *"Numbers last updated"* line at the bottom | The page couldn't read `stats.json` and used the backup route | Check that Pages **Source** is **GitHub Actions** (Step 5, point 1) and that the latest Actions run is green. |
| Actions run has a **red cross** | Usually Pages **Source** is not set to **GitHub Actions**, or the job file has a typo | Open the failed run to read the error. Fix the Pages source, or re-paste `publish.yml` exactly, then click **Re-run jobs**. |
| Actions tab shows a banner that the scheduled workflow was **disabled** | The 60-day inactivity pause | Click **Enable workflow**, or commit any small change. See Step 5. |
| A new folder or PDF doesn't appear | The job hasn't run since the change, the release is still a **draft**, or the file doesn't end in `.pdf` | Click **Run workflow** in Actions, publish the release, or rename the file to end in `.pdf`. |
| A folder has the wrong name | The folder name is the release **title** | Edit the release and change its title, then run the job. |
| A PDF appears twice | The same file is in two releases, for example after a move | Delete it from the release where it shouldn't be. |
| A logo shows only text, or a broken image | The `src` path doesn't match the uploaded file (case-sensitive), or the file isn't in `logos/` | Check the file name and folder in the repository and correct `src` in `index.html`. |
| A logo is tiny or squashed | Large empty margins in the image, or an unusual shape | Crop the image tightly. A roughly 3:1 width-to-height ratio works best. |
| Visitor count doesn't go up when **I** open the page | Intended: your browser has already been counted | Test from another device, another browser, or a private window. |
| My own visits should count again | The "already counted" marker is stored in your browser | Clear site data for `cristinaeu1.github.io` in the browser settings. |
| Page changes don't appear after uploading `index.html` | Publishing takes 1–2 minutes, and the browser may show an old copy | Wait for the green tick in **Actions**, then hard-reload. |
| Visitor count seems lower than expected | Some ad-blockers or privacy extensions block the hidden counter request | Nothing to fix on the page. Treat the count as a minimum. |
| Download counts seem higher than expected | Link-preview bots (Slack, WhatsApp, Teams) and crawlers also download files | Expected. Treat the count as approximate. |
| No **Build and deployment** section in Settings → Pages | You are in account settings, not repository settings | Use https://github.com/CristinaEU1/IEEE-RO-resources/settings/pages |

---

## 5. Settings

### In `index.html` (top of the `<script>` block)

```js
const GH_OWNER   = "CristinaEU1";        // GitHub account
const GH_REPO    = "IEEE-RO-resources";  // repository name
const GH_TAG     = "v1";                 // release that holds pageviews.dat
const VIEW_ASSET = "pageviews.dat";      // the visitor-counter file

const LOGOS = [                          // the three logo spaces, left to right
  { src: "", alt: "Logo 1", link: "" },
  { src: "", alt: "Logo 2", link: "" },
  { src: "", alt: "Logo 3", link: "" },
];

const FOLDERS_OPEN = true;               // false = folders start collapsed
```

- **Moving or renaming the repository:** update `GH_OWNER` and `GH_REPO`. The Actions job adapts automatically.
- **Changing the visitor marker:** changing `SEEN_KEY` (in the visitor block) makes every browser count once more, which is effectively a reset of who counts as "seen".
- **More or fewer logos:** add or remove lines in `LOGOS`. The row is laid out for three, so with a different number, also change `repeat(3, 1fr)` in the `.logos` style to the new number.

### In `.github/workflows/publish.yml`

- **Refresh frequency:** the line `- cron: "*/15 * * * *"`. For example, `"*/30 * * * *"` means every 30 minutes. GitHub doesn't run scheduled jobs more often than every 5 minutes.
- **Published files:** the job publishes `index.html`, the `logos/` folder (if it exists), and `stats.json`. To publish another file, add a line under **Build the site folder**, for example `cp favicon.ico _site/`.

---

## 6. Known limits

- **"Unique" means unique per browser, not per person.** The same person on a phone and a laptop counts twice. Private-browsing sessions and cleared browser data also count again. Exact unique-person counting needs a server, which this free GitHub-only setup doesn't have.
- **Numbers on the page are typically 15–30 minutes behind.** That is GitHub's own counting delay plus the refresh schedule.
- **Counts can't be edited, moved or merged.** They can only be reset, by deleting and re-uploading the file. Moving a PDF to another folder restarts its count.
- **Folders are one level deep.** There are no sub-folders, because each release is one folder.
- **Each file can be at most 2 GB.** That limit is far above any PDF.
- **There is no history over time.** GitHub keeps totals only. For monthly figures, save `stats.json` on a fixed date each month.
- **Scheduled jobs pause after 60 days without repository activity.** See Step 5.

---

## 7. Change log

| Date | Change |
|---|---|
| 2026-10-03 | Repository created; `index.html` with page-view and download counters |
| 2026-10-31 | GitHub Pages enabled (`main` / root); release `v1` created with `pageviews.dat` and test PDFs (`test1.pdf`, `Test2.pdf`) |
| 2026-10-31 | Visitor counter changed from every page view to **once per browser** (label now "Unique visitors") |
| 2026-10-31 | GitHub's 60-requests-per-hour limit hit during testing → **GitHub Actions job** added (`publish.yml`): numbers published as `stats.json` about every 15 minutes, with the direct API kept as a backup; Pages source switched to **GitHub Actions**; runbook added |
| 2026-10-31 | **Three logo placeholders** added at the top; PDFs grouped into **collapsible folders, one per release**; job now records each file's folder and publishes `logos/` |
| *open* | Step 4: reset counters and add the final PDFs and folders (after MVP validation with the customer) |
| *open* | Real logos to be added in place of the placeholders |
