# 📘 Django DevSecOps CI/CD Pipeline – Learning Notebook

## 📖 Preface

In modern software development, **security cannot be an afterthought**. Developers are responsible not only for writing clean, testable, and maintainable code but also for ensuring that the code and infrastructure are secure.

This notebook explains a **DevSecOps pipeline** for a Django application. The pipeline integrates **security, testing, and quality tools** at different stages of CI/CD.

---

## 📂 Table of Contents

1. **Introduction to DevSecOps**
2. **Tools Used in This Pipeline**

   * GitHub Actions
   * PostgreSQL (Service)
   * GitGuardian
   * Gitleaks
   * TruffleHog
   * Detect-Secrets
   * Django Unit Tests
   * SonarCloud
   * Snyk
   * Trivy
   * OWASP ZAP
   * GitHub Artifacts
3. **Pipeline Flow Explained**
4. **Case Study – Running the Pipeline**
5. **Summary**
6. **Further Reading & References**

---

## 1. 🔐 Introduction to DevSecOps

* **DevOps** = Development + Operations (fast delivery)
* **DevSecOps** = Development + Security + Operations (secure delivery)

The key principle is:
👉 *“Build security into every stage of the software development lifecycle.”*

Instead of testing and fixing security issues at the end, we integrate **static analysis, dependency scanning, secrets detection, and penetration testing** into CI/CD.

---

## 2. 🛠 Tools Used in the Pipeline

Let’s break down each tool used in your workflow.

---

### 2.1 GitHub Actions (CI/CD Orchestrator)

* **What is it?**
  GitHub Actions is a CI/CD service that automates workflows directly inside your GitHub repository.

* **Why is it used?**
  It runs tests, builds applications, deploys code, and integrates security scans automatically on **push** and **pull requests**.

* **Example:**

  ```yaml
  on:
    push:
      branches: ["main"]
  jobs:
    build:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4
        - name: Run script
          run: echo "Hello CI/CD!"
  ```

---

### 2.2 PostgreSQL (Database Service)

* **What is it?**
  PostgreSQL is an open-source relational database.

* **Why is it used?**
  Django applications typically use PostgreSQL in production. In CI/CD, we spin up a temporary DB container for running unit tests.

* **Pipeline Setup:**

  ```yaml
  services:
    postgres:
      image: postgres:14
      env:
        POSTGRES_USER: django
        POSTGRES_PASSWORD: django
        POSTGRES_DB: testdb
      ports:
        - 5432:5432
  ```

---

### 2.3 GitGuardian (Secrets Detection)

* **What is it?**
  GitGuardian scans repositories for **hardcoded secrets** (API keys, passwords, tokens).

* **Why is it used?**
  Prevents secrets from leaking into GitHub (a major cause of breaches).

* **Example Detection:**

  ```python
  AWS_SECRET = "AKIAxxxxxxxxxxxx"
  ```

  GitGuardian will flag this as a leaked AWS key.

---

### 2.4 Gitleaks (Secrets Detection)

* **What is it?**
  An open-source tool for detecting hardcoded secrets using regex + entropy checks.

* **Why is it used?**
  Acts as a second layer of protection, catching things GitGuardian might miss.

* **Pipeline Usage:**

  ```yaml
  - name: Gitleaks Scan
    uses: gitleaks/gitleaks-action@v2
  ```

---

### 2.5 TruffleHog (Secrets Detection)

* **What is it?**
  Scans git history, branches, and commits for **high-entropy strings** (potential secrets).

* **Why is it used?**
  Finds secrets hidden deep in history (even if they were deleted).

* **Example Command:**

  ```bash
  trufflehog github --repo github.com/org/repo
  ```

---

### 2.6 Detect-Secrets (Baseline Scanning)

* **What is it?**
  A tool by Yelp for **preventing new secrets** from being committed.

* **Why is it used?**
  It creates a `.secrets.baseline` file, marking which detected secrets are false positives. This way, developers won’t reintroduce new leaks.

* **Workflow:**

  1. Run `detect-secrets scan > .secrets.baseline`
  2. Audit with `detect-secrets audit .secrets.baseline`
  3. Commit baseline → future commits get checked.

---

### 2.7 Django Unit Tests

* **What is it?**
  Django’s built-in test framework for **functional and integration tests**.

