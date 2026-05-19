# Complete Kubernetes GKE Implementation Guide
# From Zero to Production — Exact Steps, Errors & Fixes

---

## PROJECT OVERVIEW

**What we built:**
- Python Flask web app deployed on Google Kubernetes Engine (GKE)
- Fully automated GitOps pipeline: code push → auto build → auto deploy
- ArgoCD for continuous deployment
- GitHub Actions for CI/CD

**Final Result:**
- App live at: http://136.116.116.254
- ArgoCD dashboard at: https://34.57.95.151
- Push code → automatically builds Docker image → automatically deploys to GKE
- Zero manual steps after `git push`

---

## ACCOUNTS & CREDENTIALS

| Item | Value |
|------|-------|
| GCP Account | akhilaakhikatepalli@gmail.com |
| GCP Project ID | gke-learning-2025 |
| Billing Account | 0104F5-48534D-553BCB |
| GKE Cluster | gke-learning-cluster |
| Zone | us-central1-a |
| GitHub Username | Vamsi-90 |
| GitHub Repo | https://github.com/Vamsi-90/gke-learning-app |
| Artifact Registry | us-central1-docker.pkg.dev/gke-learning-2025/gke-repo |
| Service Account | github-actions-sa@gke-learning-2025.iam.gserviceaccount.com |

---

## TOOLS REQUIRED (Pre-installed)

```bash
# Verify these are installed before starting
gcloud --version       # Google Cloud SDK 567.0.0
kubectl version        # Kubernetes CLI
docker --version       # Docker 29.3.1
git --version          # Git 2.39.3
```

---

## ARCHITECTURE

```
Developer pushes code
        ↓
GitHub (Vamsi-90/gke-learning-app)
        ↓
GitHub Actions triggers automatically
  Step 1: Authenticate to GCP
  Step 2: Build Docker image with commit SHA tag
  Step 3: Push image to Artifact Registry
  Step 4: Update k8s/deployment.yaml with new image tag
  Step 5: Push updated deployment.yaml back to GitHub
        ↓
ArgoCD (running inside GKE) detects deployment.yaml changed
  → Pulls new image from Artifact Registry
  → Deploys new pods
  → Keeps old pods running until new ones are healthy
        ↓
App live at public IP — zero downtime
```

---

## FILE STRUCTURE

```
gke-learning-app/
├── app.py                              # Python Flask app
├── requirements.txt                    # Python dependencies
├── Dockerfile                          # Docker build instructions
├── argocd-app.yaml                     # ArgoCD Application config
├── k8s/
│   ├── deployment.yaml                 # Kubernetes Deployment
│   └── service.yaml                    # Kubernetes Service (LoadBalancer)
├── templates/
│   └── index.html                      # Frontend HTML
├── static/
│   └── photo.jpg                       # Static assets
└── .github/
    └── workflows/
        └── build.yaml                  # GitHub Actions CI/CD pipeline
```

---

# PHASE 1 — GCP SETUP

## Step 1 — Login to GCP

```bash
gcloud auth login
# Opens browser → sign in with Google account
```

Verify login:
```bash
gcloud projects list
```

---

## Step 2 — Create GCP Project

```bash
gcloud projects create gke-learning-2025 --name="GKE Learning"
gcloud config set project gke-learning-2025
```

**NOTE:** If project ID is taken, try `gke-learning-2025-v2` or similar.

**ERROR FACED:**
```
ERROR: Project creation failed. The project ID you specified is already in use.
```
**FIX:** Use a different project ID — add numbers or suffix.

---

## Step 3 — Link Billing Account

```bash
# Find your billing account ID first
# Go to: console.cloud.google.com/billing
# Copy Account ID format: XXXXXX-XXXXXX-XXXXXX

gcloud billing projects link gke-learning-2025 --billing-account=0104F5-48534D-553BCB
```

Expected output:
```
billingEnabled: true
projectId: gke-learning-2025
```

---

## Step 4 — Enable Required APIs

```bash
gcloud services enable container.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com
```

