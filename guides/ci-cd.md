# CI/CD

Guide for implementing Continuous Integration and Continuous Deployment pipelines.

## Core Concepts

### CI vs CD

| Aspect | Continuous Integration | Continuous Delivery | Continuous Deployment |
|--------|----------------------|-------------------|---------------------|
| **Goal** | Integrate code frequently | Always deployable | Auto-deploy to production |
| **Trigger** | Every commit | Every commit | Every commit |
| **Output** | Tested build | Release candidate | Live in production |
| **Manual step** | None | Deployment approval | None |

### Pipeline Stages

```
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│  Build  │──▶│  Test   │──▶│  Scan   │──▶│ Package │──▶│ Deploy  │
└─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
     │             │             │             │             │
     ▼             ▼             ▼             ▼             ▼
  Compile      Unit +        Security     Container      Staging
  Install     Integration    SAST/DAST    Registry      Production
```

---

## GitHub Actions

### Complete Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ============ BUILD ============
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for versioning
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
      
      - name: Generate version
        id: version
        run: |
          VERSION=$(git describe --tags --always)
          echo "version=$VERSION" >> $GITHUB_OUTPUT
      
      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build
          path: dist/
          retention-days: 1

  # ============ TEST ============
  test-unit:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/lcov.info
          fail_ci_if_error: true

  test-integration:
    runs-on: ubuntu-latest
    needs: build
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Run migrations
        run: npm run db:migrate
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/test_db
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379

  # ============ QUALITY ============
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Check formatting
        run: npm run format:check
      
      - name: Type check
        run: npm run typecheck

  security:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
      
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets

  # ============ PACKAGE ============
  docker:
    runs-on: ubuntu-latest
    needs: [test-unit, test-integration, lint, security]
    if: github.event_name == 'push'
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: build
          path: dist/
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  # ============ DEPLOY ============
  deploy-staging:
    runs-on: ubuntu-latest
    needs: docker
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        run: |
          # Example: Deploy to Kubernetes
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --namespace staging
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG_STAGING }}
      
      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/myapp --namespace staging --timeout=5m
      
      - name: Run smoke tests
        run: |
          npm run test:smoke -- --url=https://staging.example.com

  deploy-production:
    runs-on: ubuntu-latest
    needs: docker
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to production
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --namespace production
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG_PRODUCTION }}
      
      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/myapp --namespace production --timeout=5m
      
      - name: Notify deployment
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployed ${{ github.sha }} to production",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment Complete* :rocket:\n*Version:* ${{ github.sha }}\n*Environment:* Production"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Reusable Workflows

```yaml
# .github/workflows/reusable-test.yml
name: Reusable Test Workflow

on:
  workflow_call:
    inputs:
      node-version:
        required: false
        type: string
        default: '20'
    secrets:
      codecov-token:
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
      
      - run: npm ci
      - run: npm test -- --coverage
      
      - uses: codecov/codecov-action@v4
        if: ${{ secrets.codecov-token }}
        with:
          token: ${{ secrets.codecov-token }}
```

```yaml
# .github/workflows/main.yml
name: Main

on: [push]

jobs:
  test:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '20'
    secrets:
      codecov-token: ${{ secrets.CODECOV_TOKEN }}
```

---

## GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - scan
  - package
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

# ============ BUILD ============
build:
  stage: build
  image: node:20
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/

# ============ TEST ============
test:unit:
  stage: test
  image: node:20
  needs: [build]
  script:
    - npm ci
    - npm run test:unit -- --coverage
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

test:integration:
  stage: test
  image: node:20
  needs: [build]
  services:
    - postgres:16
    - redis:7
  variables:
    POSTGRES_DB: test_db
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgresql://postgres:test@postgres:5432/test_db
    REDIS_URL: redis://redis:6379
  script:
    - npm ci
    - npm run db:migrate
    - npm run test:integration

# ============ SCAN ============
sast:
  stage: scan
  needs: []

dependency_scanning:
  stage: scan
  needs: []

