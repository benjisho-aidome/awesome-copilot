---
applyTo: '.github/workflows/**/*.yml,.github/workflows/**/*.yaml'
description: 'Security-focused best practices for GitHub Actions workflows including action pinning, permissions least privilege, secrets handling, OIDC, concurrency, caching, and supply-chain security'
---

# GitHub Actions Workflows Instructions

## Your Mission

Create secure, efficient, and reliable GitHub Actions workflows with emphasis on security hardening, supply-chain safety, and operational best practices.

## Action Pinning Strategy

### **Pin Actions to Full Commit SHA**

```yaml
# ❌ BAD: Mutable references
- uses: actions/checkout@main
- uses: actions/checkout@latest
- uses: actions/checkout@v4

# ✅ BEST: Immutable full-length commit SHA
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1

# ✅ ACCEPTABLE: Major version tag (less secure but easier to maintain)
- uses: actions/checkout@v4
```

**Rationale:** Full SHA pinning prevents supply-chain attacks where tags can be moved to malicious code.

### **Automated Action Updates**

Use Dependabot to keep actions current:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    commit-message:
      prefix: "ci"
      include: "scope"
```

## Permissions Least Privilege

### **Workflow-Level Permissions**

```yaml
name: CI/CD Pipeline

# Set restrictive defaults at workflow level
permissions:
  contents: read  # Default to read-only

jobs:
  build:
    runs-on: ubuntu-latest
    # Job inherits workflow permissions (contents: read)
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
  
  publish:
    needs: build
    runs-on: ubuntu-latest
    # Override for specific needs
    permissions:
      contents: read
      packages: write  # Only this job can publish
    steps:
      - uses: actions/checkout@v4
      - name: Publish package
        run: npm publish
```

### **Available Permission Scopes**

```yaml
permissions:
  actions: read|write|none           # GitHub Actions artifacts
  checks: read|write|none            # Check runs
  contents: read|write|none          # Repository contents
  deployments: read|write|none       # Deployments
  discussions: read|write|none       # Discussions
  id-token: write|none               # OIDC token
  issues: read|write|none            # Issues
  packages: read|write|none          # GitHub Packages
  pages: read|write|none             # GitHub Pages
  pull-requests: read|write|none     # Pull requests
  repository-projects: read|write|none  # Projects
  security-events: read|write|none   # Security events
  statuses: read|write|none          # Commit statuses
```

### **Example: Minimal Permissions for Common Workflows**

```yaml
# Read-only workflow (linting, testing)
permissions:
  contents: read

# PR comment workflow
permissions:
  contents: read
  pull-requests: write

# Package publishing
permissions:
  contents: read
  packages: write

# Security scanning
permissions:
  contents: read
  security-events: write

# Release creation
permissions:
  contents: write
```

## Secrets Handling

### **Never Expose Secrets in Logs**

```yaml
# ❌ BAD: Secret might leak in logs
- name: Deploy
  run: echo "Deploying with key ${{ secrets.API_KEY }}"

# ✅ GOOD: Pass as environment variable
- name: Deploy
  env:
    API_KEY: ${{ secrets.API_KEY }}
  run: ./deploy.sh
```

### **Environment-Specific Secrets**

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Environment protection rules apply
    steps:
      - name: Deploy to Production
        env:
          # Scoped to 'production' environment
          PROD_API_KEY: ${{ secrets.PROD_API_KEY }}
        run: ./deploy.sh
```

### **Conditional Secret Access**

```yaml
# Only access secrets on main branch
- name: Deploy
  if: github.ref == 'refs/heads/main'
  env:
    DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
  run: ./deploy.sh
```

### **Secret Scanning Protection**

Enable GitHub secret scanning:
- Repository Settings → Security → Secret scanning
- Enable push protection to prevent secret commits

## OIDC for Cloud Authentication

### **AWS OIDC Example**

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Required for OIDC
      contents: read
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1
      
      - name: Deploy to S3
        run: |
          aws s3 sync ./dist s3://my-bucket/
```

**AWS IAM Trust Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:org/repo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

### **Azure OIDC Example**

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Deploy to Azure
        run: |
          az webapp deploy --name myapp --resource-group myrg --src-path ./app.zip
```

### **GCP OIDC Example**

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
          service_account: 'github-actions@my-project.iam.gserviceaccount.com'
      
      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy myapp --image gcr.io/my-project/myapp:latest
```

## Concurrency Control

### **Prevent Concurrent Deployments**

```yaml
name: Deploy

on:
  push:
    branches: [main]

# Only one deployment at a time
concurrency:
  group: production-deployment
  cancel-in-progress: false  # Wait for current deployment to finish

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### **Cancel In-Progress PR Builds**

```yaml
name: CI

on:
  pull_request:

# Cancel old builds when new commit pushed
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test
```

### **Branch-Specific Concurrency**

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}
```

## Caching Strategies

### **Node.js Caching**

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'  # Built-in caching

- run: npm ci
```

### **Manual Caching**

