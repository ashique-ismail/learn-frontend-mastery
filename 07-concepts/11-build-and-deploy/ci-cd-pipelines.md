# CI/CD Pipelines

## Overview

Continuous Integration and Continuous Deployment (CI/CD) automate the process of testing, building, and deploying applications. CI/CD pipelines ensure code quality, reduce manual errors, enable frequent releases, and provide rapid feedback to development teams. Modern web applications rely heavily on robust CI/CD practices to maintain velocity and reliability.

## Core Concepts

### CI/CD Pipeline Flow

```
┌──────────────┐
│  Git Push    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Trigger CI  │ ← GitHub Actions, GitLab CI, etc.
└──────┬───────┘
       │
       ├──────────────┐
       │              │
       ▼              ▼
┌──────────┐   ┌─────────────┐
│   Lint   │   │    Test     │
└──────┬───┘   └──────┬──────┘
       │              │
       └──────┬───────┘
              ▼
       ┌──────────────┐
       │    Build     │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │   Security   │ ← Vulnerability scanning
       │    Scan      │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │    Deploy    │
       │  (Staging)   │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │    Deploy    │
       │ (Production) │
       └──────────────┘
```

## GitHub Actions

### Basic Workflow

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/coverage-final.json
```

### Build and Deploy Workflow

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  NODE_VERSION: '18.x'

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build application
        run: npm run build
        env:
          VITE_API_URL: ${{ secrets.API_URL }}
          VITE_API_KEY: ${{ secrets.API_KEY }}
      
      - name: Run tests
        run: npm test
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-files
          path: dist/
          retention-days: 7
  
  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-files
          path: dist/
      
      - name: Deploy to Staging
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./dist
          cname: staging.example.com
      
      - name: Notify Slack
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Deployed to staging'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
  
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    
    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-files
          path: dist/
      
      - name: Deploy to Production
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Sync to S3
        run: |
          aws s3 sync dist/ s3://my-production-bucket --delete
          aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_ID }} --paths "/*"
```

### Advanced Features

```yaml
# .github/workflows/advanced.yml
name: Advanced Pipeline

on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'package.json'
      - '.github/workflows/**'

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  quality-checks:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0 # Full history for SonarQube
      
      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      
      - name: Install dependencies
        run: npm ci
      
      - name: Type check
        run: npm run type-check
      
      - name: Lint with auto-fix
        run: npm run lint:fix
      
      - name: Format check
        run: npm run format:check
      
      - name: Security audit
        run: npm audit --audit-level=moderate
      
      - name: Dependency check
        run: npx depcheck
      
      - name: Bundle size check
        run: npm run build && npx bundlesize
  
  e2e-tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18.x'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install Playwright
        run: npx playwright install --with-deps
      
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/test
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
  
  lighthouse:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Audit URLs using Lighthouse
        uses: treosh/lighthouse-ci-action@v9
        with:
          urls: |
            https://example.com
            https://example.com/about
          uploadArtifacts: true
          temporaryPublicStorage: true
```

## GitLab CI/CD

### Complete Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - install
  - lint
  - test
  - build
  - deploy

variables:
  NODE_VERSION: "18"
  npm_config_cache: "$CI_PROJECT_DIR/.npm"

cache:
  key:
    files:
      - package-lock.json
  paths:
    - .npm
    - node_modules

install:
  stage: install
  image: node:${NODE_VERSION}
  script:
    - npm ci
  artifacts:
    paths:
      - node_modules
    expire_in: 1 hour

lint:
  stage: lint
  image: node:${NODE_VERSION}
  dependencies:
    - install
  script:
    - npm run lint
    - npm run type-check
  only:
    - merge_requests
    - main

test:unit:
  stage: test
  image: node:${NODE_VERSION}
  dependencies:
    - install
  script:
    - npm run test:unit -- --coverage
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    paths:
      - coverage/
    expire_in: 1 week

test:e2e:
  stage: test
  image: mcr.microsoft.com/playwright:v1.40.0-focal
  services:
    - postgres:15
  variables:
    DATABASE_URL: "postgresql://postgres:password@postgres:5432/test"
  script:
    - npm ci
    - npx playwright install
    - npm run test:e2e
  artifacts:
    when: always
    paths:
      - playwright-report/
      - test-results/
    expire_in: 1 week

