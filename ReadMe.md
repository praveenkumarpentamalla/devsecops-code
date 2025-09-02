# DevSecOps CI — documentation & step-by-step setup for your workflow

Nice — you already have a very capable pipeline. Below is a single, practical document you can follow to **write, configure, run and troubleshoot** your GitHub Actions DevSecOps workflow. It lists **every account / token / secret** you’ll need, how to generate them, where to add them in GitHub, how to interpret results, common errors and fixes.

---

# 1 — What the workflow does (summary)

Your workflow runs on `push` / `pull_request` to `main` and does:

1. Checkout repo (full history required for secret scanners)
2. Install Python deps and run Django tests (Postgres service)
3. Static secret scans: **GitGuardian**, **Gitleaks**, **TruffleHog**, **detect-secrets**
4. SAST / quality: **SonarCloud**
5. SCA: **Snyk** (dependency scanning)
6. Container scanning: **Trivy** (scans built Docker image)
7. DAST: **OWASP ZAP** scans the running Django app and uploads reports as artifacts

This covers many layers (SAST, SCA, DAST, secrets, container security). Good!

---

# 2 — Files to include in your repo

1. `.github/workflows/devsecops.yml` — the workflow you posted (keep it in `.github/workflows/`)
2. `requirements.txt` — your python deps (you have it)
3. `Dockerfile` — used by the Docker build step (ensure it exists and works)
4. `sonar-project.properties` — (optional but recommended) for SonarCloud configuration

Example `sonar-project.properties` (repo root) — **replace placeholders**:

```properties
sonar.organization=YOUR_SONARCLOUD_ORG_KEY
sonar.projectKey=YOURORG_rxcapital
sonar.projectName=rxcapital
sonar.host.url=https://sonarcloud.io
sonar.sources=.
sonar.python.version=3.10
# sonar.python.coverage.reportPaths=coverage.xml   # if you upload coverage
```

---

# 3 — Secrets & Accounts you must create (exact names for GitHub Secrets)

Add these in GitHub: **Repo → Settings → Secrets and variables → Actions → New repository secret**

| Secret name                     |                                                   Purpose | Where to get                                          |
| ------------------------------- | --------------------------------------------------------: | ----------------------------------------------------- |
| `SONAR_TOKEN`                   |                                 SonarCloud analysis token | SonarCloud → My account → Security → Generate token   |
| `SNYK_TOKEN`                    |                                           Snyk auth token | Snyk → Account settings → Auth token                  |
| `GITGUARDIAN_API_KEY`           |                                    GitGuardian Auth token | GitGuardian → Account/Settings → API key (Auth token) |
| `DOCKERHUB_USERNAME` (optional) |          Docker Hub username — if you need to push images | Docker Hub account                                    |
| `DOCKERHUB_TOKEN` (optional)    |                       Docker Hub password or PAT for push | Docker Hub → Create access token                      |
| `GITHUB_TOKEN`                  | auto-provided by GitHub Actions — no need to add manually | (GitHub provides it automatically in runs)            |

**Notes:**

* If you don’t push images to DockerHub you *don’t* need Docker secrets for Trivy (Trivy can scan local image built on runner).
* Do **not** commit tokens in code — always use GitHub Secrets. Rotate tokens regularly.

---

# 4 — How to create tokens (quick steps)

### SonarCloud token

1. Log in to SonarCloud → top-right → **My account** → **Security** → **Generate Token**.
2. Copy token and add to GitHub as `SONAR_TOKEN`.
3. Make sure you imported your GitHub repo into SonarCloud (Organization → Add Project → From GitHub).

**Important:** In SonarCloud project settings set the **Main branch** to `main` (Administration → Branches & Pull requests) or re-import project.

### Snyk token

1. Log in to Snyk → Account → **Auth Token** (or API token) → copy.
2. Add to GitHub as `SNYK_TOKEN`.
3. Test locally: `snyk auth <TOKEN>` and `snyk test --file=requirements.txt --package-manager=pip`.

