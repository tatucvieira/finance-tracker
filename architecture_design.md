# Technical Architecture Document: Finance Tracker

## 1. System Architecture Overview

### 1.1 High-Level Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                    Client (Web Browser)                      │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │   React     │ │   React     │ │     React Router    │   │
│  │ Components  │ │   Context   │ │                     │   │
│  │             │ │  (State)    │ │                     │   │
│  └─────────────┘ └─────────────┘ └─────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTPS (REST/GraphQL)
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway (NGINX)                      │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                 Backend Services (Node.js)                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │   Auth      │ │ Transaction │ │     Analytics       │   │
│  │  Service    │ │   Service   │ │     Service         │   │
│  └─────────────┘ └─────────────┘ └─────────────────────┘   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │  Budget     │ │   AI/ML     │ │  Notification       │   │
│  │  Service    │ │   Service   │ │   Service           │   │
│  └─────────────┘ └─────────────┘ └─────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                    Data Layer                               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │ PostgreSQL  │ │   Redis     │ │   S3/MinIO          │   │
│  │  (Primary)  │ │  (Cache)    │ │   (File Storage)    │   │
│  └─────────────┘ └─────────────┘ └─────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                    External Services                        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │    Plaid    │ │   SendGrid  │ │   Monitoring        │   │
│  │   (Banking) │ │  (Email)    │ │   (Datadog)         │   │
│  └─────────────┘ └─────────────┘ └─────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

**Frontend:**
- React 18+ with TypeScript
- Tailwind CSS for styling
- Recharts/Chart.js for data visualization
- React Query for server state management
- React Hook Form for forms
- Axios for HTTP client

**Backend:**
- Node.js 20+ with TypeScript
- Express.js framework
- PostgreSQL 15+ with Prisma ORM
- Redis 7+ for caching and session management
- JWT for authentication
- BullMQ for job queues

**AI/ML Services:**
- Python 3.11+ with FastAPI
- Scikit-learn for ML models
- spaCy for NLP
- Redis for model caching
- Docker containerization

**Infrastructure:**
- AWS/Azure/GCP (cloud provider)
- Docker & Docker Compose
- Kubernetes (production)
- GitHub Actions for CI/CD
- Terraform for IaC

**Monitoring & Analytics:**
- Datadog/New Relic for APM
- Sentry for error tracking
- LogDNA/Papertrail for logging

## 2. Data Models

### 2.1 Core Database Schema