**What each API does:**
- `container.googleapis.com` → GKE (Kubernetes Engine)
- `artifactregistry.googleapis.com` → Docker image storage
- `cloudbuild.googleapis.com` → Cloud Build service

---

## Step 5 — Set Up Billing Alert (IMPORTANT — avoid surprise charges)

```bash
gcloud services enable billingbudgets.googleapis.com --project=gke-learning-2025

gcloud billing budgets create --billing-account=0104F5-48534D-553BCB --display-name="GKE Learning - $250 Alert" --budget-amount=250USD --threshold-rule=percent=0.8 --threshold-rule=percent=1.0
```

This sends email alerts at $200 (80%) and $250 (100%).

---

## Step 6 — Create GKE Cluster

```bash
gcloud container clusters create gke-learning-cluster --zone=us-central1-a --num-nodes=2 --machine-type=e2-medium --disk-size=20
```

**Takes 3-5 minutes.**

Expected output:
```
NAME                  LOCATION       STATUS
gke-learning-cluster  us-central1-a  RUNNING
```

**NOTE:** Run as a single line — multiline with backslash `\` causes errors in zsh:

**ERROR FACED:**
```
ERROR: unrecognized arguments:
zsh: command not found: --zone
```
**FIX:** Always run gcloud commands as a single line, not multiline with `\`.

---

## Step 7 — Install gke-gcloud-auth-plugin

```bash
gcloud components install gke-gcloud-auth-plugin
```

This is required for kubectl to work with GKE.

**ERROR FACED (without this):**
```
CRITICAL: gke-gcloud-auth-plugin, which is needed for continued use of kubectl, was not found
```

---

## Step 8 — Connect kubectl to Cluster

```bash
gcloud container clusters get-credentials gke-learning-cluster --zone=us-central1-a --project=gke-learning-2025
```

Verify connection:
```bash
kubectl get nodes
# Should show 2 nodes with STATUS: Ready
```

---

# PHASE 2 — APPLICATION CODE

## Step 9 — Create GitHub Repo

1. Go to github.com → click "+" → "New repository"
2. Name: `gke-learning-app`
3. Visibility: Public
4. Check "Add a README file"
5. Click "Create repository"

---

## Step 10 — Clone Repo Locally

```bash
mkdir ~/gke-project
cd ~/gke-project
git clone https://github.com/Vamsi-90/gke-learning-app.git
cd gke-learning-app
```

**ERROR FACED:**
```
remote: Repository not found.
fatal: repository not found
```
**FIX:** Check exact GitHub username and repo name. Run `git remote -v` to verify URL.

---

## Step 11 — Create app.py

```python
# app.py
from flask import Flask, render_template

app = Flask(__name__)

@app.route("/")
def home():
    return render_template("index.html")

@app.route("/health")
def health():
    return "OK", 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

**Key points:**
- `/health` route is CRITICAL — Kubernetes uses it to check if app is alive
- `host="0.0.0.0"` — must be this, not `localhost` or `127.0.0.1`
- `port=8080` — must match containerPort in deployment.yaml

---

## Step 12 — Create requirements.txt

```
flask==3.0.0
```

---

## Step 13 — Create Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .
COPY templates/ templates/
COPY static/ static/

EXPOSE 8080

CMD ["python", "app.py"]
```

**CRITICAL ERROR FACED:**
```
Internal Server Error
The server encountered an internal error
```
**CAUSE:** Dockerfile only had `COPY app.py .` — did NOT copy templates/ and static/ folders.

**FIX:** Add these lines to Dockerfile:
```dockerfile
COPY templates/ templates/
COPY static/ static/
```

---

# PHASE 3 — KUBERNETES YAML FILES

## Step 14 — Create k8s Directory

```bash
mkdir k8s
```

---

## Step 15 — Create k8s/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gke-learning-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gke-learning-app
  template:
    metadata:
      labels:
        app: gke-learning-app
    spec:
      containers:
      - name: gke-learning-app
        image: us-central1-docker.pkg.dev/gke-learning-2025/gke-repo/gke-learning-app:latest
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
```

