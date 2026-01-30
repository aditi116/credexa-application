# Credexa Banking Platform - Project Report

## Executive Summary

Credexa is an enterprise-grade Fixed Deposit (FD) banking platform architected as a distributed microservices ecosystem. The system enables complete FD lifecycle management from customer onboarding through account maturity, implementing core banking operations with regulatory compliance, cryptographic security, and financial precision. Built on Spring Boot 3.5.6 and Java 17, the platform demonstrates production-ready capabilities including JWT authentication, compound interest calculations, IBAN generation, and event-driven architecture.

---

## System Architecture

### Microservices Overview

The platform comprises seven independent services communicating through RESTful APIs, orchestrated by Spring Cloud Gateway:

| Service | Port | Database | Purpose |
|---------|------|----------|---------|
| **API Gateway** | 8080 | - | Request routing, CORS, unified entry point |
| **Login Service** | 8081 | login_db | Authentication, authorization, user management |
| **Customer Service** | 8082 | customer_db | Customer lifecycle, KYC compliance |
| **Product-Pricing Service** | 8084 | product_db | Product catalog, interest rate management |
| **FD Calculator Service** | 8085 | calculator_db | Interest calculations, maturity projections |
| **Account Service** | 8087 | account_db | FD account operations, transaction processing |
| **Frontend UI** | 5173 | - | React-based customer interface |

---

## Technology Stack

### Backend Technologies
- **Runtime**: Java 17 (LTS), Spring Boot 3.5.6
- **Security**: Spring Security, JWT (JJWT 0.12.6), BCrypt password hashing
- **Data Layer**: Spring Data JPA, Hibernate ORM, MySQL 8.0
- **Messaging**: Apache Kafka, Spring Kafka
- **API Gateway**: Spring Cloud Gateway
- **Documentation**: Swagger/OpenAPI 3.0, Springdoc-OpenAPI 2.7.0
- **Build Tool**: Maven 3.6+ (multi-module project)
- **Caching**: Caffeine (24-hour TTL for reference data)

### Frontend Technologies
- **Framework**: React 19, TypeScript
- **Build Tool**: Vite 7.1.7
- **UI Library**: Radix UI components
- **Styling**: TailwindCSS 4.1
- **HTTP Client**: Axios 1.13
- **Routing**: React Router DOM 7.9
- **Visualization**: Recharts 3.3

---

## Core Banking Features Implemented

### 1. Authentication & Authorization (Login Service)

**JWT Token Management**
- Token generation with 1-hour expiration (configurable)
- HMAC-SHA256 signing algorithm with 256-bit secret key
- Refresh token mechanism for session continuity
- Token validation across all microservices via common-lib

**Account Security**
- BCrypt password hashing (10 rounds, cryptographic salt)
- Account lockout after 5 consecutive failed login attempts
- Auto-logout after idle timeout period
- OAuth 2.0 integration (Google authentication provider)

**Role-Based Access Control (RBAC)**
- Three authorization levels: ADMIN, MANAGER, USER
- Method-level security annotations (@PreAuthorize)
- Hierarchical permission model

### 2. Customer Management (Customer Service)

**KYC (Know Your Customer) Compliance**
- PAN (Permanent Account Number) validation and storage
- Aadhar (Unique Identification) number verification
- PII (Personally Identifiable Information) masking in logs
- Address proof documentation

**Customer Classification**
- Retail customers (standard rate structure)
- HNI (High Net-Worth Individual) customers (preferential rates)
- Corporate customers (bulk deposit handling)

**Customer 360° View**
- Complete profile information (demographics, contact details)
- Linked FD accounts with balances
- Transaction history across all accounts
- Nominee information for succession planning

### 3. Product & Pricing Management (Product-Pricing Service)

**Product Catalog**
- Multiple FD product variants (Regular FD, Tax Saver FD, Flexi FD)
- Tenure-based rate brackets (3M, 6M, 12M, 24M, 36M, 60M)
- Minimum/maximum investment constraints per product
- Senior citizen premium rates (typically +0.5% to +0.75%)

**Interest Rate Configuration**
- Base rate definition per product and tenure
- Rate slabs based on deposit amount thresholds
- Special rates for senior citizens (age 60+ years)
- Effective date management for rate changes

**TDS (Tax Deducted at Source) Rules**
- TDS applicability thresholds (₹40,000 for regular, ₹50,000 for senior citizens)
- Configurable TDS rates (10% standard, 20% for no-PAN cases)
- Form 15G/15H exemption handling

### 4. Fixed Deposit Calculator (FD Calculator Service)

**Interest Calculation Engine**
- **Simple Interest**: I = P × R × T / 100
- **Compound Interest**: A = P(1 + r/n)^(nt)
  - Monthly compounding (n=12)
  - Quarterly compounding (n=4)
  - Annual compounding (n=1)

**Financial Precision**
- BigDecimal arithmetic (no floating-point errors)
- RoundingMode.HALF_UP for currency rounding
- 2-decimal precision for INR currency