container_scanning:
  stage: scan
  needs: [docker]

# ============ PACKAGE ============
docker:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  needs: [test:unit, test:integration]
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
  only:
    - main
    - develop

# ============ DEPLOY ============
deploy:staging:
  stage: deploy
  image: bitnami/kubectl:latest
  needs: [docker, sast, dependency_scanning]
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - kubectl set image deployment/myapp myapp=$DOCKER_IMAGE --namespace staging
    - kubectl rollout status deployment/myapp --namespace staging
  only:
    - develop

deploy:production:
  stage: deploy
  image: bitnami/kubectl:latest
  needs: [docker, sast, dependency_scanning, container_scanning]
  environment:
    name: production
    url: https://example.com
  script:
    - kubectl set image deployment/myapp myapp=$DOCKER_IMAGE --namespace production
    - kubectl rollout status deployment/myapp --namespace production
  only:
    - main
  when: manual
```

---

## Deployment Strategies

### Blue-Green Deployment

```yaml
# blue-green-deploy.yml
deploy-blue-green:
  runs-on: ubuntu-latest
  steps:
    - name: Deploy to Green
      run: |
        # Deploy new version to green
        kubectl apply -f k8s/deployment-green.yml
        kubectl rollout status deployment/myapp-green
    
    - name: Health check Green
      run: |
        # Verify green is healthy
        curl -f https://green.example.com/health
    
    - name: Switch traffic to Green
      run: |
        # Update service to point to green
        kubectl patch service myapp \
          -p '{"spec":{"selector":{"version":"green"}}}'
    
    - name: Verify switch
      run: |
        sleep 30
        curl -f https://example.com/health
    
    - name: Cleanup Blue (optional)
      run: |
        kubectl delete deployment myapp-blue
```

### Canary Deployment

```yaml
# canary-deploy.yml
deploy-canary:
  runs-on: ubuntu-latest
  steps:
    - name: Deploy canary (10%)
      run: |
        kubectl apply -f k8s/deployment-canary.yml
        # Istio traffic split
        kubectl apply -f - <<EOF
        apiVersion: networking.istio.io/v1beta1
        kind: VirtualService
        metadata:
          name: myapp
        spec:
          hosts:
          - myapp
          http:
          - route:
            - destination:
                host: myapp
                subset: stable
              weight: 90
            - destination:
                host: myapp
                subset: canary
              weight: 10
        EOF
    
    - name: Monitor canary
      run: |
        # Wait and check error rates
        sleep 300
        ERROR_RATE=$(curl -s prometheus/api/v1/query?query=rate(http_errors[5m]))
        if [ "$ERROR_RATE" -gt "0.01" ]; then
          echo "Error rate too high, rolling back"
          exit 1
        fi
    
    - name: Promote canary (50%)
      run: |
        kubectl apply -f - <<EOF
        # ... weight: 50/50
        EOF
        sleep 300
    
    - name: Promote canary (100%)
      run: |
        kubectl apply -f - <<EOF
        # ... weight: 0/100
        EOF
    
    - name: Cleanup stable
      run: |
        kubectl delete deployment myapp-stable
        kubectl scale deployment myapp-canary --replicas=3
```

### Rolling Update

```yaml
# Kubernetes rolling update configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods above desired
      maxUnavailable: 0  # Zero downtime
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
```

---

## Environment Management

### Environment Variables

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    env:
      # Non-sensitive
      NODE_ENV: production
      LOG_LEVEL: info
    steps:
      - name: Deploy
        run: ./deploy.sh
        env:
          # Sensitive from secrets
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          API_KEY: ${{ secrets.API_KEY }}
```

### Environment Protection Rules

```yaml
# GitHub Environment settings (via UI or API)
environments:
  staging:
    wait-timer: 0
    reviewers: []
    deployment-branch-policy:
      protected_branches: false
      custom_branches:
        - develop
  
  production:
    wait-timer: 5  # 5 minute wait
    reviewers:
      - teams/devops
      - users/lead-dev
    deployment-branch-policy:
      protected_branches: true  # Only main
```