**Key fields:**
- `replicas: 2` → always keep 2 pods running
- `readinessProbe` → only send traffic after app is ready
- `livenessProbe` → restart pod if /health check fails
- `image` → will be auto-updated by GitHub Actions with commit SHA

**CRITICAL ERROR FACED:**
```
error converting YAML to JSON: yaml: line 2: mapping values are not allowed in this context
```
**CAUSE:** Leading spaces at start of file. YAML is strict about indentation.

**FIX:** Never use `cat > file << 'EOF'` with indented content. Always write files using a code editor or Claude's Write tool directly. The `EOF` heredoc adds leading spaces if the command is indented.

---

## Step 16 — Create k8s/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gke-learning-app
spec:
  type: LoadBalancer
  selector:
    app: gke-learning-app
  ports:
  - port: 80
    targetPort: 8080
```

**Key fields:**
- `type: LoadBalancer` → GCP creates a public IP automatically
- `port: 80` → internet-facing port
- `targetPort: 8080` → forwards to Flask app inside pod

---

# PHASE 4 — GITHUB ACTIONS CI/CD

## Step 17 — Create Artifact Registry Repository

```bash
gcloud artifacts repositories create gke-repo --repository-format=docker --location=us-central1 --description="GKE Learning App Images"
```

---

## Step 18 — Create GCP Service Account for GitHub Actions

```bash
# Create service account
gcloud iam service-accounts create github-actions-sa --display-name="GitHub Actions Service Account"

# Grant permission to push Docker images
gcloud projects add-iam-policy-binding gke-learning-2025 --member="serviceAccount:github-actions-sa@gke-learning-2025.iam.gserviceaccount.com" --role="roles/artifactregistry.writer"

# Create JSON key file
gcloud iam service-accounts keys create ~/gke-sa-key.json --iam-account=github-actions-sa@gke-learning-2025.iam.gserviceaccount.com
```

---

## Step 19 — Add GCP Key as GitHub Secret

1. Copy contents of `~/gke-sa-key.json`:
```bash
cat ~/gke-sa-key.json
```

2. Go to: github.com/Vamsi-90/gke-learning-app
3. Settings → Secrets and variables → Actions
4. Click "New repository secret"
5. Name: `AKHILA_GCP_SA_KEY`
6. Value: paste entire JSON content
7. Click "Add secret"

**NOTE:** Secret name used in this project is `AKHILA_GCP_SA_KEY` (not the default `GCP_SA_KEY`).

---

## Step 20 — Create GitHub Actions Workflow

Create `.github/workflows/build.yaml`:

```yaml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v2
      with:
        credentials_json: ${{ secrets.AKHILA_GCP_SA_KEY }}

    - name: Configure Docker for Artifact Registry
      run: gcloud auth configure-docker us-central1-docker.pkg.dev

    - name: Build Docker image
      run: |
        docker build -t us-central1-docker.pkg.dev/gke-learning-2025/gke-repo/gke-learning-app:${{ github.sha }} .
        docker tag us-central1-docker.pkg.dev/gke-learning-2025/gke-repo/gke-learning-app:${{ github.sha }} us-central1-docker.pkg.dev/gke-learning-2025/gke-repo/gke-learning-app:latest

    - name: Push Docker image
      run: |
        docker push us-central1-docker.pkg.dev/gke-learning-2025/gke-repo/gke-learning-app:${{ github.sha }}
        docker push us-central1-docker.pkg.dev/gke-learning-2025/gke-repo/gke-learning-app:latest

    - name: Update deployment.yaml with new image tag
      run: |
        sed -i "s|image: us-central1-docker.pkg.dev/gke-learning-2025/gke-repo/gke-learning-app:.*|image: us-central1-docker.pkg.dev/gke-learning-2025/gke-repo/gke-learning-app:${{ github.sha }}|g" k8s/deployment.yaml

    - name: Commit and push updated deployment.yaml
      run: |
        git config user.email "github-actions@github.com"
        git config user.name "GitHub Actions"
        git remote set-url origin https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/Vamsi-90/gke-learning-app.git
        git add k8s/deployment.yaml
        git commit -m "Update image tag to ${{ github.sha }}"
        git push
