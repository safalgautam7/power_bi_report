# Power BI Trading Report

Power BI project source for the company-format **QTD Trading Dashboard**,
**Running Total**, and supporting **Department Analysis** pages.

This is a private, authorized development repository. Do not make it public
without reviewing the report, business logic, branding, connection details
and documentation for organizational approval.

## Project structure

```text
report.pbip
report.Report/
report.SemanticModel/
report_ARCHITECTURE_GUIDE.md
report_TEMPLATE_NOTES.md
report_RUNNING_TOTAL_NOTES.md
```

Keep the PBIP file and its two linked folders together. The report uses a
relative reference to `../report.SemanticModel`; uploading only `report.pbip`
does not upload a working project.

## Continue at home using a Windows VM on Linux

1. Install a supported Windows version in your VM and install current
   **Power BI Desktop**. Allow adequate memory and disk for Windows, Desktop
   and the imported model. Power BI Desktop does not run natively on Linux.
2. Clone this private repository using an authorized GitHub account. Git can
   run on the Linux host/WSL or inside the VM; open the project using a path
   accessible to Windows. For reliable editing, use a Windows-local checkout
   or working copy rather than editing the same files concurrently from two OSes.
3. Open `report.pbip` in Power BI Desktop. Keep a personal working copy separate
   from the original office files.
4. Sign in to the configured Snowflake source using authorized credentials.
   Account/network restrictions or your organization's VPN may apply. The
   configured warehouse, database and views must be available to that account.
5. Refresh the model to import the business data and calculate helper objects.
6. Check both report pages, read the architecture guide, then save your work.

### Important: Git stores definitions, not the imported data

`.pbi/cache.abf` contains the local imported model/data snapshot. It is
intentionally ignored by Git, as are data-containing PBIX packages.

The initial local copy beside Documents may contain an ignored cache copied
from the office project. **That cache is not uploaded and will not exist in a
fresh clone.** A fresh clone can open with model definitions but requires a
source refresh to display data.

Snowflake credentials are not supplied by this repository. If you need an
offline data snapshot, arrange a separate company-approved secure transfer;
do not remove the ignore rules and commit cached data to solve that problem.

## Documentation

- [Architecture guide](report_ARCHITECTURE_GUIDE.md): five physical measures,
  six calculation groups, helper tables, DAX walkthroughs and exercises.
- [QTD template notes](report_TEMPLATE_NOTES.md): approved company layout and
  calculation behaviour.
- [Running Total notes](report_RUNNING_TOTAL_NOTES.md): running-page format,
  day-of-quarter charts, date mapping and checks.

The layout notes refer to office backup directories for historical context.
Those backups are not part of this repository and their paths do not exist
automatically on a home machine.

## Working safely with Git

Before starting a session:

```bash
git pull --ff-only
```

Save your work in Desktop before reviewing and committing changes. Close
Desktop before pulling changes that could conflict with its in-memory project.
Do not edit the same checkout concurrently from office, home and a shared folder.

Review the exact files to be uploaded:

```bash
git status --short
git diff
git add report.pbip report.Report report.SemanticModel
git add README.md .gitignore report_ARCHITECTURE_GUIDE.md report_TEMPLATE_NOTES.md report_RUNNING_TOTAL_NOTES.md
git diff --cached --stat
git diff --cached
git commit -m "Update Power BI report"
git push origin main
```

Do not use force-push to work around divergent office/home histories. Fetch
and review the changes first. If a remote branch already has independent
history, preserve that work instead of overwriting it.

### Files that must stay out of Git

- Imported caches, PBIX snapshots and backup copies.
- Machine-specific local settings and pending local query changes.
- Credentials, tokens, `.env` files and private keys.
- The private `scripts.md` presentation notes.

Shared `.pbi/editorSettings.json` files are retained: these are project editor
metadata, not the imported data cache. The `.platform` metadata, definitions,
themes and logo resources belong with the project.

## Origin and office copy

Remote: `https://github.com/safalgautam7/power_bi_report.git`

Initial office working directory:

```text
C:\Users\safal.gautam\power_bi_report
```

It was created by **copying**, not moving, the saved project from Documents.
The original project remains in `C:\Users\safal.gautam\Documents`.
Once you choose which working copy to use, save future edits there explicitly;
the Documents copy and this repository do not automatically synchronize.
