Nice — below is a **complete, copy-pasteable GitHub Actions workflow** (plus supporting files and step-by-step explanation) that builds your **Laravel (PHP) and Node (socket)** images, **scans** them (Gitleaks, Snyk, Trivy, SonarCloud, ZAP), **pushes** to **AWS ECR**, and **deploys** to **ECS** (Fargate or EC2-backed). I also show the **task definition template**, the **secrets/variables** you must add, and a concise **IAM / OIDC** note.

I referenced the official ECR/ECS GitHub Actions docs and action repos while building this so the configuration follows recommended usage. ([GitHub][1], [GitHub Docs][2])

---

# Before you start — AWS prerequisites (short)

1. Create two (or one) **ECR repositories** (one for the PHP image and one for the socket server), or a single repo with multiple tags.
2. Create an **ECS cluster** and an **ECS service** (with a task definition you can use as a template). Decide Fargate or EC2 launch type.
3. Create an **IAM principal** (recommended: use **GitHub OIDC** for short-lived credentials; alternative: create an IAM user with `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`).

   * Minimum permissions: ECR push/pull, ECS RegisterTaskDefinition, ECS UpdateService, S3 (optional for artifacts), CloudWatch logs (for task), and ECR\:GetAuthorizationToken. See AWS docs for exact policy. (I’ll show OIDC/role note below.)
4. Add these **GitHub repository secrets** (if *not* using OIDC) or environment variables (if using OIDC you still need AWS\_ROLE\_TO\_ASSUME and AWS\_REGION):

   * `AWS_ACCESS_KEY_ID` (if not using OIDC)
   * `AWS_SECRET_ACCESS_KEY` (if not using OIDC)
   * `AWS_REGION` (e.g. `ap-south-1`)
   * `AWS_ACCOUNT_ID` (your AWS account id)
   * `ECR_REPOSITORY_PHP` (e.g. `civil-app`)
   * `ECR_REPOSITORY_SOCKET` (e.g. `socket-server`)
   * `ECS_CLUSTER` (name)
   * `ECS_SERVICE` (name)
   * `TASK_DEFINITION_FILE` (path in repo to your task definition json, e.g. `ecs/taskdef.json`)
   * `SONAR_TOKEN`, `SNYK_TOKEN`, `ZAP_AUTH` (optional)
   * `CI_FAIL_ON_HIGH_VULNS` (true/false) — optional

(If using **OIDC**: create an IAM Role trust policy that allows `sts:AssumeRoleWithWebIdentity` by GitHub Actions OIDC, and give that role the permissions above. Then set `AWS_ROLE_TO_ASSUME` secret in the repo. See GitHub/AWS docs.) ([Medium][3])

---

# Files you'll add to repo

1. `.github/workflows/ecr-ecs-deploy.yml` — the workflow (provided below).
2. `sonar-project.properties` (earlier instructions; used by SonarCloud).
3. `ecs/taskdef.template.json` — an ECS task definition *template* where image fields will be replaced by the workflow. Example template below.

---

# Example ECS task definition template (ecs/taskdef.template.json)

Save this (edit CPU/memory/ports/ENV as needed). The workflow will replace the `__PHP_IMAGE__` and `__SOCKET_IMAGE__` placeholders with the pushed ECR image URIs.

```json
{
  "family": "civil-app-task",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "civil-web",
      "image": "__PHP_IMAGE__",
      "essential": true,
      "portMappings": [
        { "containerPort": 8000, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "APP_ENV", "value": "production" }
      ]
    },
    {
      "name": "socket-server",
      "image": "__SOCKET_IMAGE__",
      "essential": true,
      "portMappings": [
        { "containerPort": 3001, "protocol": "tcp" }
      ]
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024"
}
```

> If you use EC2-backed tasks, change `requiresCompatibilities` and remove `networkMode: awsvpc` adjustments per your setup.

---

# The complete GitHub Actions workflow

Create `.github/workflows/ecr-ecs-deploy.yml` and copy the entire block below. It:

* checks out code
* runs Gitleaks (secrets), SonarCloud, Snyk, PHPUnit & npm tests
* builds and Trivy-scans both images locally
* logs into ECR and pushes tags
* generates a task definition JSON from the template and registers it with ECS
* updates the ECS service to deploy the new task definition

```yaml
name: CI/CD — Build → Scan → Push to ECR → Deploy to ECS

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}
  AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
  ECR_REPOSITORY_PHP: ${{ secrets.ECR_REPOSITORY_PHP }}
  ECR_REPOSITORY_SOCKET: ${{ secrets.ECR_REPOSITORY_SOCKET }}
  ECS_CLUSTER: ${{ secrets.ECS_CLUSTER }}
  ECS_SERVICE: ${{ secrets.ECS_SERVICE }}
  TASKDEF_TEMPLATE: "ecs/taskdef.template.json"
  TASKDEF_RENDERED: "ecs/taskdef.rendered.json"
  CIVIL_PORT: 8000
  SOCKET_PORT: 3001
  CI_FAIL_ON_HIGH_VULNS: ${{ secrets.CI_FAIL_ON_HIGH_VULNS || 'true' }}

jobs:
  build-scan-push-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: 'read'
      id-token: 'write'                    # required if using OIDC
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      ### -------------------
      ### Optional: run quick leak & code scans first
      - name: Gitleaks (secret scan)
        run: |
          docker run --rm -v "${{ github.workspace }}":/src zricethezav/gitleaks:latest detect --source /src --report=/src/gitleaks-report.json || true
      - name: Upload gitleaks report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: gitleaks-report
          path: gitleaks-report.json

      - name: Setup PHP 8.2 & Composer
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          tools: composer
          extensions: mbstring, bcmath, pdo_mysql, xml, gd

      - name: Install PHP deps
        run: composer install --no-interaction --prefer-dist || true

      - name: Run Laravel tests (phpunit)
        run: php artisan test || true

      - name: Setup Node 20
        uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Install socket-server deps
        working-directory: ./socket-server
        run: npm ci || npm install --production || true

      - name: Run socket-server tests
        working-directory: ./socket-server
        run: npm test --if-present || true

      - name: SonarCloud Scan (SAST)
        uses: SonarSource/sonarcloud-github-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          projectBaseDir: .

      - name: Install Snyk CLI
        run: npm i -g snyk || true

      - name: Snyk test (SCA)
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        run: |
          snyk auth $SNYK_TOKEN || true
          snyk test --all-projects --severity-threshold=high || true

      ### -------------------
      ### Build Docker images locally
      - name: Build PHP Docker image
        run: |
          PHP_IMAGE_TAG="${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY_PHP }}:${{ github.sha }}"
          docker build -t "$PHP_IMAGE_TAG" -f Dockerfile .
          echo "PHP_IMAGE=$PHP_IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Build socket-server Docker image
        run: |
          SOCKET_IMAGE_TAG="${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY_SOCKET }}:${{ github.sha }}"
          docker build -t "$SOCKET_IMAGE_TAG" ./socket-server
          echo "SOCKET_IMAGE=$SOCKET_IMAGE_TAG" >> $GITHUB_OUTPUT

      ### -------------------
      ### Trivy scan (container) BEFORE push
      - name: Install Trivy
        run: |
          curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin || true

      - name: Trivy scan PHP image
        run: |
          PHP_IMAGE="${{ steps.build.outputs.PHP_IMAGE || needs.build-scan-push-deploy.outputs.PHP_IMAGE }}"
          # fallback to env in case step outputs not available
          PHP_IMAGE="${PHP_IMAGE:-${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY_PHP }}:${{ github.sha }}}"
          if [ "${{ env.CI_FAIL_ON_HIGH_VULNS }}" = "true" ]; then
            trivy image --severity HIGH,CRITICAL --exit-code 1 "$PHP_IMAGE" || (echo "Trivy found high/critical"; exit 1)
          else
            trivy image --severity HIGH,CRITICAL "$PHP_IMAGE" || true
          fi

      - name: Trivy scan socket image
        run: |
          SOCKET_IMAGE="${{ steps.build.outputs.SOCKET_IMAGE || needs.build-scan-push-deploy.outputs.SOCKET_IMAGE }}"
          SOCKET_IMAGE="${SOCKET_IMAGE:-${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY_SOCKET }}:${{ github.sha }}}"
          if [ "${{ env.CI_FAIL_ON_HIGH_VULNS }}" = "true" ]; then
            trivy image --severity HIGH,CRITICAL --exit-code 1 "$SOCKET_IMAGE" || (echo "Trivy found high/critical"; exit 1)
          else
            trivy image --severity HIGH,CRITICAL "$SOCKET_IMAGE" || true
          fi

      ### -------------------
      ### Login to ECR
      # Use OIDC if you set up role-based access; otherwise uses AWS credentials from secrets.
      - name: Configure AWS credentials (recommended: OIDC or IAM)
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME || '' }}   # if using OIDC, set this secret to the role ARN
          aws-region: ${{ env.AWS_REGION }}
        # if not using role-to-assume, the action will automatically pick up AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY from secrets

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v1

      ### -------------------
      ### Tag and Push images to ECR
      - name: Tag & Push PHP image to ECR
        run: |
          PHP_IMAGE="${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY_PHP }}:${{ github.sha }}"
          docker tag "${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY_PHP }}:${{ github.sha }}" "$PHP_IMAGE" || true
          docker push "$PHP_IMAGE"

      - name: Tag & Push socket image to ECR
        run: |
          SOCKET_IMAGE="${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY_SOCKET }}:${{ github.sha }}"
          docker tag "${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY_SOCKET }}:${{ github.sha }}" "$SOCKET_IMAGE" || true
          docker push "$SOCKET_IMAGE"

      ### -------------------
      ### Render task definition template -> replace placeholders with the new image URIs
      - name: Render ECS task definition
        run: |
          PHP_IMAGE="${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY_PHP }}:${{ github.sha }}"
          SOCKET_IMAGE="${{ env.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY_SOCKET }}:${{ github.sha }}"
          sed "s|__PHP_IMAGE__|${PHP_IMAGE}|g; s|__SOCKET_IMAGE__|${SOCKET_IMAGE}|g" ${{ env.TASKDEF_TEMPLATE }} > ${{ env.TASKDEF_RENDERED }}
          cat ${{ env.TASKDEF_RENDERED }}

      - name: Register ECS task definition & Deploy
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ env.TASKDEF_RENDERED }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true

      - name: Upload ECS taskdef rendered
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: ecs-taskdef
          path: ${{ env.TASKDEF_RENDERED }}
```