```

**CRITICAL POINTS:**
- `permissions: contents: write` — REQUIRED for workflow to push back to repo
- `secrets.GITHUB_TOKEN` — built-in GitHub token, no setup needed
- `sed` command auto-updates image tag in deployment.yaml
- Secret name must match exactly: `AKHILA_GCP_SA_KEY`

**ERROR 1 FACED:**
```
refusing to allow a Personal Access Token to create or update workflow without `workflow` scope
```
**FIX:** When creating GitHub Personal Access Token, check BOTH `repo` AND `workflow` scopes.

**ERROR 2 FACED:**
```
could not read Username for 'https://github.com': terminal prompts disabled
```
**CAUSE:** Workflow trying to git push without authentication.

**FIX:** Add these lines to the push step:
```bash
git remote set-url origin https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/Vamsi-90/gke-learning-app.git
```
AND add `permissions: contents: write` to the job.

---

## Step 21 — Push All Files to GitHub

```bash
git add .
git commit -m "Initial project setup"
git push origin main
```

**ERROR FACED:**
```
remote: Permission to Vamsi-90/gke-learning-app.git denied to MrCOOL1732
```
**CAUSE:** Multiple GitHub accounts on machine. Wrong account being used.

**FIX:** Update remote URL with correct token:
```bash
git remote set-url origin https://Vamsi-90:YOUR_GITHUB_TOKEN@github.com/Vamsi-90/gke-learning-app.git
git push origin main
```

**ERROR FACED (after auto-deploy setup):**
```
Updates were rejected because the remote contains work that you do not have locally
```
**CAUSE:** GitHub Actions pushed a commit (updated deployment.yaml) that local doesn't have.

**FIX:** Always pull before push after GitHub Actions runs:
```bash
git pull origin main --rebase && git push origin main
```

---

# PHASE 5 — ARGOCD SETUP

## Step 22 — What is ArgoCD?

ArgoCD runs INSIDE your Kubernetes cluster and:
- Watches your GitHub repo's `k8s/` folder
- When `deployment.yaml` changes → automatically deploys to GKE
- Keeps cluster always matching what's in GitHub (GitOps)

**Why ArgoCD + GitHub Actions together:**
- GitHub Actions = builds Docker image (CI)
- ArgoCD = deploys to Kubernetes (CD)
- GitHub Actions updates deployment.yaml with new image tag
- ArgoCD sees the change and deploys automatically

---

## Step 23 — Install ArgoCD on GKE

```bash
# Create dedicated namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Verify pods are running (wait 2-3 minutes)
kubectl get pods -n argocd
```

Expected output — all should be Running:
```
argocd-application-controller-0      1/1   Running
argocd-dex-server-xxx                1/1   Running
argocd-notifications-controller-xxx  1/1   Running
argocd-redis-xxx                      1/1   Running
argocd-repo-server-xxx               1/1   Running
argocd-server-xxx                    1/1   Running
argocd-applicationset-controller-xxx 0/1   Error    ← IGNORE THIS
```

**NOTE:** The `argocd-applicationset-controller` shows Error — this is harmless. It's an advanced feature we don't use.

**ERROR FACED:**
```
The CustomResourceDefinition "applicationsets.argoproj.io" is invalid: metadata.annotations: Too long
```
**FIX:** Ignore this error. It only affects applicationset-controller which we don't need.

---

## Step 24 — Expose ArgoCD UI

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Wait 1-2 minutes for EXTERNAL-IP to appear
kubectl get svc argocd-server -n argocd
```

Wait until EXTERNAL-IP shows an IP (not `<pending>`).

ArgoCD UI will be at: `https://YOUR_ARGOCD_IP`

---

## Step 25 — Get ArgoCD Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Login credentials:
- Username: `admin`
- Password: output of above command

**NOTE:** Browser will show SSL warning — click Advanced → Proceed. This is normal (no SSL cert).

---

## Step 26 — Create ArgoCD Application

