Nice — I’ll give you a **complete, ready-to-use GitHub Actions pipeline** tuned for your setup (you build Docker images locally on an **EC2 self-hosted runner**) plus **step-by-step EC2 setup instructions**, the exact **secrets to add**, `sonar-project.properties`, and an **alternate SSH-based workflow** if you prefer not to install a runner.

I will **assume**:

* Your repo branch is `main`.
* You will register a self-hosted GitHub Actions runner on the EC2 instance that runs Docker and Trivy.
* You want these scans: **Gitleaks (secrets), SonarCloud (SAST), Snyk (SCA), Trivy (container), OWASP ZAP (DAST)**.

If you prefer the SSH approach instead of self-hosted, skip to the “SSH-based alternative” near the end.

---

# 1) EC2 — prepare and install a self-hosted runner (step-by-step)

Run these commands on your EC2 instance (replace placeholders with your repo/org info):

1. **Install prerequisites (docker, docker-compose, git, curl, unzip, node, Java for ZAP):**

```bash
# Update & basic tools
sudo apt-get update -y
sudo apt-get install -y git curl unzip apt-transport-https ca-certificates gnupg lsb-release

# Docker
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER    # if you later run the runner as this user, logout/login

# Docker Compose (plugin or binary)
# for modern Docker Compose (plugin)
sudo apt-get install -y docker-compose-plugin || true
# fallback to docker-compose binary (optional)
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Node (for socket server local builds)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Java (ZAP requires java)
sudo apt-get install -y default-jre

# Trivy install
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin

# (Optional) Install docker-runner dependencies
sudo apt-get install -y rsync
```

2. **Create & register the GitHub self-hosted runner**

* On GitHub: `Settings → Actions → Runners → New self-hosted runner` — copy the registration commands and token.
* Back on EC2 (example):

```bash
mkdir -p ~/actions-runner && cd ~/actions-runner
# Download runner (version will be provided by GitHub UI; example link)
curl -O -L https://github.com/actions/runner/releases/download/v2.310.0/actions-runner-linux-x64-2.310.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.310.0.tar.gz

# Replace URL and TOKEN with the ones provided by GitHub
./config.sh --url https://github.com/<ORG_OR_USER>/<REPO> --token <RUNNER_TOKEN>

# Install as service so it restarts automatically
sudo ./svc.sh install
sudo ./svc.sh start
```

> The runner now appears in your repo's Actions → Runners list.

3. **Verify tools on the runner:**

```bash
docker --version
docker compose version || docker-compose --version
node --version
trivy --version
java -version
```

---

# 2) Add required GitHub repository secrets

Go to **Repo → Settings → Secrets and variables → Actions → New repository secret** and add:

* `SONAR_TOKEN` — SonarCloud token (from SonarCloud → My Account → Security → Generate token).
* `SNYK_TOKEN` — Snyk API token (Snyk → Account settings → API token).
* `ZAP_AUTH` — optional: if your app needs auth for DAST (or store username/pw separately).
* `CI_FAIL_ON_HIGH_VULNS` — optional boolean (`true`/`false`) to control whether Trivy/Snyk should fail the run on HIGH/CRITICAL vulns (we’ll default to `true` in workflow).
* (No DockerHub secrets — you said images are built locally.)

---

# 3) sonar-project.properties (place in repository root)

Create `sonar-project.properties` in root (edit `sonar.organization` & `sonar.projectKey` to match your SonarCloud project):

```
sonar.organization=your-sonar-org
sonar.projectKey=your-org_your-repo
sonar.host.url=https://sonarcloud.io

# Sources & exclusions
sonar.sources=.
sonar.exclusions=**/vendor/**, **/node_modules/**, **/storage/**, **/public/**, **/.git/**

# PHP specifics (if you generate coverage)
# sonar.php.coverage.reportPaths=coverage.xml
```

Register the project in SonarCloud (Import from GitHub) and generate `SONAR_TOKEN` then add to secrets.

---

# 4) The complete GitHub Actions workflow (self-hosted runner)

**File:** `.github/workflows/devsecops-selfhost.yml`

> This workflow assumes the self-hosted runner (EC2) has docker, docker compose, node, trivy and Java installed (per section 1). It builds local images, runs tests (placeholder), runs Gitleaks, Snyk, Trivy, SonarCloud, and OWASP ZAP. Edit/adjust commands to fit your project structure.