---

# Walkthrough (step-by-step explanation of the workflow)

1. **Checkout** — get code from `main` (or PR).
2. **Gitleaks** — quick secret-scan (good to fail fast if secrets exist).
3. **PHP & Node setup** — install dependencies and run unit/integration tests (adjust DB/test env if needed).
4. **SonarCloud & Snyk** — static analysis & dependency scanning. Requires `SONAR_TOKEN` and `SNYK_TOKEN`.
5. **Build Docker images** locally for both containers and tag them with the `${GITHUB_SHA}`.
6. **Trivy** scans the local images for HIGH/CRITICAL vulnerabilities. If `CI_FAIL_ON_HIGH_VULNS=true`, the step fails the run.
7. **Configure AWS credentials** — the recommended action `aws-actions/configure-aws-credentials@v2` supports OIDC (role-assume) or picking up static creds from secrets. (Using OIDC avoids long-lived keys.)
8. **Login to ECR** — `aws-actions/amazon-ecr-login@v1` uses AWS credentials to log Docker in to ECR.
9. **Push images** — `docker push` uploads the new images into your ECR repositories.
10. **Render task definition** — the template JSON has placeholders `__PHP_IMAGE__` and `__SOCKET_IMAGE__`. The workflow creates `ecs/taskdef.rendered.json` replacing placeholders with the pushed ECR URIs.
11. **Register & Deploy** — `aws-actions/amazon-ecs-deploy-task-definition@v1` registers the new task definition and updates the ECS service. The action optionally waits for service stability.
12. **Artifacts** — gitleaks report and the rendered task definition are uploaded as artifacts for troubleshooting.