**Advanced Features**
- Effective Annual Rate (EAR) computation for product comparison
- Maturity amount projection with interest breakdown
- Month-wise and year-wise interest accumulation schedules
- Redemption penalty calculations for premature withdrawal

**Performance Optimization**
- Caffeine cache for product data (24-hour TTL)
- Cache invalidation on rate updates
- WebClient for non-blocking service integration

### 5. Account Management (Account Service)

**Account Creation - Dual Mode**

*Standard Mode (Default Values)*
- All parameters inherited from product configuration
- Interest rate: Product base rate
- Compounding: Product default frequency
- TDS rules: Product-linked TDS configuration

*Custom Mode (Within Product Purview)*
- Custom interest rate: ±2% variance from base rate
- Custom compounding frequency selection
- Validation against product constraints
- Automatic tagging: "[Custom Rate Applied]" in remarks

**Account Number Generation**
- Format: `FD-YYYYMMDDHHMMSS-NNNN-C`
- Timestamp-based uniqueness (millisecond precision)
- Luhn check digit algorithm for validation
- Random 4-digit suffix for collision prevention

**IBAN Generation**
- Format: `IN<check><bank><branch><account>`
- Bank code: CRED (Credexa identifier)
- Branch code: 0001 (default branch)
- Mod 97 check digit algorithm (ISO 13616 compliance)
- Stored with unique constraint in database

**Account Inquiry - Flexible Lookup**
- By account number (primary identifier)
- By IBAN (international standard)
- By internal ID (database primary key)
- Single unified endpoint for all lookup types

**Account Balance Calculation**
- Principal amount (original deposit)
- Accrued interest (computed to date)
- TDS amount (calculated on interest)
- Net maturity amount (Principal + Interest - TDS)
- Days to maturity (countdown from current date)

**Account Status Management**
- ACTIVE: Account operational, interest accruing
- MATURED: Tenure completed, withdrawal permitted
- CLOSED: Account settled, funds disbursed
- SUSPENDED: Regulatory hold or compliance issues