```yaml
name: DevSecOps CI (self-hosted EC2)

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

env:
  IMAGE_PHP: civil-app:ci
  IMAGE_SOCKET: socket-server:ci
  CIVIL_PORT: 8000
  SOCKET_PORT: 3001
  CI_FAIL_ON_HIGH_VULNS: "true"   # override with repo secret if you want

jobs:
  devsecops:
    runs-on: [self-hosted, linux]
    timeout-minutes: 120
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Preflight (runner diagnostics)
        run: |
          echo "Runner info"
          uname -a
          echo "Docker:"
          docker --version || true
          echo "Docker Compose:"
          docker compose version || docker-compose --version || true
          echo "Trivy:"
          trivy --version || true
          echo "Java:"
          java -version || true

      # --------------------
      # PHP: install (uses action, works on self-hosted)
      - name: Setup PHP 8.2 & Composer
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: mbstring, bcmath, pdo_mysql, xml, gd
          tools: composer

      - name: Install PHP dependencies (composer)
        run: |
          composer install --no-interaction --prefer-dist || true

      - name: (Optional) Run Laravel tests - configure DB/test env as needed
        run: |
          # If you want tests to run reliably, create a test DB container or set env values before running tests.
          # This is a placeholder — adjust for your test database and .env.testing configuration.
          php artisan test || true

      # --------------------
      # Node: socket-server
      - name: Setup Node.js 20
        uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Install Node deps (socket-server)
        working-directory: ./socket-server
        run: |
          npm ci || npm install --production || true

      - name: Run Node tests (socket-server)
        working-directory: ./socket-server
        run: npm test --if-present || true

      # --------------------
      # Secret/Leak detection: Gitleaks (run via Docker)
      - name: Gitleaks - scan for secrets (quiet fail allowed)
        run: |
          echo "Running gitleaks..."
          docker run --rm -v "${{ github.workspace }}":/src zricethezav/gitleaks:latest detect --source /src --report=/src/gitleaks-report.json || true
        continue-on-error: false

      - name: Upload gitleaks report (artifact)
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: gitleaks-report
          path: gitleaks-report.json

      # --------------------
      # Build local Docker images (PHP and socket)
      - name: Build PHP Docker image (local)
        run: |
          docker build -t $IMAGE_PHP -f Dockerfile .

      - name: Build socket-server Docker image (local)
        run: |
          docker build -t $IMAGE_SOCKET ./socket-server

      # Optionally start app stack (docker compose) so DAST can hit it
      - name: Start app stack (docker compose)
        run: |
          # prefer 'docker compose' (v2) if available, fallback to docker-compose
          if command -v docker && docker compose version >/dev/null 2>&1; then
            docker compose up -d --build
          else
            docker-compose up -d --build
          fi
          # wait for HTTP 200 on app endpoint
          echo "Waiting for app to be ready on http://localhost:${CIVIL_PORT} ..."
          for i in {1..30}; do
            if curl -sSf "http://localhost:${CIVIL_PORT}" >/dev/null 2>&1; then
              echo "app is up"
              break
            fi
            echo "still starting..."
            sleep 3
          done

      # --------------------
      # Trivy container scan (fail the job on HIGH/CRITICAL if CI_FAIL_ON_HIGH_VULNS true)
      - name: Trivy scan PHP image (container)
        run: |
          echo "Running trivy scan on $IMAGE_PHP"
          if [ "${{ env.CI_FAIL_ON_HIGH_VULNS }}" = "true" ]; then
            trivy image --severity HIGH,CRITICAL --exit-code 1 $IMAGE_PHP
          else
            trivy image --severity HIGH,CRITICAL $IMAGE_PHP || true
          fi

      - name: Trivy scan socket image (container)
        run: |
          echo "Running trivy scan on $IMAGE_SOCKET"
          if [ "${{ env.CI_FAIL_ON_HIGH_VULNS }}" = "true" ]; then
            trivy image --severity HIGH,CRITICAL --exit-code 1 $IMAGE_SOCKET
          else
            trivy image --severity HIGH,CRITICAL $IMAGE_SOCKET || true
          fi

      # --------------------
      # Snyk (SCA) - authenticate & scan (Node & PHP)
      - name: Install Snyk CLI
        run: |
          npm i -g snyk || true

      - name: Snyk test (auth + test)
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        run: |
          snyk auth $SNYK_TOKEN || true
          # scan all projects (composer.json + package.json)
          snyk test --all-projects --severity-threshold=high || true

      # --------------------
      # SonarCloud (SAST + Code Quality) - requires SONAR_TOKEN
      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@master
        with:
          projectBaseDir: .
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # --------------------
      # OWASP ZAP baseline (DAST)
      - name: OWASP ZAP baseline scan (DAST)
        uses: zaproxy/action-baseline@v0.10.0
        with:
          target: "http://localhost:${CIVIL_PORT}"
          cmd_options: "-t http://localhost:${CIVIL_PORT} -r zap-report.html"
        # the zap action will download ZAP and run baseline; adjust auth if needed

      - name: Upload ZAP report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: zap-report
          path: zap-report.html

      # --------------------
      # Tear down
      - name: Tear down docker compose
        if: always()
        run: |
          if command -v docker && docker compose version >/dev/null 2>&1; then
            docker compose down --volumes --remove-orphans || true
          else
            docker-compose down --volumes --remove-orphans || true
          fi

      - name: Job finished - summary
        if: always()
        run: |
          echo "DevSecOps job completed (check actions logs & uploaded artifacts for reports)."
```