### GitGuardian API key

1. Log in to GitGuardian → Account → API Key / Auth Token (copy)
2. Add to GitHub as `GITGUARDIAN_API_KEY`.

### Docker Hub credentials (only if needed)

1. Docker Hub → Account Settings → New Access Token or use your password.
2. Add `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` as GitHub secrets.

---

# 5 — Update / verify `sonar-project.properties`

If you prefer the workflow to pass Sonar details implicitly, add `sonar-project.properties` to repo root (see sample above). Make sure:

* `sonar.organization` equals SonarCloud organization key (found in SonarCloud org settings).
* `sonar.projectKey` matches the project key SonarCloud created when you imported the repo.
* If you prefer to pass those via the action `args`, that is also possible.

---

# 6 — How to run the workflow (execute)

1. Commit your workflow to `.github/workflows/devsecops.yml`.
2. Push to `main`:

   ```bash
   git add .github/workflows/devsecops.yml sonar-project.properties
   git commit -m "Add DevSecOps CI workflow"
   git push origin main
   ```
3. On GitHub go to **Actions → Django DevSecOps CI → latest run**.
4. Expand steps — logs show results. Download artifacts (ZAP reports) from the right sidebar **Artifacts** section.

---

# 7 — Where to find results and what each tool reports

* **Unit tests** — in the *Run Django tests* step logs.
* **SonarCloud** — open SonarCloud dashboard for your project (bugs, vulnerabilities, code smells, duplications).
* **Snyk** — step logs will list vulnerable packages; also check Snyk dashboard if you have an account.
* **Trivy** — logs show vulnerable packages found in Docker image; you can make a JSON artifact with Trivy if you want.
* **OWASP ZAP** — HTML/MD/JSON reports are uploaded as artifacts (download and open `report_html.html`).
* **GitGuardian / Gitleaks / TruffleHog / detect-secrets** — logs show findings; GitGuardian may also provide a dashboard.

---

# 8 — Common issues & fixes (you already encountered some)

### Sonar: `Could not find a default branch`

* Cause: SonarCloud project exists but no default branch set or project binding missing.
* Fix: SonarCloud → Project → Administration → Branches & Pull Requests → set `main` as the main branch. If binding still shows `NONEXISTENT`, re-import project from GitHub (Organization → Add Project → From GitHub).

### Sonar: `You must define sonar.projectKey, sonar.organization`

* Cause: scanner needs project key/organization.
* Fix: add `sonar-project.properties` with `sonar.organization` and `sonar.projectKey`, or pass them as args to the Sonar action.

### Snyk: `Authentication error (SNYK-0005)`

* Cause: `SNYK_TOKEN` missing/invalid.
* Fix: create token in Snyk and add to GitHub Secrets as `SNYK_TOKEN`. Test locally with `snyk auth`.

### Snyk scanning npm instead of pip

* Cause: Snyk auto-detected npm.
* Fix: force Snyk to scan pip:

  ```yaml
  with:
    command: test
    args: --file=requirements.txt --package-manager=pip
  ```

### GitGuardian action warning: `Unexpected input(s) 'show_secrets'`

* Cause: the action doesn’t accept `show_secrets` param.
* Fix: remove that input and pass flags via `args` (or use the recommended usage from GitGuardian docs), e.g.:

  ```yaml
  with:
    args: "secret scan repo --exit-zero"
  env:
    GITGUARDIAN_API_KEY: ${{ secrets.GITGUARDIAN_API_KEY }}
  ```

### ZAP: `Resource not accessible by integration` (action tries to create issues)

* Cause: ZAP action tries to create GitHub issue using `GITHUB_TOKEN`, but token lacks permission or repo blocks it.
* Fix: set `allow_issue_writing: false` in the zap action (we did that). If you want issues created, grant `GITHUB_TOKEN` issues: write permission in repo settings / workflow permissions.

