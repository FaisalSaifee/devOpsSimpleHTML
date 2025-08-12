# Simple HTML Site on Azure Static Website (Azure DevOps CI/CD)

This repo showcases a **working static HTML site** deployed to **Azure Storage Static Website** using **Azure DevOps Pipelines**.

- **Live hosting**: `$web` container of an Azure Storage account  
- **CI/CD**: `azure-pipelines.yml`  
- **Artifacts**: versioned ZIP per run + a “latest” ZIP for convenience  
- **Versioned copies in Storage** (optional): files uploaded to `releases/<buildnumber>/`

---

## What this pipeline does

1. **Build name** → `YYYYMMDD-HHMMSS.r` (timestamped for clarity).  
2. **Archives** your site folder into `site-<buildnumber>.zip`.  
3. **Publishes two artifacts**:
   - `site-<buildnumber>.zip` (versioned, kept per retention policy)
   - `latest.zip` (always the newest)
4. **Enables Static Website** on the storage account (idempotent).  
5. **Deploys** the site files to the `$web` container (overwrites old files).  
6. **(Optional)** Uploads a **versioned copy** of the files to `releases/<buildnumber>/` for easy rollbacks.

---

## Repository layout

```
.
├─ index.html             # Your site entry point (if at repo root)
├─ ...                    # Any other static assets (css/js/img)
└─ azure-pipelines.yml    # CI/CD pipeline
```

> If your site isn’t at the repo root, change `siteFolder` in the YAML (e.g., `public`, `dist`).

---

## Prerequisites

- An **Azure subscription** and **Storage account (StorageV2)** with a resource group:
  - Resource Group: `SimpleHTMLSite`
  - Storage Account: `simplehtmlsite`
- An **Azure DevOps Service Connection** to the correct subscription/tenant:
  - Name: `AzureConnection-Dev` (or update `azure-pipelines.yml`)
  - Permissions:
    - **Contributor** at subscription or resource group scope (to read account keys)
    - (Optional) **Storage Blob Data Contributor** if using RBAC upload (`--auth-mode login`)

---

## Pipeline variables (in `azure-pipelines.yml`)

- `azureSubscription` — Azure RM Service Connection name  
- `resourceGroupName` — Resource group containing the storage account  
- `storageAccountName` — Storage account that hosts the site  
- `siteFolder` — Folder to deploy (`.` means repo root)  
- `artifactBase` — Prefix for artifact names (`site`)  
- `version` — Uses `$(Build.BuildNumber)` (auto—don’t change)

---

## How to run

1. Push to **main** (pipeline triggers on `main` by default), or run the pipeline manually.  
2. Azure DevOps → **Pipelines** → select this pipeline → **Run** (if you prefer manual).

---

## Where to see results

### 1) Artifacts (per run)
- Azure DevOps → **Pipelines → [your pipeline] → [a run] → Artifacts**  
- You’ll see:
  - `site-<buildnumber>.zip` (versioned)
  - `latest` (contains `latest.zip`)

> Retention affects how many historic runs/artifacts are kept. For critical builds, click **… → Keep forever** on the run.

### 2) Deployed website URL
- Azure Portal → Storage account `simplehtmlsite` → **Static website** → **Primary endpoint**  
  Example:  
  `https://simplehtmlsite.z9.web.core.windows.net/`

- The pipeline also prints the URL at the end of the “Enable static website + upload…” step.

### 3) Versioned copy in Storage (optional)
- Azure Portal → Storage account → **Containers** → `releases` → `/<buildnumber>/`  
  Contains the deployed files for that release.

---

## Rollback guide (two options)

**Option A — From Storage (files)**  
1. In `releases/<oldBuildNumber>/`, **Select All** → **Copy** → paste into `$web` (overwrite).  
2. Site instantly reflects the older version.

**Option B — From Artifact (ZIP)**  
1. Download `site-<oldBuildNumber>.zip` from the old run.  
2. Extract locally.  
3. Re-upload the extracted folder to `$web` (via Portal or Azure CLI):

```bash
RG="SimpleHTMLSite"
ACC="simplehtmlsite"
SRC="/path/to/extracted/folder"

KEY=$(az storage account keys list -g "$RG" -n "$ACC" --query "[0].value" -o tsv)
az storage blob upload-batch   --account-name "$ACC"   --account-key "$KEY"   --destination '$web'   --source "$SRC"   --overwrite
```

---

## Troubleshooting

- **`siteFolder not found`**  
  Ensure `siteFolder` points to the folder that contains `index.html`. If your files are at repo root, use `.`.

- **`The subscription … doesn't exist in cloud 'AzureCloud'`**  
  Fix your **service connection**:
  - Correct tenant and subscription selected (from dropdown)
  - Environment is **AzureCloud**
  - Re-authorize and ensure **Contributor** role

- **Can’t list storage keys**  
  The service principal needs **Contributor** on the storage account/subscription.  
  Or switch upload to RBAC and grant **Storage Blob Data Contributor**:
  ```bash
  az storage blob upload-batch --auth-mode login ...
  ```

- **Static website 404**  
  Confirm `$web` has `index.html` at its root and the **Static website** feature is **Enabled**.

---

## Useful commands (local)

Enable Static Website (one-time, idempotent):
```bash
az storage blob service-properties update   --account-name simplehtmlsite   --static-website true   --index-document index.html   --404-document error.html
```

Upload (overwrite):
```bash
KEY=$(az storage account keys list -g SimpleHTMLSite -n simplehtmlsite --query "[0].value" -o tsv)
az storage blob upload-batch   --account-name simplehtmlsite   --account-key "$KEY"   --destination '$web'   --source .   --overwrite
```

Show site URL:
```bash
az storage account show -g SimpleHTMLSite -n simplehtmlsite --query "primaryEndpoints.web" -o tsv
```

---

## Notes

- Build names include **timestamp**: `YYYYMMDD-HHMMSS.r`  
- The pipeline uploads with `--overwrite`, so the site always reflects the newest run.  
- Use **Retention policies** (Project settings → Pipelines → Settings → Retention) and **Keep forever** for important releases.

---

**That’s it!** This repo demonstrates a complete CI/CD flow for a static site on Azure Storage Static Website with versioned artifacts and easy rollback.
