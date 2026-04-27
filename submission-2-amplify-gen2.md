=== BUIDL TITLE ===
AWS Amplify Gen 2 Full-Stack Deployment: Next.js + Cognito + AppSync Real-Time

=== DETAILS FIELD (the actual prompt) ===

You are an AWS solution architect helping me deploy a production-ready full-stack application using AWS Amplify Gen 2, the latest infrastructure-as-code framework for serverless web apps.

## OBJECTIVE
Deploy a complete serverless web application with authentication, real-time GraphQL API, and CI/CD using AWS Amplify Gen 2. This setup includes: Next.js 14 frontend, Amazon Cognito user authentication, AWS AppSync GraphQL API with real-time subscriptions, DynamoDB database, and automatic deployments from GitHub.

## ARCHITECTURE OVERVIEW

**Frontend**: Next.js 14 (App Router) hosted on Amplify Hosting
**Authentication**: Amazon Cognito User Pools with MFA support
**API**: AWS AppSync GraphQL with real-time subscriptions
**Database**: Amazon DynamoDB with single-table design
**Storage**: Amazon S3 for file uploads (optional)
**CI/CD**: Amplify Hosting with automatic deployments from GitHub
**Monitoring**: CloudWatch Logs + X-Ray tracing

## WHY AMPLIFY GEN 2?

Amplify Gen 2 (released 2024) replaces the CLI-based Gen 1 with a **TypeScript-first, file-based** approach. Key improvements:
- Define backend in `amplify/backend.ts` (type-safe)
- No more `amplify push` — infrastructure deploys automatically
- Better Git integration and multi-environment support
- Fully compatible with AWS CDK

## STEP-BY-STEP DEPLOYMENT

### 1. Prerequisites Check

Ensure you have:
```bash
node --version  # v18.17+ or v20+
npm --version   # v9+
git --version   # any recent version
```

Install Amplify CLI (Gen 2):
```bash
npm install -g @aws-amplify/backend-cli@latest
amplify --version  # Should show 2.x
```

**Decision Point**: Using GitHub or GitLab? This guide uses GitHub. For GitLab, replace GitHub-specific steps with GitLab CI/CD.

### 2. Create Next.js Project

```bash
npx create-next-app@latest my-saas-app \
  --typescript \
  --tailwind \
  --app \
  --no-src-dir \
  --import-alias "@/*"

cd my-saas-app
```

### 3. Initialize Amplify Gen 2 Backend

```bash
npm create amplify@latest
```

This creates:
- `amplify/backend.ts` (backend definition)
- `amplify/auth/resource.ts` (Cognito config)
- `amplify/data/resource.ts` (AppSync GraphQL schema)

### 4. Configure Authentication (Cognito)

Edit `amplify/auth/resource.ts`:

```typescript
import { defineAuth } from '@aws-amplify/backend';

export const auth = defineAuth({
  loginWith: {
    email: true,  // Email + password authentication
  },
  userAttributes: {
    email: {
      required: true,
      mutable: false,
    },
    givenName: {
      required: true,
      mutable: true,
    },
    familyName: {
      required: true,
      mutable: true,
    },
  },
  // MFA configuration
  multifactor: {
    mode: 'OPTIONAL',  // 'REQUIRED' for enterprise, 'OPTIONAL' for consumer
    totp: true,        // Time-based one-time password (Google Authenticator)
    sms: false,        // SMS MFA (costs $ per message)
  },
  // Password policy
  passwordPolicy: {
    minLength: 12,
    requireLowercase: true,
    requireUppercase: true,
    requireNumbers: true,
    requireSymbols: true,
  },
  // Account recovery
  accountRecovery: 'EMAIL_ONLY',  // or 'EMAIL_AND_PHONE'
});
```

**Security Note**: Never use `requireSymbols: false` in production. OWASP requires complex passwords.

### 5. Define GraphQL Schema with Real-Time Subscriptions

Edit `amplify/data/resource.ts`:

