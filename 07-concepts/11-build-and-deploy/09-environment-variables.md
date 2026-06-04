# Environment Variables

## Overview

Environment variables are dynamic key-value pairs that affect how running processes behave. They are essential for configuring applications across different environments (development, staging, production) without modifying code. Proper environment variable management is critical for security, deployment flexibility, and maintaining the twelve-factor app methodology.

## Table of Contents
- [Core Concepts](#core-concepts)
- [Build-Time vs Runtime Variables](#build-time-vs-runtime-variables)
- [Environment Files (.env)](#environment-files-env)
- [Secret Management](#secret-management)
- [Framework-Specific Implementations](#framework-specific-implementations)
- [Cloud Provider Solutions](#cloud-provider-solutions)
- [Docker and Containerization](#docker-and-containerization)
- [CI/CD Integration](#cicd-integration)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Core Concepts

### What Are Environment Variables?

```javascript
// Accessing environment variables in Node.js
console.log(process.env.NODE_ENV);        // 'development'
console.log(process.env.API_URL);         // 'https://api.example.com'
console.log(process.env.DATABASE_URL);    // 'postgresql://...'

// Setting environment variables programmatically
process.env.CUSTOM_VAR = 'value';

// Checking if variable exists
if (process.env.FEATURE_FLAG_ENABLED === 'true') {
  // Enable feature
}
```

### Environment Variable Hierarchy

```
System Environment Variables (Highest Priority)
         ↓
Shell Environment Variables
         ↓
.env.local (Git-ignored, local overrides)
         ↓
.env.[environment] (.env.production, .env.staging)
         ↓
.env (Base configuration)
         ↓
Default Values (Lowest Priority)
```

## Build-Time vs Runtime Variables

### Build-Time Variables

Variables embedded during build process, becoming static values in output.

```javascript
// React - Build-time variables
// .env.production
REACT_APP_API_URL=https://api.production.com
REACT_APP_VERSION=1.2.3
REACT_APP_FEATURE_X=true

// src/config.ts
export const config = {
  apiUrl: process.env.REACT_APP_API_URL,     // Replaced at build time
  version: process.env.REACT_APP_VERSION,
  featureX: process.env.REACT_APP_FEATURE_X === 'true'
};

// After build, in bundle.js:
// const config = {
//   apiUrl: "https://api.production.com",
//   version: "1.2.3",
//   featureX: true
// };
```

```typescript
// Angular - Build-time variables
// environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.production.com',
  version: '1.2.3'
};

// Using in component
import { environment } from '../environments/environment';

@Component({
  selector: 'app-root',
  template: `<h1>Version: {{version}}</h1>`
})
export class AppComponent {
  version = environment.version;
  
  constructor(private http: HttpClient) {
    // API URL is hardcoded after build
    this.http.get(`${environment.apiUrl}/data`);
  }
}
```

### Runtime Variables

Variables read when application runs, allowing dynamic configuration.

```javascript
// Node.js/Express - Runtime variables
// server.js
const express = require('express');
const app = express();

// Read at runtime
const PORT = process.env.PORT || 3000;
const DATABASE_URL = process.env.DATABASE_URL;
const API_KEY = process.env.API_KEY;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

// Can be changed without rebuilding
// docker run -e PORT=8080 -e API_KEY=xyz myapp
```

```javascript
// Next.js - Runtime public variables
// next.config.js
module.exports = {
  publicRuntimeConfig: {
    apiUrl: process.env.API_URL,
  },
  serverRuntimeConfig: {
    apiKey: process.env.API_KEY,  // Server-side only
  },
};

// pages/index.js
import getConfig from 'next/config';

const { publicRuntimeConfig, serverRuntimeConfig } = getConfig();

export default function Home() {
  // Available on client
  console.log(publicRuntimeConfig.apiUrl);
  
  return <div>Home</div>;
}

export async function getServerSideProps() {
  // Only available on server
  console.log(serverRuntimeConfig.apiKey);
  
  return { props: {} };
}
```

### Runtime Configuration Pattern

```typescript
// runtime-config.ts
// Load configuration at runtime (not build time)
class RuntimeConfig {
  private config: Record<string, string> = {};
  
  async load(): Promise<void> {
    // Fetch from server endpoint
    const response = await fetch('/api/config');
    this.config = await response.json();
  }
  
  get(key: string, defaultValue?: string): string {
    return this.config[key] || defaultValue || '';
  }
  
  getBoolean(key: string): boolean {
    return this.config[key] === 'true';
  }
  
  getNumber(key: string, defaultValue: number = 0): number {
    const value = this.config[key];
    return value ? parseInt(value, 10) : defaultValue;
  }
}

export const runtimeConfig = new RuntimeConfig();

// app.tsx
import { runtimeConfig } from './runtime-config';

async function initApp() {
  await runtimeConfig.load();
  
  const apiUrl = runtimeConfig.get('API_URL');
  const timeout = runtimeConfig.getNumber('REQUEST_TIMEOUT', 5000);
  
  // Initialize app with runtime config
}

initApp();
```

## Environment Files (.env)

### Basic .env File

```bash
# .env - Base configuration (committed to repo)
# Application
NODE_ENV=development
APP_NAME=MyApp
APP_VERSION=1.0.0
LOG_LEVEL=info

# API Configuration
API_URL=http://localhost:8080
API_TIMEOUT=5000
API_RETRY_ATTEMPTS=3

# Database
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=myapp_dev

# Feature Flags
FEATURE_ADVANCED_ANALYTICS=false
FEATURE_BETA_UI=true

# Third-party Services
STRIPE_PUBLISHABLE_KEY=pk_test_xxx
ANALYTICS_TRACKING_ID=UA-XXXXX-Y
```

```bash
# .env.local - Local overrides (NOT committed, in .gitignore)
# Secrets and personal configuration
DATABASE_PASSWORD=local_dev_password
API_KEY=dev_api_key_12345
JWT_SECRET=local_jwt_secret

# Local service URLs
API_URL=http://localhost:3001
REDIS_URL=redis://localhost:6379
```

```bash
# .env.production - Production values (committed, no secrets)
NODE_ENV=production
LOG_LEVEL=error

API_URL=https://api.production.com
DATABASE_HOST=prod-db.example.com
DATABASE_PORT=5432

# Secrets injected at runtime via CI/CD or secret manager
# DATABASE_PASSWORD, API_KEY, etc.
```

### Using dotenv in Node.js

```javascript
// config/env.js
require('dotenv').config();

// Load environment-specific file
const envFile = `.env.${process.env.NODE_ENV || 'development'}`;
require('dotenv').config({ path: envFile });

module.exports = {
  nodeEnv: process.env.NODE_ENV || 'development',
  port: parseInt(process.env.PORT || '3000', 10),
  
  database: {
    host: process.env.DATABASE_HOST,
    port: parseInt(process.env.DATABASE_PORT || '5432', 10),
    name: process.env.DATABASE_NAME,
    user: process.env.DATABASE_USER,
    password: process.env.DATABASE_PASSWORD,
  },
  
  api: {
    url: process.env.API_URL,
    key: process.env.API_KEY,
    timeout: parseInt(process.env.API_TIMEOUT || '5000', 10),
  },
  
  features: {
    analytics: process.env.FEATURE_ADVANCED_ANALYTICS === 'true',
    betaUI: process.env.FEATURE_BETA_UI === 'true',
  },
};
```

```typescript
// config/env.ts - Type-safe configuration
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']),
  PORT: z.string().transform(Number).default('3000'),
  
  DATABASE_URL: z.string().url(),
  DATABASE_POOL_SIZE: z.string().transform(Number).default('10'),
  
  API_URL: z.string().url(),
  API_KEY: z.string().min(10),
  
  JWT_SECRET: z.string().min(32),
  JWT_EXPIRY: z.string().default('7d'),
  
  REDIS_URL: z.string().url().optional(),
  
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

// Validate and parse environment variables
export const env = envSchema.parse(process.env);

// TypeScript now knows the exact types
console.log(env.PORT); // number
console.log(env.NODE_ENV); // 'development' | 'staging' | 'production'
```

### React Environment Variables

```javascript
// .env.development
REACT_APP_API_URL=http://localhost:8080
REACT_APP_ENVIRONMENT=development
REACT_APP_ENABLE_ANALYTICS=false
REACT_APP_DEBUG_MODE=true

// .env.production
REACT_APP_API_URL=https://api.production.com
REACT_APP_ENVIRONMENT=production
REACT_APP_ENABLE_ANALYTICS=true
REACT_APP_DEBUG_MODE=false

// src/config/env.ts
export const config = {
  apiUrl: process.env.REACT_APP_API_URL || '',
  environment: process.env.REACT_APP_ENVIRONMENT || 'development',
  enableAnalytics: process.env.REACT_APP_ENABLE_ANALYTICS === 'true',
  debugMode: process.env.REACT_APP_DEBUG_MODE === 'true',
};

// Validation
const requiredEnvVars = ['REACT_APP_API_URL', 'REACT_APP_ENVIRONMENT'];

requiredEnvVars.forEach(envVar => {
  if (!process.env[envVar]) {
    throw new Error(`Missing required environment variable: ${envVar}`);
  }
});
```

### Angular Environment Variables

```typescript
// environments/environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:8080',
  enableAnalytics: false,
  debugMode: true,
};

// environments/environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.production.com',
  enableAnalytics: true,
  debugMode: false,
};

// angular.json - File replacements
{
  "configurations": {
    "production": {
      "fileReplacements": [
        {
          "replace": "src/environments/environment.ts",
          "with": "src/environments/environment.prod.ts"
        }
      ]
    }
  }
}

// Using custom environment variables with Angular
// set-env.ts
const fs = require('fs');
const targetPath = './src/environments/environment.ts';

const envConfigFile = `export const environment = {
  production: ${process.env.PRODUCTION === 'true'},
  apiUrl: '${process.env.API_URL}',
  apiKey: '${process.env.API_KEY}',
};
`;

fs.writeFileSync(targetPath, envConfigFile);
console.log('Environment file generated');

// package.json
{
  "scripts": {
    "config": "node set-env.ts",
    "build:prod": "npm run config && ng build --configuration production"
  }
}
```

## Secret Management

### HashiCorp Vault Integration

```javascript
// vault-client.js
const vault = require('node-vault');

class VaultClient {
  constructor() {
    this.client = vault({
      apiVersion: 'v1',
      endpoint: process.env.VAULT_ADDR,
      token: process.env.VAULT_TOKEN,
    });
  }
  
  async getSecrets(path) {
    try {
      const result = await this.client.read(path);
      return result.data;
    } catch (error) {
      console.error('Failed to read secrets from Vault:', error);
      throw error;
    }
  }
  
  async getSecret(path, key) {
    const secrets = await this.getSecrets(path);
    return secrets[key];
  }
}

// Usage
const vaultClient = new VaultClient();

async function loadConfig() {
  const secrets = await vaultClient.getSecrets('secret/data/myapp/production');
  
  return {
    database: {
      password: secrets.database_password,
    },
    api: {
      key: secrets.api_key,
    },
  };
}
```

### AWS Secrets Manager

```javascript
// aws-secrets.js
const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');

class AWSSecretsManager {
  constructor() {
    this.client = new SecretsManagerClient({
      region: process.env.AWS_REGION || 'us-east-1',
    });
  }
  
  async getSecret(secretName) {
    try {
      const command = new GetSecretValueCommand({ SecretId: secretName });
      const response = await this.client.send(command);
      
      if (response.SecretString) {
        return JSON.parse(response.SecretString);
      }
      
      // Binary secret
      const buff = Buffer.from(response.SecretBinary, 'base64');
      return buff.toString('ascii');
    } catch (error) {
      console.error('Error retrieving secret:', error);
      throw error;
    }
  }
}

// Usage
const secretsManager = new AWSSecretsManager();

async function initializeApp() {
  const dbSecrets = await secretsManager.getSecret('production/database');
  const apiSecrets = await secretsManager.getSecret('production/api');
  
  process.env.DATABASE_PASSWORD = dbSecrets.password;
  process.env.API_KEY = apiSecrets.key;
  
  // Start application
  require('./app');
}

initializeApp();
```

### Azure Key Vault

```typescript
// azure-keyvault.ts
import { SecretClient } from '@azure/keyvault-secrets';
import { DefaultAzureCredential } from '@azure/identity';

class AzureKeyVault {
  private client: SecretClient;
  
  constructor() {
    const keyVaultName = process.env.AZURE_KEYVAULT_NAME;
    const vaultUrl = `https://${keyVaultName}.vault.azure.net`;
    
    const credential = new DefaultAzureCredential();
    this.client = new SecretClient(vaultUrl, credential);
  }
  
  async getSecret(secretName: string): Promise<string> {
    try {
      const secret = await this.client.getSecret(secretName);
      return secret.value || '';
    } catch (error) {
      console.error(`Failed to retrieve secret ${secretName}:`, error);
      throw error;
    }
  }
  
  async setSecret(secretName: string, value: string): Promise<void> {
    await this.client.setSecret(secretName, value);
  }
}

// Usage
const keyVault = new AzureKeyVault();

export async function loadSecrets() {
  const [databasePassword, apiKey, jwtSecret] = await Promise.all([
    keyVault.getSecret('database-password'),
    keyVault.getSecret('api-key'),
    keyVault.getSecret('jwt-secret'),
  ]);
  
  return {
    databasePassword,
    apiKey,
    jwtSecret,
  };
}
```

### Encrypted .env Files

```javascript
// Using dotenv-vault
// Install: npm install dotenv-vault-core

// .env.vault
DOTENV_VAULT_PRODUCTION="encrypted_content_here"
DOTENV_VAULT_STAGING="encrypted_content_here"

// Load encrypted secrets
require('dotenv').config();
require('dotenv-vault-core').config();

// Secrets are decrypted using DOTENV_KEY environment variable
// Set in CI/CD: DOTENV_KEY=dotenv://:key_xxx@dotenv.org/vault/.env.vault?environment=production
```

## Framework-Specific Implementations

### Next.js Environment Variables

```javascript
// next.config.js
module.exports = {
  // Public variables (exposed to browser)
  env: {
    CUSTOM_KEY: process.env.CUSTOM_KEY,
  },
  
  // Runtime configuration
  publicRuntimeConfig: {
    apiUrl: process.env.API_URL,
  },
  
  serverRuntimeConfig: {
    apiSecret: process.env.API_SECRET,
  },
};

// .env.local
NEXT_PUBLIC_API_URL=https://api.example.com
NEXT_PUBLIC_GA_ID=UA-XXXXX-Y
API_SECRET=secret_key_here
DATABASE_URL=postgresql://...

// pages/index.tsx
// NEXT_PUBLIC_* variables available in browser
export default function Home() {
  return (
    <div>
      API URL: {process.env.NEXT_PUBLIC_API_URL}
    </div>
  );
}

// API route - All variables available
export default async function handler(req, res) {
  const apiSecret = process.env.API_SECRET;
  const dbUrl = process.env.DATABASE_URL;
  
  // Server-side only
  res.json({ message: 'Success' });
}
```

### Vite Environment Variables

```javascript
// .env
VITE_API_URL=http://localhost:8080
VITE_APP_TITLE=My App

// .env.production
VITE_API_URL=https://api.production.com

// vite.config.ts
import { defineConfig, loadEnv } from 'vite';

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '');
  
  return {
    define: {
      __APP_VERSION__: JSON.stringify(env.npm_package_version),
    },
    server: {
      port: parseInt(env.VITE_PORT || '3000'),
    },
  };
});

// src/config.ts
export const config = {
  apiUrl: import.meta.env.VITE_API_URL,
  appTitle: import.meta.env.VITE_APP_TITLE,
  isDev: import.meta.env.DEV,
  isProd: import.meta.env.PROD,
};

// Type definitions
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_APP_TITLE: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### NestJS Configuration

```typescript
// config/configuration.ts
export default () => ({
  port: parseInt(process.env.PORT || '3000', 10),
  database: {
    host: process.env.DATABASE_HOST,
    port: parseInt(process.env.DATABASE_PORT || '5432', 10),
    username: process.env.DATABASE_USER,
    password: process.env.DATABASE_PASSWORD,
    database: process.env.DATABASE_NAME,
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN || '7d',
  },
});

// app.module.ts
import { ConfigModule } from '@nestjs/config';
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [configuration],
      isGlobal: true,
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test')
          .default('development'),
        PORT: Joi.number().default(3000),
        DATABASE_HOST: Joi.string().required(),
        DATABASE_PORT: Joi.number().default(5432),
        JWT_SECRET: Joi.string().required(),
      }),
    }),
  ],
})
export class AppModule {}

// Using configuration
import { ConfigService } from '@nestjs/config';

@Injectable()
export class AppService {
  constructor(private configService: ConfigService) {}
  
  getDatabaseConfig() {
    return {
      host: this.configService.get<string>('database.host'),
      port: this.configService.get<number>('database.port'),
    };
  }
}
```

## Cloud Provider Solutions

### AWS Parameter Store

```javascript
// aws-parameter-store.js
const { SSMClient, GetParameterCommand, GetParametersByPathCommand } = require('@aws-sdk/client-ssm');

class ParameterStore {
  constructor() {
    this.client = new SSMClient({ region: process.env.AWS_REGION });
  }
  
  async getParameter(name, decrypt = true) {
    const command = new GetParameterCommand({
      Name: name,
      WithDecryption: decrypt,
    });
    
    const response = await this.client.send(command);
    return response.Parameter.Value;
  }
  
  async getParametersByPath(path, decrypt = true) {
    const command = new GetParametersByPathCommand({
      Path: path,
      Recursive: true,
      WithDecryption: decrypt,
    });
    
    const response = await this.client.send(command);
    
    const parameters = {};
    response.Parameters.forEach(param => {
      const key = param.Name.split('/').pop();
      parameters[key] = param.Value;
    });
    
    return parameters;
  }
}

// Usage
const paramStore = new ParameterStore();

async function loadAppConfig() {
  const params = await paramStore.getParametersByPath('/myapp/production');
  
  Object.keys(params).forEach(key => {
    const envKey = key.toUpperCase();
    process.env[envKey] = params[key];
  });
}
```

### Google Cloud Secret Manager

```javascript
// gcp-secret-manager.js
const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');

class GCPSecretManager {
  constructor() {
    this.client = new SecretManagerServiceClient();
    this.projectId = process.env.GCP_PROJECT_ID;
  }
  
  async getSecret(secretName, version = 'latest') {
    const name = `projects/${this.projectId}/secrets/${secretName}/versions/${version}`;
    
    try {
      const [version] = await this.client.accessSecretVersion({ name });
      const payload = version.payload.data.toString('utf8');
      return payload;
    } catch (error) {
      console.error(`Error accessing secret ${secretName}:`, error);
      throw error;
    }
  }
  
  async createSecret(secretName, secretValue) {
    const parent = `projects/${this.projectId}`;
    
    const [secret] = await this.client.createSecret({
      parent: parent,
      secretId: secretName,
      secret: {
        replication: {
          automatic: {},
        },
      },
    });
    
    await this.client.addSecretVersion({
      parent: secret.name,
      payload: {
        data: Buffer.from(secretValue, 'utf8'),
      },
    });
  }
}
```

## Docker and Containerization

### Docker Environment Variables

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

# Build-time arguments
ARG NODE_ENV=production
ARG API_URL

# Set as environment variables
ENV NODE_ENV=${NODE_ENV}
ENV API_URL=${API_URL}

COPY package*.json ./
RUN npm ci --only=production

COPY . .

# Default values
ENV PORT=3000
ENV LOG_LEVEL=info

EXPOSE ${PORT}

CMD ["node", "server.js"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build:
      context: .
      args:
        - NODE_ENV=production
        - API_URL=https://api.production.com
    environment:
      # Direct values
      - PORT=3000
      - NODE_ENV=production
      
      # From .env file
      - DATABASE_URL=${DATABASE_URL}
      - API_KEY=${API_KEY}
      
      # From file
      - JWT_SECRET
    env_file:
      - .env
      - .env.production
    ports:
      - "3000:3000"
    secrets:
      - db_password
      - api_key

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    file: ./secrets/api_key.txt
```

```bash
# Running Docker with environment variables
# From command line
docker run -e NODE_ENV=production -e API_KEY=xyz myapp

# From file
docker run --env-file .env.production myapp

# From multiple files
docker run --env-file .env --env-file .env.production myapp
```

### Kubernetes ConfigMaps and Secrets

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  NODE_ENV: "production"
  API_URL: "https://api.production.com"
  LOG_LEVEL: "info"
  app.json: |
    {
      "timeout": 5000,
      "retries": 3
    }

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_PASSWORD: "supersecret"
  API_KEY: "api_key_value"
  JWT_SECRET: "jwt_secret_value"

---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        env:
        # From ConfigMap
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: NODE_ENV
        # From Secret
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DATABASE_PASSWORD
        envFrom:
        # All keys from ConfigMap
        - configMapRef:
            name: app-config
        # All keys from Secret
        - secretRef:
            name: app-secrets
```

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    env:
      NODE_ENV: production
      API_URL: ${{ secrets.API_URL }}
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Create .env file
        run: |
          echo "NODE_ENV=${{ env.NODE_ENV }}" >> .env
          echo "API_URL=${{ secrets.API_URL }}" >> .env
          echo "API_KEY=${{ secrets.API_KEY }}" >> .env
          echo "DATABASE_URL=${{ secrets.DATABASE_URL }}" >> .env
      
      - name: Build
        env:
          REACT_APP_API_URL: ${{ secrets.API_URL }}
          REACT_APP_VERSION: ${{ github.sha }}
        run: |
          npm ci
          npm run build
      
      - name: Deploy to Vercel
        env:
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
        run: |
          vercel --prod \
            --token=$VERCEL_TOKEN \
            --build-env API_URL=${{ secrets.API_URL }} \
            --build-env API_KEY=${{ secrets.API_KEY }}
```

### GitLab CI

```yaml
# .gitlab-ci.yml
variables:
  NODE_ENV: "production"
  API_URL: "https://api.production.com"

stages:
  - build
  - deploy

build:
  stage: build
  script:
    - echo "NODE_ENV=${NODE_ENV}" >> .env
    - echo "API_URL=${API_URL}" >> .env
    - echo "API_KEY=${API_KEY}" >> .env
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
  only:
    - main

deploy:production:
  stage: deploy
  script:
    - kubectl create configmap app-config --from-env-file=.env -o yaml --dry-run=client | kubectl apply -f -
    - kubectl set env deployment/web-app --from=configmap/app-config
    - kubectl set env deployment/web-app DATABASE_PASSWORD="${DATABASE_PASSWORD}"
  environment:
    name: production
  only:
    - main
```

## Common Mistakes

### 1. Hardcoding Secrets in Code

```javascript
// Bad: Secrets in source code
const API_KEY = 'sk_live_abc123xyz';
const DATABASE_PASSWORD = 'password123';

fetch('https://api.example.com', {
  headers: { 'Authorization': `Bearer sk_live_abc123xyz` }
});

// Good: Use environment variables
const API_KEY = process.env.API_KEY;
const DATABASE_PASSWORD = process.env.DATABASE_PASSWORD;

if (!API_KEY) {
  throw new Error('API_KEY is required');
}
```

### 2. Committing .env Files with Secrets

```bash
# Bad: .gitignore missing .env.local
.env

# Good: Ignore all local environment files
.env.local
.env.*.local
.env.development.local
.env.test.local
.env.production.local
```

### 3. Not Validating Environment Variables

```javascript
// Bad: No validation
const config = {
  port: process.env.PORT,
  apiUrl: process.env.API_URL,
};

// Good: Validate required variables
function validateEnv() {
  const required = ['PORT', 'API_URL', 'DATABASE_URL'];
  const missing = required.filter(key => !process.env[key]);
  
  if (missing.length > 0) {
    throw new Error(`Missing required environment variables: ${missing.join(', ')}`);
  }
}

validateEnv();
```

### 4. Exposing Server Secrets to Client

```javascript
// Bad: Exposing secret to browser (React)
// .env
SECRET_API_KEY=secret_key_here

// App.js
const apiKey = process.env.SECRET_API_KEY; // Exposed in bundle!

// Good: Prefix with REACT_APP_ only for public vars
// .env
REACT_APP_PUBLIC_API_URL=https://api.example.com

// API calls should go through your backend
```

### 5. Using String Comparison for Booleans

```javascript
// Bad: Always truthy
if (process.env.FEATURE_ENABLED) {  // "false" is truthy!
  enableFeature();
}

// Good: Explicit comparison
if (process.env.FEATURE_ENABLED === 'true') {
  enableFeature();
}

// Better: Helper function
const parseBoolean = (value) => value === 'true';
const isFeatureEnabled = parseBoolean(process.env.FEATURE_ENABLED);
```

## Best Practices

### 1. Use Different Files for Different Environments

```
.env                # Default values (committed)
.env.development    # Development overrides (committed)
.env.test           # Test overrides (committed)
.env.production     # Production values, no secrets (committed)
.env.local          # Local overrides (NOT committed)
.env.*.local        # Environment-specific local overrides (NOT committed)
```

### 2. Implement Configuration Validation

```typescript
// config/validator.ts
import Joi from 'joi';

const envVarsSchema = Joi.object({
  NODE_ENV: Joi.string()
    .valid('development', 'production', 'test')
    .required(),
  PORT: Joi.number().default(3000),
  DATABASE_URL: Joi.string().uri().required(),
  API_KEY: Joi.string().min(20).required(),
  JWT_SECRET: Joi.string().min(32).required(),
  LOG_LEVEL: Joi.string()
    .valid('debug', 'info', 'warn', 'error')
    .default('info'),
}).unknown();

const { error, value: envVars } = envVarsSchema.validate(process.env);

if (error) {
  throw new Error(`Config validation error: ${error.message}`);
}

export const config = {
  env: envVars.NODE_ENV,
  port: envVars.PORT,
  database: {
    url: envVars.DATABASE_URL,
  },
  api: {
    key: envVars.API_KEY,
  },
  jwt: {
    secret: envVars.JWT_SECRET,
  },
  logLevel: envVars.LOG_LEVEL,
};
```

### 3. Document Required Variables

```bash
# .env.example - Committed to repository
# Application
NODE_ENV=development
PORT=3000

# Database (required)
DATABASE_URL=postgresql://user:password@localhost:5432/dbname

# API Configuration (required)
API_URL=https://api.example.com
API_KEY=your_api_key_here

# Optional Features
ENABLE_ANALYTICS=false
LOG_LEVEL=info

# Instructions:
# 1. Copy this file to .env.local
# 2. Fill in the required values
# 3. Never commit .env.local to git
```

### 4. Use Type-Safe Access

```typescript
// config/env.d.ts
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV: 'development' | 'production' | 'test';
      PORT: string;
      DATABASE_URL: string;
      API_KEY: string;
      JWT_SECRET: string;
      REDIS_URL?: string;
      LOG_LEVEL?: 'debug' | 'info' | 'warn' | 'error';
    }
  }
}

export {};
```

### 5. Implement Secret Rotation

```javascript
// secret-rotation.js
const schedule = require('node-schedule');
const { SecretsManager } = require('./aws-secrets');

class SecretRotation {
  constructor() {
    this.secretsManager = new SecretsManager();
    this.currentSecrets = {};
  }
  
  async rotateSecrets() {
    console.log('Rotating secrets...');
    
    try {
      const newSecrets = await this.secretsManager.getSecrets();
      this.currentSecrets = newSecrets;
      
      // Update application configuration
      process.env.API_KEY = newSecrets.apiKey;
      process.env.JWT_SECRET = newSecrets.jwtSecret;
      
      console.log('Secrets rotated successfully');
    } catch (error) {
      console.error('Secret rotation failed:', error);
    }
  }
  
  startRotation(cronExpression = '0 */6 * * *') {
    // Rotate every 6 hours
    schedule.scheduleJob(cronExpression, () => {
      this.rotateSecrets();
    });
    
    // Initial load
    this.rotateSecrets();
  }
}

const rotation = new SecretRotation();
rotation.startRotation();
```

## When to Use/Not to Use

### When to Use Environment Variables

1. **Configuration That Changes Per Environment**: API URLs, database connections
2. **Sensitive Information**: Passwords, API keys, tokens
3. **Feature Flags**: Enable/disable features per environment
4. **External Service Configuration**: Third-party API keys, service URLs
5. **Build Configuration**: Node environment, logging levels
6. **Infrastructure Settings**: Ports, timeouts, connection pools
7. **CI/CD Pipelines**: Deployment-specific values
8. **Container Orchestration**: Docker, Kubernetes configurations

### When Not to Use Environment Variables

1. **Application Logic**: Business rules should be in code
2. **Large Configuration Objects**: Use configuration files or databases
3. **User Preferences**: Store in databases or user storage
4. **Dynamic Runtime Config**: Use configuration services
5. **Frequently Changing Values**: Use databases or config servers
6. **Structured Data**: Complex JSON/YAML better in files
7. **Public Constants**: Version numbers, app name (unless for builds)
8. **Default Behavior**: Application defaults should be in code

## Interview Questions

### 1. What's the difference between build-time and runtime environment variables?

**Answer**: Build-time variables are embedded during the build process and become static values in the output bundle (e.g., `REACT_APP_*` variables in Create React App). They can't be changed without rebuilding. Runtime variables are read when the application runs, allowing configuration changes without rebuilding (common in Node.js servers). Choose build-time for client-side SPAs and runtime for server-side applications.

### 2. How do you securely manage secrets in a production environment?

**Answer**: Never hardcode secrets or commit them to version control. Use secret management services like AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, or Google Secret Manager. In CI/CD, use encrypted secrets provided by the platform (GitHub Secrets, GitLab Variables). Rotate secrets regularly, use IAM roles instead of credentials where possible, and implement least-privilege access. For Kubernetes, use Sealed Secrets or external-secrets operators.

### 3. How would you handle environment variables in a React application?

**Answer**: React (CRA) requires variables prefixed with `REACT_APP_` to be embedded at build time. Create `.env`, `.env.development`, and `.env.production` files. Access via `process.env.REACT_APP_VAR_NAME`. For runtime configuration, fetch from a `/config` endpoint or use `window._env_` pattern. Never expose secrets in client-side code. For sensitive operations, proxy through your backend API.

### 4. What's the difference between .env, .env.local, and .env.production?

**Answer**: 
- `.env`: Base configuration, committed to git, no secrets
- `.env.local`: Local overrides, NOT committed, can contain secrets
- `.env.production`: Production-specific values, committed but without secrets
- Priority: `.env.local` > `.env.[environment].local` > `.env.[environment]` > `.env`

### 5. How do you validate environment variables at application startup?

**Answer**: Use validation libraries like Joi, Zod, or Yup to define a schema for required variables, their types, and constraints. Parse and validate `process.env` at startup before initializing the application. Throw errors for missing or invalid variables to fail fast. Consider using TypeScript declaration files for type safety.

### 6. How would you implement feature flags using environment variables?

**Answer**: Store boolean flags as string environment variables (`FEATURE_NEW_UI=true`). Parse them properly (`=== 'true'`). For complex flag logic, use dedicated feature flag services (LaunchDarkly, Unleash). Environment variables work well for simple on/off toggles per environment but lack targeting, gradual rollouts, or runtime changes. Consider a hybrid approach: env vars for environment-level flags, services for user-level flags.

### 7. What are the security risks of environment variables?

**Answer**: Environment variables can be logged accidentally, exposed through error messages, visible in process listings (`ps aux`), leaked in crash dumps, or exposed in client-side code. Mitigate by: using secret managers, sanitizing logs, not echoing env vars in scripts, using minimal permissions, encrypting at rest, and never exposing server secrets to clients.

### 8. How do you handle environment variables in Docker containers?

**Answer**: Use `ENV` in Dockerfile for defaults, `ARG` for build-time variables, pass via `docker run -e`, use `--env-file`, or mount as volumes. In docker-compose, use `environment:` or `env_file:`. For sensitive data, use Docker secrets or Kubernetes secrets. Never bake secrets into images. Prefer runtime injection over build-time embedding.

## Key Takeaways

1. **Environment variables separate configuration from code**: Follow the twelve-factor app methodology
2. **Build-time vs runtime matters**: Understand when variables are evaluated and embedded
3. **Never commit secrets to version control**: Use `.gitignore` and secret management services
4. **Validate environment variables at startup**: Fail fast with clear error messages
5. **Use consistent naming conventions**: ALL_CAPS_SNAKE_CASE, descriptive names
6. **Framework-specific prefixes are important**: `REACT_APP_*`, `VITE_*`, `NEXT_PUBLIC_*`
7. **Different environments need different values**: Development, staging, production configurations
8. **Secret management services are essential**: Don't rely solely on `.env` files in production
9. **Client-side apps can't have secrets**: Proxy sensitive operations through backend APIs
10. **Documentation is crucial**: Maintain `.env.example` with all required variables

## Resources

### Documentation
- [The Twelve-Factor App - Config](https://12factor.net/config)
- [dotenv Documentation](https://github.com/motdotla/dotenv)
- [Node.js process.env](https://nodejs.org/api/process.html#process_process_env)

### Secret Management
- [HashiCorp Vault](https://www.vaultproject.io/)
- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)
- [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/)
- [Google Secret Manager](https://cloud.google.com/secret-manager)

### Tools
- [dotenv](https://github.com/motdotla/dotenv) - Load environment variables
- [dotenv-expand](https://github.com/motdotla/dotenv-expand) - Variable expansion
- [env-cmd](https://github.com/toddbluhm/env-cmd) - Run commands with env files
- [cross-env](https://github.com/kentcdodds/cross-env) - Cross-platform env vars
- [envalid](https://github.com/af/envalid) - Environment variable validation

### Security
- [OWASP Secrets Management](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [GitGuardian](https://www.gitguardian.com/) - Secret scanning
- [TruffleHog](https://github.com/trufflesecurity/trufflehog) - Find secrets in git

### Articles
- [Environment Variables in React](https://create-react-app.dev/docs/adding-custom-environment-variables/)
- [Environment Variables in Next.js](https://nextjs.org/docs/basic-features/environment-variables)
- [Docker Environment Variables Best Practices](https://docs.docker.com/engine/reference/builder/#env)