build:
  stage: build
  image: node:${NODE_VERSION}
  dependencies:
    - install
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week
  only:
    - main
    - develop

deploy:staging:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache curl
  script:
    - echo "Deploying to staging..."
    - curl -X POST $STAGING_WEBHOOK_URL
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

deploy:production:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache aws-cli
  script:
    - aws s3 sync dist/ s3://$PRODUCTION_BUCKET --delete
    - aws cloudfront create-invalidation --distribution-id $CLOUDFRONT_ID --paths "/*"
  environment:
    name: production
    url: https://example.com
  only:
    - main
  when: manual
```

## CircleCI

```yaml
# .circleci/config.yml
version: 2.1

orbs:
  node: circleci/node@5.1.0
  aws-s3: circleci/aws-s3@3.1.1

executors:
  node-executor:
    docker:
      - image: cimg/node:18.16.0
    working_directory: ~/project

jobs:
  install-and-test:
    executor: node-executor
    steps:
      - checkout
      
      - restore_cache:
          keys:
            - v1-dependencies-{{ checksum "package-lock.json" }}
            - v1-dependencies-
      
      - run:
          name: Install Dependencies
          command: npm ci
      
      - save_cache:
          paths:
            - node_modules
          key: v1-dependencies-{{ checksum "package-lock.json" }}
      
      - run:
          name: Run Linter
          command: npm run lint
      
      - run:
          name: Run Tests
          command: npm test -- --coverage
      
      - store_test_results:
          path: coverage
      
      - store_artifacts:
          path: coverage
  
  build:
    executor: node-executor
    steps:
      - checkout
      
      - restore_cache:
          keys:
            - v1-dependencies-{{ checksum "package-lock.json" }}
      
      - run:
          name: Build Application
          command: npm run build
      
      - persist_to_workspace:
          root: .
          paths:
            - dist
  
  deploy:
    executor: node-executor
    steps:
      - attach_workspace:
          at: .
      
      - aws-s3/sync:
          from: dist
          to: 's3://my-production-bucket'
          arguments: --delete

workflows:
  build-test-deploy:
    jobs:
      - install-and-test
      
      - build:
          requires:
            - install-and-test
      
      - deploy:
          requires:
            - build
          filters:
            branches:
              only: main
```

## Azure Pipelines

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  nodeVersion: '18.x'

stages:
  - stage: Build
    displayName: 'Build and Test'
    jobs:
      - job: BuildJob
        displayName: 'Build and Test Application'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: $(nodeVersion)
            displayName: 'Install Node.js'
          
          - task: Cache@2
            inputs:
              key: 'npm | "$(Agent.OS)" | package-lock.json'
              path: $(npm_config_cache)
            displayName: 'Cache npm packages'
          
          - script: npm ci
            displayName: 'Install dependencies'
          
          - script: npm run lint
            displayName: 'Run linter'
          
          - script: npm test -- --coverage
            displayName: 'Run tests'
          
          - task: PublishCodeCoverageResults@1
            inputs:
              codeCoverageTool: 'Cobertura'
              summaryFileLocation: '$(System.DefaultWorkingDirectory)/coverage/cobertura-coverage.xml'
          
          - script: npm run build
            displayName: 'Build application'
            env:
              VITE_API_URL: $(API_URL)
              VITE_API_KEY: $(API_KEY)
          
          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: 'dist'
              ArtifactName: 'drop'
  
  - stage: Deploy
    displayName: 'Deploy to Azure'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployWeb
        displayName: 'Deploy Web App'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: DownloadBuildArtifacts@0
                  inputs:
                    buildType: 'current'
                    downloadType: 'single'
                    artifactName: 'drop'
                    downloadPath: '$(System.ArtifactsDirectory)'
                
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'Azure-Subscription'
                    appName: 'my-web-app'
                    package: '$(System.ArtifactsDirectory)/drop'
```

## Testing Automation

