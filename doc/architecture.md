# CoreTrades Conceptual Architecture and Development Structure

## Overview
CoreTrades is a holistic trading platform composed of Spring Boot microservices and a React Portal frontend. The platform is designed to support high availability and horizontal scaling on Kubernetes with Istio service mesh. CoreTrades comprises multiple application namespaces (each namespace is a separate Spring Boot project) that evolve independently while sharing centralized cross-cutting concerns.

The architecture applies proven microservices design patterns to mitigate distributed-system challenges:
- Service Discovery (Eureka)
- Edge Server / API Gateway
- Centralized Security (Authentication & Authorization) with Social Login extensions (Google, Facebook, LinkedIn, Microsoft, Apple)
- Central Configuration (Config Server)
- Reactive microservices where appropriate
- Centralized log aggregation and analysis
- Distributed tracing
- Circuit breaker and resilience patterns
- Control Loop / operational automation
- Centralized monitoring and alarms

## Key Roles
- **Admin (System Administrator)**: Root access; can create users, roles, and investing groups; manages all other roles.
- **Money Manager**: Represents institutions/partners creating investment products (Mutual Funds, ETFs, sector ETFs, portfolios across asset classes). Can create and manage portfolios for Stocks, Futures, Options, Bonds, ETFs, MFs, and Cryptos.
- **User**: Standard trading platform user. Manages own Savings Account (initial balance $100,000) and can withdraw cash to purchase products.
- **Database Schema Roles**: `Stocks_DB`, `Futures_DB`, `Options_DB`, `ETFs_DB`, `Bonds_DB`, `MFs_DB`, `Cryptos_DB` — each role owns its schema.
- **Group Role**: Technical role that can perform CRUD on database tables.

### Money Manager Attributes
- manager_id (unique identifier for each manager)
- first_name
- last_name
- email (unique contact for formal communication)
- phone_number
- company_id (reference to organization/company)
- specialization (e.g., Equities, Fixed Income, ESG)
- years_experience
- certification_status (e.g., CFA, CFP)
- status (active/inactive)
- created_at
- updated_at

### Group Table Attributes
- id
- group_name
- user_id
- name
- type
- symbol
- quantity
- purchase_date
- purchase_price_per_unit
- total_cost_basis
- current_price_per_unit
- market_value
- gain_loss
- currency

### User Profile (1:1 with User)
- user_id
- first_name
- last_name
- created_at
- updated_at
- picture
- phone_number
- timezone
- locale
- address
- city
- state_or_province
- zip_code
- country
- custom fields

## Domains and Microservices
The backend uses Spring Boot microservices only. The system consists of 10 major domains, each containing seven or more microservices. The first domain to implement is the **Tech Domain**.

### CoreTrades Application Namespaces (separate Spring Boot projects)

#### coretrades-tech/ (Tech / Platform domain)
- api-gateway-service
- security-service
- bff-web-portal-service
- bff-mobile-service
- config-server-service
- kafka-admin-service
- file-media-service
- reporting-etl-service
- batch-scheduler-service
- search-audit-service
- metrics-service
- tracing-service
- logging-agg-service
- slo-alerts-service
- audit-trails-service
- k8s-health-service
- notification-service

#### coretrades-core/ (Core financial backbone)
- trading-core-service
- account-ledger-service
- limits-service
- margin-service
- exposure-service
- corporate-actions-service

#### coretrades-shared/ (Shared / cross-asset)
- investment-product-service
- group-management-service
- pricing-agg-service
- portfolio-agg-service
- risk-agg-service
- settlement-service
- fees-service

#### coretrades-stocks/ (Stocks domain)
- stocks-order-service
- stocks-pricing-service
- stocks-position-service
- stocks-pnl-projector
- stocks-risk-service
- stocks-portfolio-service
- stocks-corp-actions-service
- stocks-refdata-service

#### coretrades-futures/ (Futures domain)
- futures-order-service
- futures-pricing-service
- futures-position-service
- futures-pnl-projector
- futures-risk-service
- futures-portfolio-service
- futures-contract-meta-service

#### coretrades-options/ (Options domain)
- options-order-service
- options-pricing-service
- options-position-service
- options-pnl-projector
- options-risk-service
- options-portfolio-service
- options-chain-service
- options-refdata-service

#### coretrades-etfs/ (ETFs domain)
- etfs-order-service
- etfs-pricing-service
- etfs-position-service
- etfs-pnl-projector
- etfs-risk-service
- etfs-portfolio-service

#### coretrades-bonds/ (Bonds domain)
- bonds-order-service
- bonds-pricing-service
- bonds-position-service
- bonds-pnl-projector
- bonds-risk-service
- bonds-portfolio-service

#### coretrades-mfs/ (Mutual Funds domain)
- mfs-order-service
- mfs-pricing-service
- mfs-position-service
- mfs-pnl-projector
- mfs-portfolio-service

#### coretrades-cryptos/ (Cryptos domain)
- cryptos-order-service
- cryptos-pricing-service
- cryptos-position-service
- cryptos-pnl-projector
- cryptos-risk-service
- cryptos-wallet-service
- cryptos-portfolio-service

## Conceptual Integration Architecture (ASCII)

