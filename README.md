# Student Registration System

A simple **Flask** web application to manage student records with **MongoDB** as the backend database, wired up with a dual **Jenkins** + **GitHub Actions** CI/CD setup for automated testing, staging, and production deployment.

---

## Tech Stack & Features

* **Backend:** Python, Flask
* **Database:** MongoDB Atlas (via `Flask-PyMongo`, TLS via `certifi`)
* **Frontend:** HTML, Jinja2 templates, Bootstrap 5
* **Environment Variables:** Managed via a `.env` file
* **CI/CD:** Jenkins Pipeline + GitHub Actions (staging/production deploys), pytest, Docker

**App features:**
* List all students on the home page
* Add a new student
* Update existing student details
* Delete a student
* Simple, responsive UI using Bootstrap

---

## Local Setup

**1. Clone the repository**
```bash
git clone <your-repo-url>
cd herovired_cicd_flask_Practice
```

**2. Create and activate a virtual environment**
```bash
python -m venv venv
# Windows:
venv\Scripts\activate
# Linux / Mac:
source venv/bin/activate
```

**3. Install dependencies**
```bash
pip install -r requirements.txt
```

**4. Configure environment variables** — create a `.env` file in the project root:
```env
MONGO_URI=<your-mongodb-connection-string>
SECRET_KEY=<your-secret-key>
```

**5. Run the application**
```bash
python app.py
```
Open your browser at [http://localhost:5000](http://localhost:5000).

---

## Project Structure

```
herovired_cicd_flask_Practice/
│
├── .github/
│   └── workflows/
│       ├── ci-cd.yml               # Build/test + staging/production deploy
│       ├── securegate.yml          # Security gate (Gitleaks/Semgrep/Trivy/SonarQube)
│       └── securegate-summary.yaml # Scheduled SecureGate summary notifications
│
├── templates/
│   ├── base.html
│   ├── index.html
│   ├── add_student.html
│   └── update_student.html
│
├── Screenshots/                    # CI/CD setup & pipeline run screenshots
│
├── app.py                          # Flask routes + MongoDB access
├── test_app.py                     # pytest suite
├── Jenkinsfile                     # Jenkins Build → Test → Deploy pipeline
├── azure-pipelines.yml             # Azure Pipelines CI
├── requirements.txt
├── start_flask.sh
└── README.md
```

---

## Application Screenshots

**Home Page** — lists all students with Edit/Delete buttons.
<img width="900" alt="Home page" src="https://github.com/user-attachments/assets/a58a6a6d-4978-4769-8074-232e4d31e69d" />

**Add Student** — form to add a new student.
<img width="900" alt="Add student form" src="https://github.com/user-attachments/assets/d65d25c3-ebb5-410a-adb1-e130ad7c5878" />

**Update Student** — form pre-filled with student details.
<img width="900" alt="Update student form" src="https://github.com/user-attachments/assets/04febf01-879f-431f-ab07-abcfb993acf1" />

---

## CI/CD Pipelines

This repository fulfils **two separate assignment briefs**, one per tool, kept as independent sections below: **Part 1 — Jenkins**, and **Part 2 — GitHub Actions**. A shared prerequisite for both:

- A MongoDB Atlas cluster (or any reachable MongoDB instance) with a database user, and a **Network Access** rule allowing the deploying server's IP (or `0.0.0.0/0` for a lab setup).
- A Linux deploy target — this project uses a single AWS EC2 Ubuntu instance that hosts **both** Jenkins and the running Flask app — with inbound security group rules open for ports `22` (SSH), `8080` (Jenkins UI), `5000` (staging Flask), `5001` (production Flask).

---

## Part 1: Jenkins CI/CD Pipeline for Flask Application

### 1. Setup

Jenkins was installed on a self-provisioned AWS EC2 Ubuntu VM (rather than a managed cloud Jenkins service), and configured with Python, Docker, and Java 21.

1. Launched an AWS EC2 instance (Ubuntu, `t2.micro`) and opened security group ports `22`, `8080`, `5000`, `5001` to `0.0.0.0/0` — both browser-based EC2 Instance Connect and inbound GitHub webhook calls originate from shifting AWS/GitHub IP ranges, not a fixed "My IP".
<img width="900" alt="EC2 security group inbound rules" src="Screenshots/Screenshot 2026-07-23 000650.png" />
<img width="900" alt="EC2 instance running" src="Screenshots/Screenshot 2026-07-23 001015.png" />
<img width="900" alt="EC2 instance security tab" src="Screenshots/Screenshot 2026-07-23 001043.png" />

2. Installed Java 21, Docker, and Python; added Jenkins' apt repo using its current signing key (Jenkins periodically rotates this key, so always pull the current URL from Jenkins' own install docs):
```bash
sudo apt update && sudo apt install -y openjdk-21-jre python3-venv python3-pip docker.io

sudo mkdir -p /etc/apt/keyrings

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list

sudo apt update && sudo apt install -y jenkins

sudo systemctl enable --now jenkins
```