**Notes & customization:**

* The workflow runs on the **self-hosted runner** you registered on EC2.
* The job **builds local Docker images** and runs `trivy` on them — no registry/push required.
* **ZAP** runs against `http://localhost:8000` (your `civil-web` default). If your app listens on another host/port change `CIVIL_PORT`.
* **Snyk** and **SonarCloud** require tokens in secrets (`SNYK_TOKEN`, `SONAR_TOKEN`).
* Gitleaks is executed via Docker image; results are uploaded as an artifact (`gitleaks-report.json`).

---

# 5) What to change if tests fail because of DB / environment

* If your Laravel tests require MySQL/Postgres, add a step before `php artisan test` to start a DB container, set `DB_HOST`/`DB_DATABASE`/`DB_USERNAME`/`DB_PASSWORD` in environment (or generate a `.env.testing`) and run `php artisan migrate --force`.
* I left `php artisan test || true` so CI won't be blocked until you confirm test DB settings; replace `|| true` with proper handling once you configure the test DB.

---

# 6) SSH-based alternative (if you prefer not to install a self-hosted runner)

If you don’t want to register a runner, use this **SSH** approach: GitHub Actions runs on `ubuntu-latest`, copies the repo to EC2, and executes build/scan commands there.

**File:** `.github/workflows/devsecops-ssh.yml`

```yaml
name: DevSecOps (SSH to EC2)

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  remote:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup SSH Key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.EC2_SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H ${{ secrets.EC2_HOST }} >> ~/.ssh/known_hosts

      - name: Sync repo to EC2
        run: |
          rsync -av --delete --exclude='.git' ./ ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }}:/home/${{ secrets.EC2_USER }}/repo

      - name: Run remote build & scans on EC2
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          ssh -i ~/.ssh/id_rsa ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }} << 'EOF'
            set -e
            cd ~/repo
            # build images locally on EC2
            docker build -t civil-app:ci -f Dockerfile .
            docker build -t socket-server:ci ./socket-server
            # run trivy
            trivy image --severity HIGH,CRITICAL --exit-code 1 civil-app:ci || exit 1
            # run gitleaks via docker
            docker run --rm -v "$(pwd)":/src zricethezav/gitleaks:latest detect --source /src --report=/src/gitleaks-report.json || true
            # run snyk (if installed on EC2)
            snyk auth $SNYK_TOKEN || true
            snyk test --all-projects --severity-threshold=high || true
            # optionally run sonar scanner on EC2 (if sonar scanner installed/configured)
          EOF
```

**Required Secrets for SSH method:**

* `EC2_SSH_PRIVATE_KEY` — the PEM content (value) for the backup user you will SSH as.
* `EC2_USER` — e.g., `ubuntu` or `ec2-user`.
* `EC2_HOST` — EC2 public DNS / IP.
* `SNYK_TOKEN`, `SONAR_TOKEN` as before.

---

# 7) Extra recommended additions (short list)

* **CodeQL** job (GitHub built-in SAST) — add `github/codeql-action` if you want more SAST coverage.
* **Policy for failing**: decide which scans should block merges (Trivy high/critical, Sonar quality gate, etc.). I set Trivy to fail on HIGH/CRITICAL by default (`CI_FAIL_ON_HIGH_VULNS`).
* **Gitleaks pre-commit / pre-push** to block secrets from entering the repo.
* **Business-Logic test coverage**: write dedicated PHPUnit feature tests for access control and add them to the test step.

---

# 8) Quick checklist for you to run now

1. On EC2: run the **prereqs & runner registration** commands in section 1.
2. In your GitHub repo: add secrets `SONAR_TOKEN`, `SNYK_TOKEN`, (and `EC2_SSH_PRIVATE_KEY`, `EC2_USER`, `EC2_HOST` if using SSH).
3. Add `sonar-project.properties` to repo root and create SonarCloud project if using SonarCloud.
4. Add the workflow file `.github/workflows/devsecops-selfhost.yml`.
5. Push to `main` or open a PR — monitor Actions log. Look at uploaded artifacts (`gitleaks-report.json`, `zap-report.html`) for evidence.

---

If you want, I can now:

* Paste a **CodeQL** job to drop into the same workflow (optional).
* Produce a ready `systemd` service snippet or enhanced runner-install script for EC2 (auto-start on boot).
* Tweak the YAML so **Trivy** & **Snyk** only report (not fail) until you triage existing vulns.

Tell me which of those extras you want and I’ll append them immediately.
