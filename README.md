This guide summarizes the professional workflow for versioning large datasets using **Git** (for code) and **DVC** (for data), specifically utilizing **Google Drive** as a low-cost, collaborative remote storage.

---

## 1. Project Architecture
The core principle is to keep the heavy data out of your Git history while maintaining a "pointer" that tells Git which version of the data belongs to which version of the code.



| Component | Responsibility | Stored In |
| :--- | :--- | :--- |
| **Source Code** | Logic, scripts, configurations | Git (GitHub/GitLab) |
| **Large Data** | Datasets, model weights, binaries | DVC (Google Drive) |
| **Metadata** (`.dvc` files) | Links code commits to data hashes | Git (GitHub/GitLab) |

---

## 2. Initial Setup
Run these commands in your project root to initialize the environment:

```bash
# Install DVC with Google Drive support
pip install "dvc[gdrive]"

# Initialize version control
git init
dvc init

# Commit the initial DVC metadata
git commit -m "Initialize DVC"
```

---

## 3. Configuring the Google Drive Remote
1.  **Create a Folder:** Create a folder in Google Drive and copy the **Folder ID** from the browser URL.
2.  **Add Remote:**
    ```bash
    dvc remote add -d mygdrive gdrive://YOUR_FOLDER_ID_HERE
    git add .dvc/config
    git commit -m "Configure Google Drive remote"
    ```

---

## 4. The Core Workflow
Use these steps every time you add or update data:

1.  **Track Data:** `dvc add data/my_large_folder`
    * *This moves the data to a local cache and creates a `.dvc` pointer.*
2.  **Version Metadata:** ```bash
    git add data/my_large_folder.dvc .gitignore
    git commit -m "Tracked new dataset version"
    ```
3.  **Upload to Cloud:** `dvc push`
    * *First-time users will see a browser popup to authorize Google Drive access.*



---

## 5. Security & Secret Management
**CRITICAL:** Never commit credentials to the main `.dvc/config` file. GitHub's secret scanning will block your push if you do.

* **Public Config (`.dvc/config`):** Contains only the Folder ID.
* **Private Config (`.dvc/config.local`):** Contains your `client_id` or `client_secret`.
* **Best Practice:** If you accidentally commit a secret, use `git commit --amend` after cleaning the file to rewrite the history before pushing.

---

## 6. Team Collaboration
For a teammate to access the project:

1.  **Sync Code:** `git pull`
2.  **Sync Data:** `dvc pull`
    * *DVC reads the `.dvc` file and automatically downloads the correct version from the shared Google Drive folder.*

---

## 7. Troubleshooting Common Errors

| Error | Cause | Solution |
| :--- | :--- | :--- |
| **Secret Scanning Block** | Committed `client_secret` to Git | Move secrets to `dvc remote modify --local` and amend the commit. |
| **Extra Keys Not Allowed** | Syntax error in `.dvc/config` | Use `dvc remote modify --unset` to clear the error, then re-add with correct syntax. |
| **Permission Denied** | GDrive folder not shared | Ensure the folder is shared with your team's emails (as Editors). |



---

## 8. Summary of Essential Commands

* `dvc status`: Check for data changes.
* `dvc pull`: Download data from the cloud.
* `dvc push`: Upload data to the cloud.
* `dvc checkout`: Sync local files to match the current Git branch.
* `dvc remote modify --local`: Set private credentials safely.




## what to do if you added/modifed local large data file

Here is the technical guide formatted as a Markdown (`.md`) file. You can copy and paste this directly into your project's `CONTRIBUTING.md` or a dedicated `DATA_GUIDE.md`.

---

# Data Update Workflow (DVC & Git)

This guide outlines the standard operating procedure for updating data files within the tracked `large_data/` directory. Because DVC uses "snapshots" rather than live-tracking, these steps must be followed every time the contents of the folder change.

## Prerequisites
* Ensure you have the latest version of the code and data pointers:
    ```bash
    git pull
    dvc pull
    ```

---

## Step-by-Step Update Sequence

### 1. Identify Changes
After adding, deleting, or modifying files inside the `large_data/` folder, verify that DVC detects the modification:
```bash
dvc status
```
*DVC should report that `large_data` is "modified".*

### 2. Update the DVC Snapshot
Take a new snapshot of the folder. This updates the local cache and the metadata pointer.
```bash
dvc add large_data
```
**What this does:**
* Scans the folder for new/changed files.
* Hashes the new data and stores it in `.dvc/cache`.
* Updates `large_data.dvc` with the new folder hash.



### 3. Record the Version in Git
Link the new data state to your current code state by committing the updated pointer.
```bash
# Stage the updated metadata
git add large_data.dvc

# Commit the change
git commit -m "data: added new training samples to large_data"
```

### 4. Sync to Cloud Storage
Upload the physical data to Google Drive and the pointers to GitHub so the rest of the team can access the changes.
```bash
# Upload data to Google Drive
dvc push

# Upload pointers to GitHub
git push origin <your-branch-name>
```

---

## Troubleshooting & Best Practices

### Why do I need to `dvc add` again?
DVC does not automatically watch folders for changes. Running `dvc add` is the equivalent of taking a versioned "save point" of your dataset. Without this step, your `.dvc` file will still point to the old version of the folder.

### The "Golden Rule"
**Never** push your Git code without also running `dvc push`. If a teammate pulls your code but the data hasn't been uploaded to Google Drive, their `dvc pull` will fail with a "File not found" error.

### Reverting Changes
If you make a mistake and want to revert your local data to the last committed version:
```bash
# Revert the pointer file
git checkout large_data.dvc

# Revert the actual data
dvc checkout
```

### how other users can pull from dvc
- dvc remote modify --local mygdrive gdrive_client_id your_actual_id
- dvc remote modify --local mygdrive gdrive_client_secret your_actual_secret