3. Added the `jenkins` system user to the `docker` group and restarted Jenkins, so the pipeline's `Test` stage can run containers:
```bash
sudo usermod -aG docker jenkins && sudo systemctl restart jenkins
```

4. Unlocked Jenkins with the generated initial admin password and installed the suggested plugins plus the **Email Extension Plugin**.
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
<img width="900" alt="Unlock Jenkins screen" src="Screenshots/Screenshot 2026-07-23 004555.png" />

### 2. Source Code

This repository is used directly as the Jenkins pipeline's source — the Pipeline job (see step 3 below) pulls it straight from GitHub via Git SCM, rather than a manual `git clone` on the server.
<img width="900" alt="Jenkins Pipeline job SCM configuration" src="Screenshots/Screenshot 2026-07-23 012651.png" />

### 3. Jenkins Pipeline (`Jenkinsfile`)

A `Jenkinsfile` at the repository root defines three stages:
1. **Build** — creates a `venv` and runs `pip install -r requirements.txt`.
2. **Test** — starts a throwaway `mongo:7` Docker container, runs `pytest test_app.py` against it, then tears the container down.
3. **Deploy** — pulls `MONGO_URI`/`SECRET_KEY` from the Jenkins credential store, writes them to `.env`, starts the app in the background on the staging environment, and health-checks it with `curl`.

**Jenkins credentials required** (Manage Jenkins → Credentials):

| Credential ID | Type | Value |
|---|---|---|
| `mongo-uri` | Secret text | MongoDB Atlas connection string (with database name) |
| `flask-secret-key` | Secret text | Flask `SECRET_KEY` |
| SMTP auth (see Notifications below) | Username/password | Gmail address + App Password |

<img width="900" alt="Jenkins mongo-uri credential" src="Screenshots/Screenshot 2026-07-23 005540.png" />
<img width="900" alt="Jenkins flask-secret-key credential" src="Screenshots/Screenshot 2026-07-23 005641.png" />

The Pipeline job itself was created via *Pipeline script from SCM* → Git → repo URL → branch `*/main` → script path `Jenkinsfile`, then run once manually so Jenkins registers the `githubPush()` trigger declared in the file (see Triggers below).

A completed run showing the Build → Test → Deploy stages passing end to end:
<img width="900" alt="Jenkins job status showing successful build with no test failures" src="Screenshots/Screenshot 2026-07-23 214840.png" />
<img width="900" alt="Jenkins build #9 detail page showing git revision and passing tests" src="Screenshots/Screenshot 2026-07-23 214857.png" />
<img width="900" alt="Jenkins console output showing the pipeline finishing with SUCCESS" src="Screenshots/Screenshot 2026-07-23 214912.png" />

### 4. Triggers

The `Jenkinsfile` declares `triggers { githubPush() }`, paired with a GitHub webhook — every push to `main` starts a new build automatically. The Pipeline job also has "GitHub hook trigger for GITScm polling" checked.

- **Webhook**: payload URL `http://<ec2-ip>:8080/github-webhook/`, content type `application/json`, "Just the push event".
<img width="900" alt="GitHub webhook configuration" src="Screenshots/Screenshot 2026-07-23 013041.png" />
<img width="900" alt="GitHub webhook recent delivery success" src="Screenshots/Screenshot 2026-07-23 013343.png" />

### 5. Notifications

A `post { success / failure }` block in the `Jenkinsfile` emails the build result either way via the **Email Extension plugin**, attaching `flask_app.log` on failure. SMTP was configured under **both** "E-mail Notification" and "Extended E-mail Notification" (Manage Jenkins → System) using Gmail (`smtp.gmail.com:465`, SSL, an App Password generated after enabling 2-Step Verification — a normal Gmail password is rejected with `530 5.7.0 Authentication Required`).
<img width="900" alt="Jenkins Extended E-mail Notification SMTP config" src="Screenshots/Screenshot 2026-07-23 010854.png" />
<img width="900" alt="Jenkins Extended E-mail Notification saved" src="Screenshots/Screenshot 2026-07-23 011833.png" />