Create file `argocd-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gke-learning-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Vamsi-90/gke-learning-app
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply it:
```bash
kubectl apply -f argocd-app.yaml
```

**IMPORTANT:** Do NOT use heredoc (`kubectl apply -f - <<EOF`) for this. It causes YAML indentation errors.

**ERROR FACED:**
```
error parsing STDIN: error converting YAML to JSON: yaml: line 21: could not find expected ':'
```
**CAUSE:** Using heredoc adds leading spaces to YAML content.

**FIX:** Write to a file first, then apply:
```bash
# Write to file (using editor or Claude Write tool)
# Then apply:
kubectl apply -f argocd-app.yaml
```

---

## Step 27 — Fix ArgoCD Sync Issues

After applying, check status:
```bash
kubectl get application -n argocd
```

**ERROR FACED:**
```
SYNC STATUS: Unknown
Message: Failed to unmarshal "deployment.yaml": yaml: line 2: mapping values are not allowed in this context
```
**CAUSE:** deployment.yaml had leading spaces (created via heredoc).

**FIX:**
1. Rewrite the YAML files properly (no leading spaces)
2. Force ArgoCD to refresh cache:
```bash
kubectl annotate application gke-learning-app -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

---

# PHASE 6 — VERIFICATION & TESTING

## Step 28 — Verify Deployment

```bash
# Check ArgoCD status
kubectl get application -n argocd
# Should show: SYNC STATUS=Synced, HEALTH STATUS=Healthy

# Check pods running
kubectl get pods
# Should show 2 pods Running

# Check public IP
kubectl get svc gke-learning-app
# Copy EXTERNAL-IP and open in browser
```

---

## Step 29 — Test 24/7 Uptime (Kill a Pod)

```bash
# Get pod names
kubectl get pods

# Delete one pod
kubectl delete pod gke-learning-app-XXXXX-XXXXX

# Watch Kubernetes recreate it automatically
kubectl get pods -w
```

Pod will be recreated in seconds. Website never goes down.

---

## Step 30 — Test Failure Protection

```bash
# Break the app (add syntax error to app.py)
# Push to GitHub

# Force restart with broken image
kubectl rollout restart deployment/gke-learning-app

# Watch — new pod crashes, OLD pods keep running!
kubectl get pods -w
# New pod: CrashLoopBackOff (broken)
# Old pods: Running (still serving traffic!)

# Website stays UP throughout!
```

---

## Step 31 — Rollback

```bash
# Instantly go back to previous working version
kubectl rollout undo deployment/gke-learning-app

# View full deployment history
kubectl rollout history deployment/gke-learning-app
```

---

# COST MANAGEMENT

## Stop Cluster (Save Money)

```bash
# Scale nodes to 0 — stops VM billing, keeps config
gcloud container clusters resize gke-learning-cluster --node-pool default-pool --num-nodes 0 --zone us-central1-a
```

## Restart Cluster

```bash
# Scale back to 2 nodes
gcloud container clusters resize gke-learning-cluster --node-pool default-pool --num-nodes 2 --zone us-central1-a

# Reconnect kubectl
gcloud container clusters get-credentials gke-learning-cluster --zone=us-central1-a --project=gke-learning-2025

# Verify
kubectl get nodes
kubectl get pods
```

---

# COMPLETE ERRORS & FIXES REFERENCE

