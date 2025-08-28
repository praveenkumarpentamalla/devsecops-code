Perfect 👍 thanks for sharing Dockerfile and docker-compose. Let’s set this up in **GitHub Actions** so that every push/PR to `main` runs a **full DevSecOps pipeline** with:

* **SonarQube / SonarCloud (SAST + code quality)**
* **Snyk (Dependency scanning)**
* **OWASP ZAP (DAST)**
* **Trivy (Container & IaC scanning)**

---

## 🔹 Step 1: Choose SonarCloud or SonarQube

* If you want **self-hosted SonarQube**, you need to run a SonarQube server inside your infra. That means hosting it and maintaining DB, storage, and upgrades.
* **SonarCloud (recommended for GitHub projects):**

  * Free for open-source repos.
  * Fully managed, integrates directly with GitHub Actions.
  * No extra hosting needed.

✅ For your use case → **Use SonarCloud** unless you already run SonarQube internally.

---

## 🔹 Step 2: Add GitHub Secrets

Go to **Repo → Settings → Secrets and variables → Actions** and add:

| Secret Name           | Value                                                             |
| --------------------- | ----------------------------------------------------------------- |
| `SONAR_TOKEN`         | Token from \[SonarCloud → My Account → Security → Generate Token] |
| `SNYK_TOKEN`          | Token from \[Snyk → Account Settings → API Token]                 |
| `DOCKER_HUB_USERNAME` | Your DockerHub username (for Trivy build push)                    |
| `DOCKER_HUB_TOKEN`    | DockerHub PAT/password                                            |
| `ZAP_AUTH`            | (Optional) Auth script/API key if your Laravel app requires login |

---

## 🔹 Step 3: GitHub Actions Workflow

Create file: `.github/workflows/devsecops.yml`

```yaml
name: DevSecOps CI/CD

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-test-scan:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: laravel
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping -h 127.0.0.1 -uroot -proot"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      # Checkout
      - uses: actions/checkout@v3

      # Setup PHP
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: mbstring, bcmath, pdo_mysql, xml, gd
          tools: composer

      # Install Laravel deps
      - name: Install PHP dependencies
        run: composer install --no-interaction --prefer-dist

      # Run Laravel tests
      - name: Run Laravel tests
        run: php artisan test

      # Setup Node
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'

      # Install Node deps (WebSocket server)
      - name: Install Node dependencies
        working-directory: ./socket-server
        run: npm ci

      # Run Node tests (if available)
      - name: Run Node tests
        working-directory: ./socket-server
        run: npm test --if-present

      # ---- Security Scans ----
      # SonarCloud (SAST)
      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@master
        with:
          projectBaseDir: .
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # Snyk (SCA)
      - name: Snyk Open Source Scan
        uses: snyk/actions/node@master
        with:
          args: test --all-projects
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      # Trivy (Container scan)
      - name: Build Docker image
        run: docker build -t myapp:latest .

      - name: Run Trivy Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:latest
          format: table
          exit-code: 0   # change to 1 if you want pipeline to fail on vulns
          severity: HIGH,CRITICAL

      # OWASP ZAP (DAST)
      - name: Run OWASP ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.10.0
        with:
          target: 'http://localhost:8000'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'
```

---

## 🔹 Step 4: SonarCloud Project Setup

1. Go to [SonarCloud](https://sonarcloud.io/).
2. Import your GitHub repo.
3. Copy the **project key** and add to `sonar-project.properties` in repo root:

```properties
sonar.organization=your-org
sonar.projectKey=your-project-key
sonar.php.coverage.reportPaths=coverage.xml
sonar.sources=.
sonar.host.url=https://sonarcloud.io
```

---

## 🔹 Step 5: Verify & Extend

* Add **Gitleaks** or **GitGuardian** for **Secrets detection** (you missed this, I recommend adding it).
* Add **Business Logic/Access Control tests** → this is usually **manual** or requires custom test cases, not automated.
* Add **CodeQL** (GitHub’s native SAST) as an optional extra.

---

✅ With this pipeline:

* Laravel + Node build & test
* SonarCloud → SAST
* Snyk → Dependency scanning
* Trivy → Container scan
* OWASP ZAP → DAST
* Redis + MySQL spin up for tests

---

Do you want me to **add Gitleaks & CodeQL** into this GitHub Action too so you’ll have *zero major security gaps* in the pipeline?