### Multi-Environment Config

```yaml
# config/environments/production.yml
database:
  host: prod-db.example.com
  pool_size: 20
  ssl: true

cache:
  host: prod-redis.example.com
  ttl: 3600

features:
  new_checkout: true
  beta_features: false

monitoring:
  sample_rate: 0.1
  error_reporting: true
```

```yaml
# GitHub Actions matrix for environments
jobs:
  deploy:
    strategy:
      matrix:
        environment: [staging, production]
    environment: ${{ matrix.environment }}
    steps:
      - name: Deploy to ${{ matrix.environment }}
        run: ./deploy.sh ${{ matrix.environment }}
```

---

## Pipeline Optimization

### Caching

```yaml
# Node.js caching
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'

# Python caching
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'

# Docker layer caching
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max

# Custom caching
- uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: gradle-${{ hashFiles('**/*.gradle*') }}
    restore-keys: gradle-
```

### Parallelization

```yaml
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - name: Run tests (shard ${{ matrix.shard }}/4)
        run: npm test -- --shard=${{ matrix.shard }}/4

  # Combine results
  test-results:
    needs: test
    steps:
      - name: Merge coverage
        run: ./merge-coverage.sh
```

### Conditional Execution

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Check for changes
        id: changes
        run: |
          if git diff --name-only HEAD~1 | grep -q "^src/"; then
            echo "src_changed=true" >> $GITHUB_OUTPUT
          fi
      
      - name: Build (if changed)
        if: steps.changes.outputs.src_changed == 'true'
        run: npm run build
```

---

## Monitoring and Notifications

### Slack Notifications

```yaml
- name: Notify on failure
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "Pipeline failed! :x:",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "*Pipeline Failed*\n*Repo:* ${{ github.repository }}\n*Branch:* ${{ github.ref_name }}\n*Commit:* ${{ github.sha }}\n*Author:* ${{ github.actor }}"
            }
          },
          {
            "type": "actions",
            "elements": [
              {
                "type": "button",
                "text": { "type": "plain_text", "text": "View Run" },
                "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
              }
            ]
          }
        ]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Deployment Tracking

```yaml
- name: Create deployment
  uses: chrnorm/deployment-action@v2
  id: deployment
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    environment: production

- name: Deploy
  run: ./deploy.sh

- name: Update deployment status
  uses: chrnorm/deployment-status@v2
  if: always()
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    deployment-id: ${{ steps.deployment.outputs.deployment_id }}
    state: ${{ job.status }}
```

---

## Security in CI/CD

### Secret Management

```yaml
# Use GitHub secrets
env:
  API_KEY: ${{ secrets.API_KEY }}

# Use OIDC for cloud providers (no stored secrets)
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions
    aws-region: us-east-1

# Use HashiCorp Vault
- name: Get secrets from Vault
  uses: hashicorp/vault-action@v2
  with:
    url: https://vault.example.com
    method: jwt
    role: github-actions
    secrets: |
      secret/data/myapp api_key | API_KEY
```

### Signed Commits and Artifacts

```yaml
# Require signed commits
- name: Verify commit signature
  run: |
    git verify-commit HEAD || exit 1

# Sign container images
- name: Sign image
  run: |
    cosign sign --key cosign.key $IMAGE
```

---

## CI/CD Checklist

### Pipeline Design
- [ ] Fast feedback (< 10 min for PR checks)
- [ ] Parallel jobs where possible
- [ ] Effective caching strategy
- [ ] Clear stage separation

### Testing
- [ ] Unit tests in CI
- [ ] Integration tests with services
- [ ] Security scanning (SAST, dependencies)
- [ ] Code coverage tracking

### Deployment
- [ ] Environment protection rules
- [ ] Rollback capability
- [ ] Health checks
- [ ] Deployment notifications

### Security
- [ ] Secrets in secure storage
- [ ] Least privilege permissions
- [ ] Audit logging
- [ ] Signed artifacts