**Denormalization Strategy**
- Customer snapshot stored in account entity
- Product details frozen at account creation
- Historical accuracy preserved (rate changes don't affect existing accounts)
- Performance optimization (reduced cross-service calls)

### 6. Transaction Processing

**Transaction Types**
- CREDIT: Deposit/principal booking
- INTEREST_CREDIT: Periodic interest posting
- TDS_DEBIT: Tax deduction at source
- PENALTY: Premature withdrawal penalty
- MATURITY_CREDIT: Final maturity amount
- REVERSAL: Transaction correction/cancellation

**Redemption (Premature Withdrawal)**
- Penalty calculation: Penalty Rate × Interest Earned
- Reduced interest rate: Original Rate - Penalty Rate
- Tenure proportional calculation
- Net redemption amount: Principal + Reduced Interest - TDS

**Audit Trail**
- Transaction reference (UUID-based)
- Transaction timestamp (millisecond precision)
- Performed by user (audit user tracking)
- Balance after transaction (snapshot)
- Remarks (narrative description)

### 7. API Gateway (Spring Cloud Gateway)

**Routing Configuration**
- Path-based routing to backend services
- StripPrefix filter for context path handling
- PreserveHostHeader for service discovery

**Security**
- JWT validation at gateway level (optional per route)
- Token forwarding to downstream services
- Public endpoint exclusions (health, swagger-ui)

**CORS (Cross-Origin Resource Sharing)**
- Allowed origins: localhost:3000, localhost:5173
- Allowed methods: GET, POST, PUT, DELETE, PATCH, OPTIONS
- Allowed headers: * (all headers)
- Exposed headers: Authorization
- Credentials support: true
- MaxAge: 3600 seconds

### 8. Frontend Application

**Customer Interface**
- Login/Registration with OAuth support
- Customer dashboard (account summary, analytics)
- FD calculator (maturity projections, product comparison)
- Account creation forms (standard and custom modes)
- Account inquiry and balance viewing
- Transaction history with filters

**Administrative Interface**
- Customer management (create, update, view)
- Product configuration (rates, terms, constraints)
- Account approval workflows (custom rate approvals)
- Report generation and analytics

**UI Components**
- Radix UI primitives (dialogs, dropdowns, tabs)
- Form validation with error messages
- Responsive tables with pagination
- Charts and visualizations (Recharts)
- Toast notifications (Sonner)

---

## Banking Domain Concepts

### Fixed Deposit (FD)
A term deposit investment where customers lock principal for a fixed tenure at a predetermined interest rate. At maturity, customers receive principal plus accrued interest minus TDS.

### Compounding Frequency
The frequency at which interest is calculated and added to principal:
- **Monthly**: Interest compounds 12 times annually (highest EAR)
- **Quarterly**: Interest compounds 4 times annually
- **Annual**: Interest compounds once annually (lowest EAR)

### Effective Annual Rate (EAR)
The actual annual return considering compounding effects, calculated as:
EAR = (1 + r/n)^n - 1, where r = nominal rate, n = compounding frequency

### Tax Deducted at Source (TDS)
Advance tax deduction on interest income exceeding specified thresholds:
- ₹40,000 for regular customers
- ₹50,000 for senior citizens (60+ years)
- 10% standard rate, 20% for PAN non-submission

### Premature Withdrawal/Redemption
Early closure of FD before maturity date, resulting in:
- Reduced interest rate (penalty applied)
- Penalty amount = (Original Rate - Reduced Rate) × Interest Earned
- TDS applicable on reduced interest

### Senior Citizen Concession
Additional interest premium (typically +0.5% to +0.75%) offered to customers aged 60 years and above, recognizing their fixed-income dependency.

### IBAN (International Bank Account Number)
ISO 13616 standardized account identifier enabling:
- Cross-border fund transfers
- Automated payment validation
- International regulatory compliance
- Check digit verification (Mod 97 algorithm)

### KYC (Know Your Customer)
Regulatory due diligence process verifying customer identity through:
- Government-issued ID (PAN, Aadhar)
- Address proof (utility bills, bank statements)
- Photograph and signature
- Ongoing monitoring for AML (Anti-Money Laundering)

### Account Classification
- **Retail**: Individual customers with standard deposits
- **HNI (High Net-Worth Individual)**: ₹1 crore+ relationships, preferential rates
- **Corporate**: Business entities with bulk deposits

---

## Integration Architecture

### Service Communication Patterns

**Synchronous Integration (REST)**
- FD Calculator → Product-Pricing (product details, interest rates)
- FD Calculator → Customer Service (customer classification)
- Account Service → Customer Service (customer verification)
- Account Service → Product Service (product validation)
- Account Service → Calculator Service (maturity computation)

**Asynchronous Integration (Kafka)**
- Account creation events → Notification Service (email/SMS alerts)
- Transaction events → Audit Service (compliance logging)
- Customer updates → Account Service (profile synchronization)

**Common Library (common-lib)**
- JWT utilities (token generation, validation, parsing)
- DTOs (Data Transfer Objects) for inter-service communication
- Encryption utilities for sensitive data
- PII masking utilities for logging compliance
- Custom exceptions and error responses

### Data Consistency Strategy

**Cache-Aside Pattern**
- Product data cached for 24 hours (low volatility)
- Interest rates cached with TTL (daily refresh)
- Customer classification cached (periodic sync)

**Eventual Consistency**
- Kafka events ensure cross-service synchronization
- Retry mechanisms with exponential backoff
- Idempotency keys for duplicate prevention

**Historical Accuracy**
- Account entities store product/customer snapshots
- Rate changes don't affect existing accounts (frozen rates)
- Audit trail maintains complete transaction history

---

## Key Achievements

✅ **Production-Grade Architecture**: 7 microservices with independent scalability, resilience, and deployment
✅ **Financial Precision**: BigDecimal arithmetic eliminating floating-point errors in currency calculations
✅ **Enterprise Security**: JWT authentication, BCrypt hashing, RBAC, account lockout, audit logging
✅ **Regulatory Compliance**: KYC workflows, TDS calculations, PII masking, IBAN standards
✅ **Performance Optimization**: Caching (24-hour TTL), denormalization, database indexing, query optimization
✅ **Banking Accuracy**: Compound interest with multiple frequencies, Luhn/Mod97 algorithms, EAR computation
✅ **Modern Frontend**: React 19, TypeScript, responsive design, component-based architecture
✅ **API Documentation**: Swagger UI for all services, interactive testing, schema validation
✅ **Event-Driven**: Kafka integration for notifications, audit trails, asynchronous processing
✅ **Multi-Currency Ready**: Configurable decimal places, localization support, internationalization framework

---

## Project Statistics

- **Total Services**: 7 (6 backend + 1 gateway)
- **Lines of Code**: ~15,000+ (Java backend)
- **Frontend Components**: 50+ React components
- **Database Tables**: 25+ across 6 databases
- **API Endpoints**: 80+ RESTful endpoints
- **Test Coverage**: Unit tests, integration tests, API tests
- **Documentation**: Comprehensive Swagger/OpenAPI specs

---

## Deployment Architecture

**Local Development**
- Individual service startup scripts (`.bat` files)
- `start-all-services.bat` for orchestrated startup
- `stop-all-services.bat` for graceful shutdown
- Docker-ready containerization setup

**Database Setup**
- MySQL 8.0 with auto-creation enabled
- Hibernate DDL auto-update for schema evolution
- Separate database per microservice (data isolation)

**Service Dependencies**
- Start sequence: Gateway → Login → Customer → Product → Calculator → Account
- Health check endpoints: `/actuator/health`
- Dependency verification scripts

---

## Conclusion

Credexa represents a comprehensive Fixed Deposit banking platform demonstrating microservices best practices, financial domain expertise, and production-ready implementation. The system successfully implements complex banking operations including compound interest calculations, TDS compliance, IBAN generation, and secure authentication, while maintaining scalability, maintainability, and regulatory adherence. The platform is ready for production deployment with enterprise-grade security, performance optimization, and complete API documentation.