```typescript
import { type ClientSchema, a, defineData } from '@aws-amplify/backend';

const schema = a.schema({
  // Todo model with real-time subscriptions
  Todo: a
    .model({
      content: a.string().required(),
      completed: a.boolean().default(false),
      priority: a.enum(['low', 'medium', 'high']),
      ownerId: a.string().required(),  // User ID from Cognito
      createdAt: a.datetime(),
      updatedAt: a.datetime(),
    })
    .authorization(allow => [
      // Only the owner can read/update/delete their own todos
      allow.owner('ownerId'),
      // Admins can access all todos
      allow.groups(['Admins']),
    ]),

  // Custom query example
  getTodoStats: a
    .query()
    .arguments({
      userId: a.string().required(),
    })
    .returns(a.customType({
      total: a.integer(),
      completed: a.integer(),
      pending: a.integer(),
    }))
    .authorization(allow => [allow.authenticated()])
    .handler(a.handler.function('getTodoStatsFunction')),
});

export type Schema = ClientSchema<typeof schema>;

export const data = defineData({
  schema,
  authorizationModes: {
    defaultAuthorizationMode: 'userPool',  // Cognito User Pools
    apiKeyAuthorizationMode: {
      expiresInDays: 30,  // For public read-only access (optional)
    },
  },
});
```

**Authorization Strategy**:
- `allow.owner()` — User can only access their own data
- `allow.groups()` — Admin users can access all data
- `allow.authenticated()` — Any logged-in user

### 6. Create Lambda Resolver for Custom Query (Optional)

Create `amplify/data/getTodoStatsFunction.ts`:

```typescript
import type { Schema } from './resource';

export const handler: Schema['getTodoStats']['functionHandler'] = async (
  event
) => {
  const { userId } = event.arguments;

  // Query DynamoDB for user's todos
  const todos = await fetchTodosFromDynamoDB(userId);

  return {
    total: todos.length,
    completed: todos.filter(t => t.completed).length,
    pending: todos.filter(t => !t.completed).length,
  };
};

async function fetchTodosFromDynamoDB(userId: string) {
  // Implementation using AWS SDK v3
  // See AWS AppSync documentation for DynamoDB query patterns
  return [];
}
```

### 7. Configure Backend in `amplify/backend.ts`

```typescript
import { defineBackend } from '@aws-amplify/backend';
import { auth } from './auth/resource';
import { data } from './data/resource';

const backend = defineBackend({
  auth,
  data,
});

// Add custom CloudWatch alarms
backend.addOutput({
  custom: {
    apiEndpoint: backend.data.url,
    region: backend.data.stack.region,
  },
});
```

### 8. Deploy Backend to AWS

```bash
npx ampx sandbox  # Starts a local dev environment (fast iteration)
```

For production deployment:

```bash
npx ampx deploy --branch main
```

**Wait for deployment** (~5 minutes). You'll see:
```
 Auth deployed: User Pool ID: us-east-1_abc123
 Data deployed: GraphQL API: https://xxxxx.appsync-api.us-east-1.amazonaws.com/graphql
 DynamoDB table: Todo-main-xxxxx
```

### 9. Connect Frontend to Backend

Install Amplify libraries:

```bash
npm install aws-amplify @aws-amplify/ui-react
```

Create `app/amplify-client-config.ts`:

```typescript
import { Amplify } from 'aws-amplify';
import outputs from '../amplify_outputs.json';

Amplify.configure(outputs);
```

Create authentication wrapper `app/providers.tsx`:

```typescript
'use client';

import { Authenticator } from '@aws-amplify/ui-react';
import '@aws-amplify/ui-react/styles.css';
import './amplify-client-config';

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <Authenticator>
      {({ signOut, user }) => (
        <main>
          <h1>Hello {user?.username}</h1>
          <button onClick={signOut}>Sign out</button>
          {children}
        </main>
      )}
    </Authenticator>
  );
}
```

Update `app/layout.tsx`:

```typescript
import { Providers } from './providers';

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

### 10. Implement GraphQL Queries with Real-Time Subscriptions

Create `app/todos/page.tsx`:

```typescript
'use client';

import { useState, useEffect } from 'react';
import { generateClient } from 'aws-amplify/data';
import type { Schema } from '@/amplify/data/resource';

const client = generateClient<Schema>();