```
+-----------------------------------------------------------------------------------+
|                                  React Portal (UI)                                |
|  Tabs: Home | Stocks | Futures | Options | ETFs | Bonds | MFs | Cryptos | Tech*    |
|  *Tech tab visible only for Admin                                                 |
+------------------------------------+----------------------------------------------+
                                     |
                                     v
+-----------------------------------------------------------------------------------+
|                           BFF Layer (Spring Boot Facades)                          |
|                 bff-web-portal-service            bff-mobile-service               |
|         (aggregation, caching, UI-friendly APIs, auth propagation)                 |
+------------------------------------+----------------------------------------------+
                                     |
                                     v
+-----------------------------------------------------------------------------------+
|                            Edge / Ingress + API Gateway                            |
|                        Istio Ingress Gateway / API Gateway Service                 |
|                (routing, rate limits, auth filters, WAF-like controls)             |
+------------------------------------+----------------------------------------------+
                                     |
                                     v
+-----------------------------------------------------------------------------------+
|                                Service Mesh (Istio)                                |
| mTLS | retries | timeouts | circuit breakers | traffic shaping | policy enforcement |
+------------------------------------+----------------------------------------------+
                                     |
                                     v
+---------------------------+------------------+------------------+------------------+
|       Tech Domain         |   Core Domain    |  Shared Domain   | Business Domains |
| (cross-cutting platform)  | (financial core) | (cross-asset)    | (asset-specific) |
+---------------------------+------------------+------------------+------------------+
| - security-service        | - trading-core   | - investment-prod| - coretrades-    |
| - config-server-service   | - account-ledger | - group-mgmt     |   stocks/*       |
| - kafka-admin-service     | - limits/margin  | - pricing-agg    | - coretrades-    |
| - logging-agg-service     | - exposure       | - portfolio-agg  |   futures/*      |
| - tracing-service         | - corp-actions   | - risk-agg       | - coretrades-    |
| - metrics-service         |                  | - settlement     |   options/*      |
| - search-audit-service    |                  | - fees           | - coretrades-    |
| - notification-service    |                  |                  |   etfs/*         |
| - reporting-etl/batch     |                  |                  | - coretrades-    |
| - file-media-service      |                  |                  |   bonds/*        |
| - slo-alerts/audit-trails |                  |                  | - coretrades-    |
| - k8s-health-service      |                  |                  |   mfs/*          |
|                           |                  |                  | - coretrades-    |
|                           |                  |                  |   cryptos/*      |
+---------------------------+------------------+------------------+------------------+
                                     |
                                     v
+-----------------------------------------------------------------------------------+
|                             Data + Eventing + Observability                        |
| PostgreSQL per domain/schema | Redis | Kafka | OpenSearch | Prometheus/Grafana     |
| OTel Collector + Tempo/Jaeger | Fluent Bit -> OpenSearch | Immutable audit index   |
+-----------------------------------------------------------------------------------+

Legend:
- Requests flow: React -> BFF -> API Gateway -> Istio Mesh -> domain services
- Cross-cutting: security, config, logs, traces, metrics, audit integrated across all domains
- Eureka supports service discovery for microservices integration within the platform
```

## Spring Boot Best-Practice Development File Structure (per microservice)
Each microservice is an independent Spring Boot application, packaged and deployed separately.

Recommended structure (example for any service, e.g., `stocks-order-service`):

```
<service-root>/
  src/
    main/
      java/
        com/coretrades/<domain>/<service>/
          <ServiceApplication>.java
          config/
          controller/
          dto/
          entity/
          event/
          exception/
          mapper/
          repository/
          security/
          service/
          validation/
          web/
      resources/
        application.yml
        bootstrap.yml
        db/migration/               # Flyway
        static/
        templates/
        logback-spring.xml
    test/
      java/
        com/coretrades/<domain>/<service>/
          ...
      resources/
        application-test.yml
  Dockerfile
  pom.xml
  README.md
```

Notes:
- Prefer clean layering: controller -> service -> repository.
- Use DTOs and mappers at boundaries.
- Use Flyway for schema migrations.
- Use Spring Actuator for health probes and metrics.

## React Portal Frontend Concept
A single React Portal frontend provides a tabbed navigation UI:
- Home
- Stocks Trading
- Futures Trading
- Options Trading
- ETFs Trading
- Bonds Trading
- Mutual Funds
- Cryptos
- Tech Domain (Admin-only)

Tech Domain tab contains sub-tabs/pages for platform administration (e.g., user/role management, monitoring links, audit search, config view, etc.).

Recommended frontend structure:

```
react-portal/
  src/
    app/
      routes/
      layout/
      auth/
    features/
      home/
      tech/
      stocks/
      futures/
      options/
      etfs/
      bonds/
      mfs/
      cryptos/
    components/
      tabs/
      nav/
      forms/
      tables/
    api/
      httpClient.ts
      auth.ts
      coretradesApi.ts
    styles/
    main.tsx
    App.tsx
```

Security:
- Integrate login/registration UI with the `security-service` (via BFF).
- Use role-based UI gating (Admin sees Tech tab).

## Business Function Summary
- Users register and authenticate via central Security Service.
- Users manage Savings Account (starting at $100,000) and place orders across asset classes.
- Money Managers create investment products and portfolios for users to purchase.
- Groups support CRUD-based operational workflows.
- Each domain owns its own schema and microservices, enabling independent evolution.
- Cross-cutting Tech Domain services provide centralized security, configuration, observability, auditability, and operational controls.