```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email_verified BOOLEAN DEFAULT FALSE,
    verification_token VARCHAR(255),
    reset_token VARCHAR(255),
    reset_token_expires TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP
);

-- Financial institutions table
CREATE TABLE financial_institutions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plaid_institution_id VARCHAR(100) UNIQUE,
    name VARCHAR(255) NOT NULL,
    logo_url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User linked accounts
CREATE TABLE linked_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    institution_id UUID REFERENCES financial_institutions(id),
    plaid_access_token TEXT NOT NULL,
    plaid_item_id VARCHAR(100) UNIQUE NOT NULL,
    account_name VARCHAR(255) NOT NULL,
    account_type VARCHAR(50) NOT NULL, -- checking, savings, credit, investment
    account_subtype VARCHAR(50),
    mask VARCHAR(10),
    official_name TEXT,
    balance_current DECIMAL(15,2),
    balance_available DECIMAL(15,2),
    balance_limit DECIMAL(15,2),
    iso_currency_code VARCHAR(3) DEFAULT 'USD',
    last_synced_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Transaction categories
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    icon VARCHAR(50),
    color VARCHAR(7),
    is_system BOOLEAN DEFAULT TRUE,
    parent_category_id UUID REFERENCES categories(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Default system categories
INSERT INTO categories (name, icon, color, is_system) VALUES
('Food & Dining', 'utensils', '#FF6B6B', TRUE),
('Transportation', 'car', '#4ECDC4', TRUE),
('Shopping', 'shopping-bag', '#45B7D1', TRUE),
('Entertainment', 'film', '#96CEB4', TRUE),
('Bills & Utilities', 'file-invoice-dollar', '#FFEAA7', TRUE),
('Healthcare', 'heart-pulse', '#DDA0DD', TRUE),
('Education', 'graduation-cap', '#98D8C8', TRUE),
('Personal Care', 'spa', '#F7DC6F', TRUE),
('Travel', 'plane', '#BB8FCE', TRUE),
('Gifts & Donations', 'gift', '#82E0AA', TRUE),
('Income', 'money-bill-wave', '#2ECC71', TRUE),
('Transfer', 'exchange-alt', '#3498DB', TRUE),
('Uncategorized', 'question-circle', '#95A5A6', TRUE);

-- Transactions table
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    account_id UUID REFERENCES linked_accounts(id),
    plaid_transaction_id VARCHAR(100) UNIQUE,
    name VARCHAR(255) NOT NULL,
    merchant_name VARCHAR(255),
    amount DECIMAL(15,2) NOT NULL,
    iso_currency_code VARCHAR(3) DEFAULT 'USD',
    date DATE NOT NULL,
    pending BOOLEAN DEFAULT FALSE,
    category_id UUID REFERENCES categories(id),
    user_category_id UUID REFERENCES categories(id),
    payment_channel VARCHAR(50),
    location JSONB,
    notes TEXT,
    is_manual BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Category rules for AI learning
CREATE TABLE category_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    pattern_type VARCHAR(20) NOT NULL, -- merchant, keyword, amount_range
    pattern_value TEXT NOT NULL,
    category_id UUID REFERENCES categories(id) NOT NULL,
    priority INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Budgets table
CREATE TABLE budgets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    period VARCHAR(20) NOT NULL, -- monthly, weekly, yearly
    start_date DATE NOT NULL,
    end_date DATE,
    total_limit DECIMAL(15,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Budget categories
CREATE TABLE budget_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_id UUID REFERENCES budgets(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id),
    limit_amount DECIMAL(15,2) NOT NULL,
    current_spent DECIMAL(15,2) DEFAULT 0,
    alert_threshold INTEGER DEFAULT 80, -- percentage
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Savings goals
CREATE TABLE savings_goals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    target_amount DECIMAL(15,2) NOT NULL,
    current_amount DECIMAL(15,2) DEFAULT 0,
    target_date DATE,
    monthly_contribution DECIMAL(15,2),
    icon VARCHAR(50),
    color VARCHAR(7),
    is_completed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- AI insights
CREATE TABLE insights (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    insight_type VARCHAR(50) NOT NULL, -- spending_spike, savings_opportunity, budget_alert
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    data JSONB, -- supporting data for the insight
    is_read BOOLEAN DEFAULT FALSE,
    generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP
);

-- User preferences
CREATE TABLE user_preferences (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    currency VARCHAR(3) DEFAULT 'USD',
    timezone VARCHAR(50) DEFAULT 'UTC',
    notification_settings JSONB DEFAULT '{
        "email": true,
        "push": true,
        "budget_alerts": true,
        "weekly_summary": true
    }',
    dashboard_layout JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2.2 Redis Data Structures
- `session:{sessionId}` - User session data
- `user:{userId}:tokens` - JWT refresh tokens
- `cache:transactions:{userId}:{filters}` - Cached transaction queries
- `cache:analytics:{userId}:{period}` - Cached analytics data
- `queue:plaid_sync` - Background job queue for Plaid sync
- `queue:ai_processing` - AI categorization and insight generation queue

## 3. File Structure

```
finance-tracker/
├── client/
│   ├── public/
│   │   ├── index.html
│   │   ├── favicon.ico
│   │   └── manifest.json
│   ├── src/
│   │   ├── components/
│   │   │   ├── common/
│   │   │   │   ├── Button/
│   │   │   │   ├── Input/
│   │   │   │   ├── Card/
│   │   │   │   ├── Modal/
│   │   │   │   ├── LoadingSpinner/
│   │   │   │   └── ErrorBoundary/
│   │   │   ├── layout/
│   │   │   │   ├── Header/
│   │   │   │   ├── Sidebar/
│   │   │   │   ├── Footer/
│   │   │   │   └── Layout.tsx
│   │   │   ├── dashboard/
│   │   │   │   ├── Overview/
│   │   │   │   ├── Transactions/
│   │   │   │   ├── Analytics/
│   │   │   │   ├── Budget/
│   │   │   │   └── Goals/
│   │   │   ├── auth/
│   │   │   │   ├── Login/
│   │   │   │   ├── Register/
│   │   │   │   └── ForgotPassword/
│   │   │   └── onboarding/
│   │   │       ├── ConnectAccounts/
│   │   │       └── SetupBudget/
│   │   ├── contexts/
│   │   │   ├── AuthContext.tsx
│   │   │   ├── ThemeContext.tsx
│   │   │   └── NotificationContext.tsx
│   │   ├── hooks/
│   │   │   ├── useAuth.ts
│   │   │   ├── useTransactions.ts
│   │   │   ├── useBudget.ts
│   │   │   └── useAnalytics.ts
│   │   ├── services/
│   │   │   ├── api/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── transactions.ts
│   │   │   │   ├── accounts.ts
│   │   │   │   ├── budget.ts
│   │   │   │   └── goals.ts
│   │   │   └── plaid/
│   │   │       ├── link.ts
│   │   │       └── webhooks.ts
│   │   ├── utils/
│   │   │   ├── formatters.ts
│   │   │   ├── validators.ts
│   │   │   ├── constants.ts
│   │   │   └── helpers.ts
│   │   ├── types/
│   │   │   ├── user.ts
│   │   │   ├── transaction.ts
│   │   │   ├── budget.ts
│   │   │   └── api.ts
│   │   ├── styles/
│   │   │   ├── globals.css
│   │   │   └── tailwind.config.js
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   └── routes.tsx
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   └── .env.example
│
├── server/
│   ├── src/
│   │   ├── config/
│   │   │   ├── database.ts
│   │   │   ├── redis.ts
│   │   │   ├── plaid.ts
│   │   │   └── auth.ts
│   │   ├── middleware/
│   │   │   ├── auth.ts
│   │   │   ├── validation.ts
│   │   │   ├── errorHandler.ts
│   │   │   └── rateLimiter.ts
│   │   ├── controllers/
│   │   │   ├── auth.controller.ts
│   │   │   ├── accounts.controller.ts
│   │   │   ├── transactions.controller.ts
│   │   │   ├── categories.controller.ts
│   │   │   ├── budget.controller.ts
│   │   │   ├── goals.controller.ts
│   │   │   └── insights.controller.ts
│   │   ├── services/
│   │   │   ├── auth.service.ts
│   │   │   ├── plaid.service.ts
│   │   │   ├── transaction.service.ts
│   │   │   ├── categorization.service.ts
│   │   │   ├── budget.service.ts
│   │   │   ├── analytics.service.ts
│   │   │   └── notification.service.ts
│   │   ├── jobs/
│   │   │   ├── plaidSync.job.ts
│   │   │   ├── categorization.job.ts
│   │   │   ├── insightGeneration.job.ts
│   │   │   └── budgetAlerts.job.ts
│   │   ├── models/
│   │   │   ├── user.model.ts
│   │   │   ├── transaction.model.ts
│   │   │   ├── account.model.ts
│   │   │   ├── budget.model.ts
│   │   │   └── goal.model.ts
│   │   ├── routes/
│   │   │   ├── auth.routes.ts
│   │   │   ├── accounts.routes.ts
│   │   │   ├── transactions.routes.ts
│   │   │   ├── categories.routes.ts
│   │   │   ├── budget.routes.ts
│   │   │   ├── goals.routes.ts
│   │   │   └── insights.routes.ts
│   │   ├── utils/
│   │   │   ├── logger.ts
│   │   │   ├── validators.ts
│   │   │   ├── encryption.ts
│   │   │   └── helpers.ts
│   │   ├── app.ts
│   │   └── server.ts
│   ├── prisma/
│   │   ├── schema.prisma
│   │   └── migrations/
│   ├── package.json
│   ├── tsconfig.json
│   └── .env.example
│
├── ai-service/
│   ├── src/
│   │   ├── models/
│   │   │   ├