export default function TodosPage() {
  const [todos, setTodos] = useState<Schema['Todo'][]>([]);

  useEffect(() => {
    // Fetch initial todos
    client.models.Todo.list().then(({ data }) => setTodos(data));

    // Subscribe to real-time updates
    const subscription = client.models.Todo.onCreate().subscribe({
      next: (todo) => {
        setTodos((prev) => [...prev, todo]);
      },
    });

    return () => subscription.unsubscribe();
  }, []);

  const addTodo = async (content: string) => {
    await client.models.Todo.create({
      content,
      completed: false,
      ownerId: 'current-user-id',  // Get from Cognito
    });
  };

  return (
    <div>
      <h1>My Todos</h1>
      <ul>
        {todos.map((todo) => (
          <li key={todo.id}>{todo.content}</li>
        ))}
      </ul>
      <button onClick={() => addTodo('New todo')}>Add Todo</button>
    </div>
  );
}
```

**Real-Time Subscriptions**: Any create/update/delete operation on `Todo` model will automatically trigger subscriptions on all connected clients.

### 11. Set Up CI/CD with Amplify Hosting

Connect your GitHub repository:

```bash
npx ampx hosting add
```

Follow prompts:
1. Select "GitHub"
2. Authorize GitHub OAuth
3. Select repository: `your-org/my-saas-app`
4. Select branch: `main`
5. Build settings: Auto-detected (Next.js)

**Build Specification** (auto-generated `amplify.yml`):

```yaml
version: 1
backend:
  phases:
    build:
      commands:
        - npx ampx deploy --branch $AWS_BRANCH
frontend:
  phases:
    preBuild:
      commands:
        - npm ci
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: .next
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
      - .next/cache/**/*
```

**Deploy from GitHub**:

```bash
git add .
git commit -m "feat: initial Amplify Gen 2 setup"
git push origin main
```

Amplify Hosting will automatically:
1. Build frontend
2. Deploy backend
3. Provision CDN
4. Generate HTTPS certificate

**Deployment URL**: `https://main.xxxxxx.amplifyapp.com`

### 12. Configure Custom Domain (Optional)

```bash
aws amplify create-domain-association \
  --app-id d123abc \
  --domain-name myapp.com \
  --sub-domain-settings prefix=www,branchName=main \
  --enable-auto-sub-domain
```

**DNS Setup**: Add CNAME record:
```
www.myapp.com → main.xxxxxx.amplifyapp.com
```

### 13. Enable Monitoring and Alarms

CloudWatch Logs are automatically enabled. Create alarms:

```bash
# AppSync API error rate alarm
aws cloudwatch put-metric-alarm \
  --alarm-name amplify-appsync-errors \
  --metric-name 4XXError \
  --namespace AWS/AppSync \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=GraphQLAPIId,Value=xxxxx \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts

# Cognito user registration spike (potential abuse)
aws cloudwatch put-metric-alarm \
  --alarm-name amplify-cognito-spike \
  --metric-name UserAuthentication \
  --namespace AWS/Cognito \
  --statistic Sum \
  --period 60 \
  --threshold 1000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=UserPool,Value=us-east-1_abc123 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:security-alerts
```

### 14. Cost Controls

Set up budget:

```bash
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "Amplify-Monthly",
    "BudgetLimit": {"Amount": "100", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }'
```

**Estimated Monthly Cost**:
- Amplify Hosting: $15/month (includes 15 GB bandwidth)
- AppSync: $4/1M requests + $2/1M minutes (real-time)
- Cognito: Free for first 50K MAU, then $0.0055/MAU
- DynamoDB: $1.25/month (on-demand, < 10GB, < 1M writes)
- **Total for < 10K users**: $20-$50/month

### 15. Validation Steps

Test authentication:
```bash
# Open app in browser
open https://main.xxxxxx.amplifyapp.com

# Sign up with email
# Verify email (check inbox)
# Sign in
```

Test GraphQL API:
```bash
# Open AppSync console
aws appsync list-graphql-apis

# Test query
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {{ID_TOKEN}}" \
  -d '{"query":"{ listTodos { items { content } } }"}' \
  https://xxxxx.appsync-api.us-east-1.amazonaws.com/graphql
```

Test real-time subscriptions:
```bash
# Open two browser tabs
# Tab 1: Create todo → should appear in Tab 2 instantly
```

=== CONTEXT & DOCUMENTATION FIELD ===

**Prerequisites**:
- AWS account with billing enabled
- GitHub account (for CI/CD)
- Node.js 18.17+ or 20+
- AWS CLI v2 configured with AdministratorAccess (or PowerUserAccess + IAMFullAccess)
- Basic understanding of React, Next.js, and GraphQL

**Use Case**:
This prompt is for developers building:
- SaaS MVPs that need auth + real-time data
- Internal tools with user authentication
- Customer portals with personalized dashboards
- Chat applications or collaborative tools
- E-commerce storefronts with inventory sync

Typical customers: Startups shipping MVP in 2-4 weeks, solo founders, small dev teams (2-5 engineers).