```yaml
- name: Cache Dependencies
  uses: actions/cache@v3
  with:
    path: |
      ~/.npm
      ~/.cache
      node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

### **Docker Layer Caching**

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and Push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myapp:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## Artifacts Management

### **Upload Build Artifacts**

```yaml
- name: Build Application
  run: npm run build

- name: Upload Artifacts
  uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
    retention-days: 7  # Auto-delete after 7 days
```

### **Download Artifacts**

```yaml
- name: Download Build Artifacts
  uses: actions/download-artifact@v4
  with:
    name: build-output
    path: dist/
```

### **Artifact Retention Strategy**

```yaml
# Short retention for PR builds
- uses: actions/upload-artifact@v4
  if: github.event_name == 'pull_request'
  with:
    retention-days: 3

# Longer retention for releases
- uses: actions/upload-artifact@v4
  if: startsWith(github.ref, 'refs/tags/')
  with:
    retention-days: 90
```

## Security Hardening

### **Dependency Review**

```yaml
name: Dependency Review

on:
  pull_request:

permissions:
  contents: read

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Dependency Review
        uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: moderate
          deny-licenses: GPL-2.0, GPL-3.0
```

### **CodeQL Analysis**

```yaml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'  # Weekly on Monday

permissions:
  security-events: write
  contents: read

jobs:
  analyze:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        language: [javascript, python]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
      
      - name: Autobuild
        uses: github/codeql-action/autobuild@v3
      
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
```

### **Container Scanning**

```yaml
- name: Build Docker Image
  run: docker build -t myapp:${{ github.sha }} .

- name: Scan Image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'

- name: Upload Trivy Results
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

### **Supply Chain Security**

```yaml
# Sign artifacts with Sigstore
- name: Install Cosign
  uses: sigstore/cosign-installer@v3

- name: Sign Container Image
  env:
    COSIGN_EXPERIMENTAL: 1
  run: |
    cosign sign --yes ghcr.io/myorg/myapp:${{ github.sha }}
```

### **SBOM Generation**

```yaml
- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    image: myapp:${{ github.sha }}
    format: spdx-json
    output-file: sbom.spdx.json

- name: Upload SBOM
  uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: sbom.spdx.json
```

## Lint and Validation

### **Actionlint for Workflow Validation**

```yaml
name: Lint Workflows

on:
  pull_request:
    paths:
      - '.github/workflows/**'

jobs:
  actionlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run actionlint
        uses: rliebz/actionlint-action@v1
        with:
          flags: -color
```

### **YAML Linting**

```yaml
- name: YAML Lint
  uses: ibiqlik/action-yamllint@v3
  with:
    file_or_dir: .github/workflows/
    config_data: |
      extends: default
      rules:
        line-length:
          max: 120
        indentation:
          spaces: 2
```

## Environment Protection

### **Required Reviewers**

```yaml
# Repository Settings → Environments → production
# Configure:
# - Required reviewers (1-6 people)
# - Wait timer (0-43,800 minutes)
# - Deployment branches (specific branches only)

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://prod.example.com
    steps:
      - run: ./deploy.sh
```

### **Branch Protection**

Configure branch protection rules:
- Require pull request reviews
- Require status checks to pass
- Require signed commits
- Restrict who can push
- Require deployments to succeed

## Workflow Security Checklist

- [ ] **Actions pinned to full commit SHA** or major version tag
- [ ] **Permissions set to least privilege** (default `contents: read`)
- [ ] **Secrets accessed via environment variables**, not inline
- [ ] **OIDC used for cloud authentication** instead of long-lived credentials
- [ ] **Concurrency control configured** for deployments
- [ ] **Caching implemented** for dependencies
- [ ] **Artifacts retention set appropriately** (7-90 days)
- [ ] **Dependency review enabled** on pull requests
- [ ] **Security scanning integrated** (CodeQL, container scanning)
- [ ] **Workflow validation** with actionlint
- [ ] **Environment protection** configured for production
- [ ] **Branch protection rules** enabled
- [ ] **Secret scanning** enabled with push protection
- [ ] **Supply chain security** (SBOM, signing) implemented
- [ ] **No hardcoded credentials** in workflows
- [ ] **Third-party actions from trusted sources** only

## Best Practices Summary

1. **Pin all actions to full commit SHA** for supply-chain security
2. **Use least privilege permissions** at workflow and job level
3. **Never log secrets** or expose them in outputs
4. **Prefer OIDC over long-lived credentials** for cloud access
5. **Implement concurrency control** to prevent conflicts
6. **Cache dependencies** to speed up workflows
7. **Set appropriate artifact retention** to manage costs
8. **Scan dependencies and code** for vulnerabilities
9. **Validate workflows with actionlint** before merging
10. **Use environment protection** for production deployments
11. **Enable secret scanning** with push protection
12. **Generate and sign SBOMs** for container images
13. **Audit third-party actions** before use
14. **Keep actions updated** with Dependabot
15. **Test workflows in forks** before enabling on main repo

## Additional Resources

- [GitHub Actions Security Hardening](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [OIDC with GitHub Actions](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Actionlint Documentation](https://github.com/rhysd/actionlint)
- [GitHub Supply Chain Security](https://docs.github.com/en/code-security/supply-chain-security)
