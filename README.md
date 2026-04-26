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