| Error | Cause | Fix |
|-------|-------|-----|
| `Project ID already in use` | Project name taken globally | Add numbers to project name |
| `unrecognized arguments: --zone` | Multiline command with `\` fails in zsh | Run as single line command |
| `gke-gcloud-auth-plugin not found` | Plugin not installed | `gcloud components install gke-gcloud-auth-plugin` |
| `Repository not found` | Wrong GitHub username | Check exact username and repo URL |
| `Permission denied to MrCOOL1732` | Multiple GitHub accounts | Use PAT token in remote URL |
| `refusing to allow PAT without workflow scope` | Token missing scope | Create new token with `repo` + `workflow` scopes checked |
| `could not read Username: terminal prompts disabled` | No auth for git push in Actions | Add `permissions: contents: write` + use `GITHUB_TOKEN` in remote URL |
| `Updates rejected, remote has work` | GitHub Actions pushed a commit | `git pull origin main --rebase && git push origin main` |
| `yaml: line 2: mapping values not allowed` | Leading spaces in YAML | Write files directly, never use indented heredoc |
| `Internal Server Error` on Flask | Dockerfile missing templates/static | Add `COPY templates/ templates/` and `COPY static/ static/` to Dockerfile |
| `Failed to unmarshal deployment.yaml` | YAML indentation wrong | Rewrite file + force ArgoCD refresh with annotation |
| ArgoCD SYNC STATUS: Unknown | Cached YAML error | `kubectl annotate application ... argocd.argoproj.io/refresh=hard --overwrite` |
| App not updating after push | Using `:latest` tag, ArgoCD sees no change | Set up GitHub Actions to auto-update deployment.yaml with commit SHA |

---

# COMMANDS CHEAT SHEET

## Connect to Cluster
```bash
gcloud container clusters get-credentials gke-learning-cluster --zone=us-central1-a --project=gke-learning-2025
```

## Cluster Status
```bash
kubectl get nodes                    # See VMs
kubectl get pods                     # See running app copies
kubectl get svc                      # See public IPs
kubectl get namespaces               # See all namespaces
kubectl get pods -n argocd           # See ArgoCD pods
kubectl get application -n argocd    # See ArgoCD app status
```

## Deployment
```bash
kubectl rollout restart deployment/gke-learning-app    # Pick up new image
kubectl rollout undo deployment/gke-learning-app       # Rollback
kubectl rollout history deployment/gke-learning-app    # View history
kubectl rollout status deployment/gke-learning-app     # Watch rollout progress
```

## Debugging
```bash
kubectl logs POD_NAME                                  # View app logs
kubectl describe pod POD_NAME                          # Pod details
kubectl describe application gke-learning-app -n argocd  # ArgoCD details
```

## ArgoCD
```bash
# Force refresh
kubectl annotate application gke-learning-app -n argocd argocd.argoproj.io/refresh=hard --overwrite

# Get ArgoCD password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Cost Control
```bash
# Stop nodes
gcloud container clusters resize gke-learning-cluster --node-pool default-pool --num-nodes 0 --zone us-central1-a

# Start nodes
gcloud container clusters resize gke-learning-cluster --node-pool default-pool --num-nodes 2 --zone us-central1-a
```

---

# HOW TO DEPLOY NEW CODE

```bash
# 1. Make changes to your code
# 2. Push to GitHub
git add .
git commit -m "your change description"
git pull origin main --rebase && git push origin main

# 3. GitHub Actions automatically:
#    - Builds new Docker image with commit SHA tag
#    - Pushes to Artifact Registry
#    - Updates k8s/deployment.yaml with new image tag
#    - Pushes deployment.yaml back to GitHub

# 4. ArgoCD automatically:
#    - Detects deployment.yaml changed
#    - Deploys new pods
#    - Keeps old pods until new ones are healthy

# 5. Website updates automatically — NO manual commands needed!
```

---

# IMPORTANT NOTES FOR NEXT CLAUDE INSTANCE

1. **Always run gcloud commands as single lines** — multiline with `\` breaks in zsh
2. **Never use heredoc for YAML files** — creates leading spaces, breaks YAML parsing
3. **Write YAML files directly** using Write tool, not via terminal commands
4. **Secret name is `AKHILA_GCP_SA_KEY`** — not `GCP_SA_KEY`
5. **After any GitHub Actions run**, do `git pull origin main --rebase` before pushing
6. **ArgoCD applicationset-controller error is harmless** — ignore it
7. **The `/health` route in Flask is critical** — Kubernetes uses it for health checks
8. **Dockerfile must copy templates/ and static/ folders** — not just app.py
9. **`permissions: contents: write`** must be in build.yaml for GitHub Actions to push back
10. **After scaling nodes back up**, always reconnect kubectl with `get-credentials` command