### ZAP: `Connection refused` / ZAP can't reach target

* Cause: Django server not started or not reachable on `localhost:8035`.
* Fix: start the Django server earlier in the workflow (we added `nohup python manage.py runserver 0.0.0.0:8035 &` and a wait loop). If using Docker container, publish the port and run container before ZAP.

---

# 9 — Useful local test commands

Before pushing, test locally:

* Run tests:

  ```bash
  python -m venv .venv
  source .venv/bin/activate
  pip install -r requirements.txt
  python manage.py migrate
  python manage.py test
  ```

* Build Docker image:

  ```bash
  docker build -t rxcapital:latest .
  ```

* Run Trivy locally:

  ```bash
  trivy image --severity HIGH,CRITICAL --format json -o trivy-report.json rxcapital:latest
  ```

* Run Snyk locally:

  ```bash
  snyk auth <TOKEN>
  snyk test --file=requirements.txt --package-manager=pip
  ```

* Run SonarScanner locally (if you have one installed):

  ```bash
  sonar-scanner \
    -Dsonar.projectKey=YOUR_PROJECT_KEY \
    -Dsonar.organization=YOUR_ORG \
    -Dsonar.host.url=https://sonarcloud.io \
    -Dsonar.login=YOUR_SONAR_TOKEN
  ```

* Run ZAP baseline locally (if you install ZAP CLI):

  ```bash
  zap-baseline.py -t http://localhost:8035 -r zap-report.html -a
  ```

---

# 10 — Security & policy recommendations

* Store **all tokens in GitHub Secrets** (never in repo).
* Limit tokens permissions (least privilege). Rotate regularly.
* Use branch protection rules to require the pipeline to pass before merging.
* Consider failing the pipeline on **HIGH/CRITICAL** vulnerabilities only (decide policy).
* Don’t expose `show_secrets` in logs in public repos. Use `--exit-zero` for monitoring mode, then change to fail mode after you triage.

---

# 11 — Optional improvements (performance & reliability)

* **Cache pip** (you already use `actions/setup-python@v5` with `cache: pip`).
* **Cache Docker layers** using buildx and registry cache if you push to a registry.
* Run non-dependent scans **in parallel jobs** to reduce total runtime (e.g., Snyk, Trivy, secret scans can be parallel jobs).
* Use **CodeQL** for another SAST perspective (GitHub builtin).
* Add a *policy job* that fails only on HIGH/CRITICAL issues.

---

# 12 — Checklist to verify before first run

* [ ] `requirements.txt` present and pip install works locally
* [ ] `Dockerfile` builds locally `docker build -t rxcapital:latest .`
* [ ] Add GitHub Secrets: `SONAR_TOKEN`, `SNYK_TOKEN`, `GITGUARDIAN_API_KEY` (and optionally Docker Hub creds)
* [ ] SonarCloud: project imported, `main` set as main branch, `sonar-project.properties` configured or pass args in action
* [ ] Push `.github/workflows/devsecops.yml` and watch **Actions** UI
* [ ] Download ZAP artifacts and review `report_html.html` for warnings/alerts
* [ ] Fix findings incrementally (start with High/Critical)

---

# 13 — Short troubleshooting quick commands (when something fails)

* Sonar `default branch` error → open SonarCloud UI and set main branch or re-import.
* Snyk 401 → re-add `SNYK_TOKEN` to secrets and re-run.
* GitGuardian `Invalid API key` → re-add `GITGUARDIAN_API_KEY` secret.
* ZAP `connection refused` → check `django-server.log` artifact for errors, ensure `python manage.py runserver` started and bound to `0.0.0.0:8035`.

---

If you want, I can:

* Produce a **one-page checklist** you can keep in your repo README.
* Update your **SonarCloud configuration file** if you paste your Sonar org key and project key.
* Convert some steps into **parallel jobs** to shorten run time.

Which of those would help you next?