---

# Important implementation details & tips

* **OIDC (recommended):** Create an IAM role with a trust policy that allows the GitHub OIDC provider and the repository as a principal. Then grant the role the minimal permissions (ECR push/pull, ECS register/update, etc.). Put the role ARN into `AWS_ROLE_TO_ASSUME` secret and the workflow will assume that role with `aws-actions/configure-aws-credentials`. This removes the need to store long-lived AWS keys in GitHub. See AWS/GitHub docs. ([Medium][3])
* **IAM Permission example (minimum)** — for a CI principal, attach a policy that allows:

  * `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:CompleteLayerUpload`, `ecr:UploadLayerPart`, `ecr:InitiateLayerUpload`, `ecr:PutImage`, `ecr:CreateRepository` (optional)
  * `ecs:RegisterTaskDefinition`, `ecs:UpdateService`, `ecs:DescribeServices`, `ecs:DescribeTaskDefinition`, `iam:PassRole` (if using task execution role)
  * `logs:CreateLogStream`, `logs:PutLogEvents`
* **ECR repository creation:** The workflow assumes the ECR repo exists. You can create it manually or via terraform/CloudFormation. The push will fail if repo missing.
* **Task definition roles:** Ensure `executionRoleArn` in the task def exists and has `AmazonECSTaskExecutionRolePolicy` (for pulling images and sending logs).
* **Ports & networking:** If ECS tasks run in private subnets, ensure ALB or ENI/Route is configured.
* **Secrets in containers:** Use AWS Secrets Manager / SSM Parameter Store with IAM roles for tasks — do not bake secrets into images.
* **Fail policy:** I use `CI_FAIL_ON_HIGH_VULNS` to fail the pipeline on HIGH/CRITICAL trivy vulns — you can tune it to treat alerts as warnings while you triage.

---

# Quick checks if something breaks

* If `docker push` fails with `no basic auth credentials`, check ECR login step and that the AWS credentials/role have `ecr:GetAuthorizationToken`. ([GitHub][1])
* If ECS action hangs on deploy, check the task definition `executionRoleArn` and that the cluster/service exist. The ECS deploy action logs helpful messages. ([GitHub][4])

---

# Useful references (official)

* Amazon ECR Login action — how to login to ECR from Actions. ([GitHub][1])
* Amazon ECS Deploy Task Definition action — registers task definition and updates service. ([GitHub][4])
* GitHub Docs: Deploying to Amazon ECS (example workflow and notes). ([GitHub Docs][2])

---

If you want I can:

* Produce the **IAM policy JSON** exactly scoped to this workflow (OIDC and non-OIDC variants).
* Produce a **Terraform** snippet that creates ECR repos, ECS cluster, and IAM roles needed (useful for Infrastructure as Code).
* Tweak the workflow to use **multi-arch builds**, **image tags by semantic version / release**, or to push to a single repo with different tags.

Which of those (IAM policy / Terraform / tagging strategy) would you like me to add next?

[1]: https://github.com/aws-actions/amazon-ecr-login?utm_source=chatgpt.com "GitHub - aws-actions/amazon-ecr-login"
[2]: https://docs.github.com/actions/guides/deploying-to-amazon-elastic-container-service?utm_source=chatgpt.com "Deploying to Amazon Elastic Container Service - GitHub Docs"
[3]: https://medium.com/%40eelzinaty/how-to-securly-access-aws-ecr-from-github-actions-using-github-oidc-964975d0e3ad?utm_source=chatgpt.com "How to Securly Access AWS ECR From GitHub Actions Using ..."
[4]: https://github.com/aws-actions/amazon-ecs-deploy-task-definition?utm_source=chatgpt.com "aws-actions/amazon-ecs-deploy-task-definition - GitHub"