Success notification email received after a passing build:
<img width="900" alt="Received SUCCESS email for flask-cicd build" src="Screenshots/Screenshot 2026-07-23 214956.png" />

---

## Part 2: GitHub Actions CI/CD Pipeline Flask App

### 1. Setup

This repository has both a `main` branch and a `staging` branch, created specifically so the workflow can distinguish staging vs. production deploys by branch/tag.

### 2. GitHub Actions Workflow

A `.github/workflows/ci-cd.yml` file defines the workflow, triggered on push to `main`/`staging` and on `v*` tags.

### 3. Workflow Steps

Three jobs:
- **`build-and-test`** — installs dependencies and runs the test suite. Spins up a `mongo:7` service container and runs `pytest test_app.py` against it directly — no secrets or mocking required.
- **`deploy-staging`** — the "prepare for deployment" + "deploy to staging" steps combined. Runs only on push to `staging`, after `build-and-test` passes. SSHes into the EC2 box, pulls the `staging` branch, writes `.env` from secrets, (re)starts the app on port `5000`, and health-checks it — dumping `flask_app_staging.log` if the check fails.
- **`deploy-production`** — runs only when a tag matching `v*` is pushed, after `build-and-test` passes. SSHes into the same box, checks out the tag, and (re)starts the app on port `5001`.

Pushed to `staging` → `build-and-test` and `deploy-staging` succeeded, app responded at `http://<ec2-ip>:5000`:
<img width="900" alt="GitHub Actions staging deploy success" src="Screenshots/Screenshot 2026-07-23 023007.png" />
<img width="900" alt="Flask app running on staging port 5000" src="Screenshots/Screenshot 2026-07-23 023251.png" />

Tagged a release (`git tag v1.0.0 && git push origin v1.0.0`) → `deploy-production` succeeded, app responded at `http://<ec2-ip>:5001`:
<img width="900" alt="GitHub Release v1.0.0" src="Screenshots/Screenshot 2026-07-23 023723.png" />
<img width="900" alt="GitHub Actions production deploy success" src="Screenshots/Screenshot 2026-07-23 024542.png" />
<img width="900" alt="Flask app running on production port 5001" src="Screenshots/Screenshot 2026-07-23 024644.png" />

### 4. Environment Secrets

**Required GitHub Secrets** (Settings → Secrets and variables → Actions):

| Secret | Purpose |
|---|---|
| `MONGO_URI` | Atlas connection string (must include a database name, e.g. `/student_db`, before the `?`) |
| `SECRET_KEY` | Flask session secret |
| `SSH_PRIVATE_KEY` | Private key matching the EC2 instance's key pair |
| `SERVER_HOST` | EC2 public IP/hostname |
| `SERVER_USER` | SSH username (`ubuntu`) |

<img width="900" alt="GitHub repository secrets" src="Screenshots/Screenshot 2026-07-23 013819.png" />

---

### Troubleshooting notes from setup

*(applies to both pipelines above)*


- **EC2 "Error establishing SSH connection"** — usually a security group source set to "My IP" gone stale; widen it to `0.0.0.0/0` or refresh it to your current IP.
- **Jenkins apt `NO_PUBKEY` error** — Jenkins rotates its signing key periodically; always pull the current key URL from Jenkins' own install docs instead of reusing an old one.
- **`pymongo.errors.ServerSelectionTimeoutError: SSL handshake failed`** against a plain local/CI MongoDB — caused by unconditionally passing `tlsCAFile` to `PyMongo()`. TLS is now only forced for `mongodb+srv://` (Atlas) URIs, not plain local ones (see `app.py`).
- **Gmail SMTP `530 5.7.0 Authentication Required`** — requires a Google App Password (needs 2-Step Verification enabled first); a normal account password is rejected.
- **`pymongo.errors.OperationFailure: bad auth`** — a stale/incorrect database user password; rotate it in Atlas → Database Access and update it everywhere it's referenced (`.env`, the GitHub `MONGO_URI` secret, and the Jenkins `mongo-uri` credential).
- **Lost Jenkins admin access** — stop Jenkins, temporarily set `<useSecurity>false</useSecurity>` in `/var/lib/jenkins/config.xml`, restart, reset the password under Manage Jenkins → Users, then re-enable security from Manage Jenkins → Security.

---

## Notes

* Delete action removes a student immediately (no confirmation step at the route level).
* Uses `ObjectId` from `bson` to work with MongoDB document IDs.
* MongoDB Atlas TLS verification uses the `certifi` CA bundle explicitly, applied only for `mongodb+srv://` URIs — see the Troubleshooting section above.

---

## License

MIT License