* **Why is it used?**
  Ensures code correctness, migrations work, and DB logic functions properly.

* **Example:**

  ```python
  from django.test import TestCase

  class SimpleTest(TestCase):
      def test_addition(self):
          self.assertEqual(1 + 1, 2)
  ```

---

### 2.8 SonarCloud (SAST & Code Quality)

* **What is it?**
  A cloud-based service for **Static Application Security Testing (SAST)** and **code quality analysis**.

* **Why is it used?**
  Detects vulnerabilities, code smells, and bugs in Python code.

* **Example Findings:**

  * Hardcoded credentials
  * SQL injection risks
  * Duplicate code

---

### 2.9 Snyk (Dependency Scanning – SCA)

* **What is it?**
  A **Software Composition Analysis (SCA)** tool for finding known vulnerabilities in dependencies.

* **Why is it used?**
  Python dependencies (like `django`, `requests`) may have CVEs (security flaws).

* **Example:**

  * `Django==2.2.0` → Vulnerable to CVE-2019-6975
  * Snyk flags this and recommends upgrading.

---

### 2.10 Trivy (Container Scanning)

* **What is it?**
  A vulnerability scanner for Docker images.

* **Why is it used?**
  Detects vulnerabilities in **OS packages + Python libraries** inside your Docker image.

* **Example Findings:**

  * `openssl 1.1.1` has HIGH vulnerability
  * Suggest upgrade in Dockerfile

---

### 2.11 OWASP ZAP (Dynamic Application Security Testing – DAST)

* **What is it?**
  A tool for runtime **penetration testing** of web applications.

* **Why is it used?**
  Simulates real-world attacks (SQL Injection, XSS, etc.) against your running Django app.

* **Example Scan:**

  ```bash
  zap-baseline.py -t http://localhost:8035 -r report.html
  ```

  Produces a detailed report with vulnerabilities.

---

### 2.12 GitHub Artifacts (Report Storage)

* **What is it?**
  GitHub Action to **store scan reports and logs** as downloadable artifacts.

* **Why is it used?**
  Security reports (ZAP, Trivy, SonarCloud) are too long for console logs → saved for analysis.

* **Example:**

  ```yaml
  - name: Upload Reports
    uses: actions/upload-artifact@v4
    with:
      name: zap-scan-reports
      path: report_html.html
  ```

---

## 3. 🔄 Pipeline Flow

1. **Checkout code & set up Python**
2. **Spin up PostgreSQL container**
3. **Run secrets detection (GitGuardian, Gitleaks, TruffleHog, Detect-Secrets)**
4. **Run Django unit tests**
5. **Run static code analysis (SonarCloud)**
6. **Scan dependencies (Snyk)**
7. **Build Docker & scan (Trivy)**
8. **Run OWASP ZAP against live Django server**
9. **Upload reports as artifacts**

---

## 4. 📊 Case Study – Running the Pipeline

Imagine a Django project with:

* A leaked AWS key in `.env`
* Outdated Django version (`2.2.0`)
* SQL injection bug in a view

🔍 The pipeline would catch:

* GitGuardian/Gitleaks → AWS key
* Snyk → Django vulnerability
* SonarCloud → SQL injection warning
* ZAP → runtime injection attempt succeeds

---

## 5. 📌 Summary

* **Secrets Detection:** GitGuardian, Gitleaks, TruffleHog, Detect-Secrets
* **Testing:** Django Unit Tests
* **Static Analysis (SAST):** SonarCloud
* **Dependency Scan (SCA):** Snyk
* **Container Security:** Trivy
* **Dynamic Testing (DAST):** OWASP ZAP
* **Report Storage:** GitHub Artifacts

👉 This gives **end-to-end DevSecOps coverage** for Django apps.

---

## 6. 📚 Further Reading

* GitHub Actions Docs → [https://docs.github.com/actions](https://docs.github.com/actions)
* OWASP ZAP Guide → [https://www.zaproxy.org](https://www.zaproxy.org)
* SonarCloud → [https://sonarcloud.io](https://sonarcloud.io)
* Snyk → [https://snyk.io](https://snyk.io)
* Trivy → [https://aquasecurity.github.io/trivy](https://aquasecurity.github.io/trivy)

---