**Expected Outcome**:
- Next.js 14 app deployed to Amplify Hosting with HTTPS
- Cognito User Pool with email auth + optional MFA
- AppSync GraphQL API with real-time subscriptions
- DynamoDB table for data storage
- Automatic CI/CD from GitHub main branch
- CloudWatch monitoring and alarms
- **Estimated cost**: $20-$50/month for < 10K users

**Troubleshooting Tips**:

1. **"Amplify push failed: No credentials found"**
   - Run `aws configure` and enter IAM user credentials
   - Ensure IAM user has `AdministratorAccess` or `Amplify*` + `CloudFormation*` permissions

2. **"GraphQL mutations fail with 'Unauthorized'"**
   - Check authorization rules in `amplify/data/resource.ts`
   - Verify user is signed in: `console.log(await fetchAuthSession())`
   - Ensure `defaultAuthorizationMode: 'userPool'` is set

3. **"Real-time subscriptions not working"**
   - Verify WebSocket connection in Network tab
   - Check AppSync API permissions (must allow subscriptions)
   - Ensure client is authenticated before subscribing

4. **"Build fails in Amplify Hosting"**
   - Check `amplify.yml` has correct Node version: `runtime: nodejs:20`
   - Verify all dependencies in `package.json`
   - Review build logs in Amplify Console → App Settings → Build settings

5. **"High Cognito costs (> $50/month)"**
   - Check Monthly Active Users (MAU) in Cognito metrics
   - Implement email verification to prevent fake signups
   - Consider switching to Cognito Identity Pools + passwordless auth

=== AWS SERVICES & BEST PRACTICES FIELD ===

**Services used**:
- AWS Amplify (hosting + CI/CD)
- AWS AppSync (GraphQL API)
- Amazon Cognito (authentication)
- Amazon DynamoDB (database)
- AWS Lambda (custom resolvers)
- Amazon CloudWatch (monitoring)
- AWS CloudFormation (infrastructure-as-code)
- Amazon S3 (static assets)
- Amazon CloudFront (CDN)
- AWS Certificate Manager (HTTPS)

**Well-Architected pillars addressed**:

1. **Security**:
   - Cognito User Pools with MFA support
   - Row-level authorization in GraphQL (owner-based access)
   - TLS 1.2+ for all API calls
   - Secrets managed via Amplify environment variables
   - HTTPS-only hosting with AWS Certificate Manager
   - CloudWatch Logs for audit trail

2. **Reliability**:
   - Serverless architecture (no servers to manage)
   - Multi-AZ DynamoDB replication
   - AppSync managed service (99.95% SLA)
   - CloudFront global CDN for low latency
   - Automatic failover for all AWS services

3. **Performance Efficiency**:
   - DynamoDB on-demand scaling
   - CloudFront edge caching
   - AppSync built-in caching
   - Next.js automatic code splitting
   - Real-time subscriptions (no polling required)

4. **Cost Optimization**:
   - Serverless pay-per-use pricing
   - DynamoDB on-demand (no provisioned capacity waste)
   - Amplify free tier (first 1000 build minutes free)
   - CloudFront free tier (1 TB/month)
   - Budget alerts at $100/month

5. **Operational Excellence**:
   - Infrastructure-as-code with Amplify Gen 2
   - Automatic deployments from GitHub
   - CloudWatch Logs for debugging
   - X-Ray tracing for AppSync
   - Environment-specific backends (dev/staging/prod)

**Security controls included**:
- Cognito User Pools with password policy (12+ chars, complex)
- MFA support (TOTP)
- AppSync authorization (user pools + API key)
- HTTPS-only hosting
- Secrets in environment variables (not hardcoded)
- CloudWatch Logs for security audit
- WAF integration (optional, via Amplify Hosting settings)

**Cost optimization measures**:
- DynamoDB on-demand (pay only for reads/writes)
- Serverless Lambda (no idle costs)
- CloudFront caching reduces AppSync requests
- Budget alerts prevent overruns
- Free tier usage: Cognito (50K MAU), AppSync (250K queries/month)
- Amplify Hosting (first 1000 build minutes free)

**Estimated Monthly Cost Breakdown**:
- Amplify Hosting: $15 (includes 15 GB transfer)
- AppSync: $4 (100K requests) + $2 (10K real-time minutes)
- Cognito: Free (< 50K MAU)
- DynamoDB: $1.25 (< 1M writes, < 10GB)
- Lambda: < $1 (< 1M invocations)
- **Total: $22-$50/month** for a typical SaaS MVP with < 10K users
