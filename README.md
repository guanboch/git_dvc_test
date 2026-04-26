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
| **OAuth2 browser popup despite service account JSON being set** | `gdrive_use_service_account` flag is missing | Run `dvc remote modify --local mygdrive gdrive_use_service_account true`. Having the JSON path alone is not enough. |



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


### use google service accounts

This guide provides a professional, step-by-step tutorial for using **DVC with a Google Service Account**. This method is superior for teams and automated environments (like GPU clusters or CI/CD) because it uses a dedicated "robot" identity rather than a personal login.

---

### Phase 1: Google Cloud Platform (GCP) Configuration

First, you must create the "identity" for DVC within the Google Cloud ecosystem.

1.  **Create a Project:** Go to the [Google Cloud Console](https://console.cloud.google.com/). If you don't have a project, create one (e.g., `dvc-data-project`).
2.  **Enable the Drive API:**
    * In the search bar, type **"Google Drive API"**.
    * Click on it and select **Enable**.
3.  **Create a Service Account:**
    * Navigate to **IAM & Admin > Service Accounts**.
    * Click **+ Create Service Account**.
    * Name it (e.g., `dvc-manager`). Click **Create and Continue**.
    * For the role, you can leave it blank or select **Project > Viewer** (the actual permissions are handled in Google Drive, not here). Click **Done**.
4.  **Download the JSON Key:**
    * Click on the email address of your new service account.
    * Go to the **Keys** tab.
    * Click **Add Key > Create new key**.
    * Select **JSON** and click **Create**. 
    * **Crucial:** A `.json` file will download. Move this file to a secure folder *outside* of your Git repository (e.g., `~/keys/dvc-key.json`).

---

### Phase 2: Link Google Drive to the Service Account

By default, the Service Account has access to nothing. You must explicitly share your data folder with it.

1.  **Copy the Email:** In the GCP Console, copy the email address of your service account (e.g., `dvc-manager@project-id.iam.gserviceaccount.com`).
2.  **Share the Folder:** * Open Google Drive in your browser.
    * Right-click the folder you intend to use for DVC storage.
    * Click **Share**.
    * Paste the **Service Account email** and set the permission to **Editor**.

---

### Phase 3: DVC Implementation

Now, configure DVC to use this "robot" identity instead of your browser.

1.  **Initialize the Remote (Shared with Team):**
    These settings are stored in `.dvc/config` and are pushed to Git.
    ```bash
    # Add the remote URL (replace FOLDER_ID with your Drive folder's ID)
    dvc remote add -d mygdrive gdrive://YOUR_FOLDER_ID_HERE
    
    # Allow large file downloads (bypasses virus scan warnings)
    dvc remote modify mygdrive gdrive_acknowledge_abuse true
    ```

2.  **Add the Credentials (Local Only):**
    You must use the `--local` flag so your personal file path is not pushed to GitHub.
    ```bash
    # Point DVC to your JSON key
    dvc remote modify --local mygdrive gdrive_service_account_json_file_path /path/to/your/keys/dvc-key.json

    # Tell DVC to actually USE the service account (required — without this, DVC still falls back to OAuth2 browser login)
    dvc remote modify --local mygdrive gdrive_use_service_account true
    ```

---

### Phase 4: The Versioning Workflow

With authentication handled, you can now manage your data.

1.  **Track a Folder:**
    ```bash
    dvc add data/large_dataset/
    ```
2.  **Commit Pointers:**
    ```bash
    git add data/large_dataset.dvc .gitignore
    git commit -m "Tracked large dataset using Service Account"
    ```
3.  **Push Data:**
    ```bash
    dvc push
    ```
    *Note: Unlike the browser method, this will NOT open a window. It will simply upload the data in the background.*

---

### Phase 5: Team and Security Standards

#### **How Teammates Join**
Each teammate follows these steps:
1.  `git pull` the repository.
2.  Obtain the JSON key (securely shared via a password manager or internal vault).
3.  Run the **Local Only** command from Phase 3, Step 2 to point to their local copy of the JSON key.
4.  `dvc pull`.

#### **The Security "Firewall"**
To ensure you never accidentally push the JSON key to GitHub, verify your `.gitignore` contains these lines:
```text
# DVC local settings (where your JSON path lives)
.dvc/config.local

# If you mistakenly put keys in the project, ignore them
*.json
```

#### **Why this is the Industry Standard**
* **Headless Support:** Works on remote Linux servers and SSH sessions without a GUI.
* **Safety:** If the JSON key is compromised, the attacker only sees the specific Google Drive folder you shared, not your personal Gmail or Photos.
* **Reliability:** Service Account keys do not expire like browser session tokens, meaning your automated scripts will never break due to a "login timeout."