```yaml
# .github/workflows/test.yml
name: Test Automation

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [16, 18, 20]
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
      
      - run: npm ci
      - run: npm run test:unit
  
  integration-tests:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
      
      mongodb:
        image: mongo:6
        options: >-
          --health-cmd "mongosh --eval 'db.runCommand({ ping: 1 })'"
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test:integration
        env:
          REDIS_URL: redis://redis:6379
          MONGODB_URL: mongodb://mongodb:27017/test
  
  visual-regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run build
      
      - name: Percy visual tests
        run: npm run percy
        env:
          PERCY_TOKEN: ${{ secrets.PERCY_TOKEN }}
  
  accessibility-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run build
      - run: npm run test:a11y
```

## Deployment Strategies

### Blue-Green Deployment

```yaml
# .github/workflows/blue-green.yml
name: Blue-Green Deployment

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build application
        run: npm run build
      
      - name: Deploy to Green environment
        run: |
          aws deploy create-deployment \
            --application-name my-app \
            --deployment-group-name green \
            --s3-location bucket=my-bucket,key=app.zip,bundleType=zip
      
      - name: Run smoke tests
        run: npm run test:smoke -- --url https://green.example.com
      
      - name: Switch traffic to Green
        run: |
          aws elbv2 modify-listener \
            --listener-arn ${{ secrets.LISTENER_ARN }} \
            --default-actions Type=forward,TargetGroupArn=${{ secrets.GREEN_TARGET_GROUP }}
      
      - name: Wait and monitor
        run: sleep 300 # 5 minutes
      
      - name: Rollback if needed
        if: failure()
        run: |
          aws elbv2 modify-listener \
            --listener-arn ${{ secrets.LISTENER_ARN }} \
            --default-actions Type=forward,TargetGroupArn=${{ secrets.BLUE_TARGET_GROUP }}
```

### Canary Deployment

```yaml
# .github/workflows/canary.yml
name: Canary Deployment

on:
  workflow_dispatch:
    inputs:
      canary_percentage:
        description: 'Percentage of traffic to canary'
        required: true
        default: '10'

jobs:
  canary-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy canary version
        run: |
          kubectl set image deployment/my-app \
            app=my-app:${{ github.sha }} \
            --record
      
      - name: Route traffic to canary
        run: |
          kubectl patch service my-app \
            -p '{"spec":{"selector":{"version":"canary"}}}'
      
      - name: Monitor metrics
        run: |
          # Check error rates, latency, etc.
          npm run monitor:canary -- --duration 10m
      
      - name: Promote or rollback
        run: |
          if [ "${{ job.status }}" == "success" ]; then
            kubectl set image deployment/my-app app=my-app:${{ github.sha }}
          else
            kubectl rollout undo deployment/my-app
          fi
```

## Common Mistakes

1. **Not caching dependencies** - slow pipeline runs
2. **Running tests sequentially** - wasted time
3. **No artifact retention policy** - storage bloat
4. **Missing rollback strategy** - risky deployments
5. **Hardcoded secrets** - security vulnerability
6. **Not using matrix builds** - missing OS/version issues
7. **No deployment gates** - automatic production deploys
8. **Missing notifications** - team unaware of failures

## Best Practices

1. Use caching for dependencies and build artifacts
2. Run jobs in parallel when possible
3. Implement proper secret management
4. Add manual approval for production deploys
5. Use deployment environments with protection rules
6. Implement health checks and rollback mechanisms
7. Monitor pipeline performance and optimize
8. Set up notifications for failures
9. Use artifacts for build outputs
10. Document pipeline requirements and setup

## Key Takeaways

1. CI/CD automates testing, building, and deployment
2. Multiple platform options (GitHub, GitLab, CircleCI, Azure)
3. Parallel jobs significantly reduce pipeline time
4. Caching is critical for performance
5. Secrets must be properly managed
6. Deployment strategies reduce risk
7. Testing automation ensures code quality
8. Artifacts enable efficient workflows
9. Manual gates protect production
10. Monitoring and notifications are essential

## Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitLab CI/CD](https://docs.gitlab.com/ee/ci/)
- [CircleCI Documentation](https://circleci.com/docs/)
- [Azure Pipelines](https://docs.microsoft.com/en-us/azure/devops/pipelines/)
- [CI/CD Best Practices](https://www.atlassian.com/continuous-delivery/principles/continuous-integration-vs-delivery-vs-deployment)
