# AquaPulse: Complete Water Monitoring System (Weeks 1–4)
## Project Guide, Architecture & Live Presentation Demo Script

Welcome to **AquaPulse** — a multi-tenant smart water monitoring, automated tiered billing, and leak detection platform built with **React, Spring Boot 3.3.3, Spring Security 6 (JWT), and PostgreSQL**.

---

## 1. System Architecture & Core Formulas

```
+-------------------------------------------------------------------------+
|                  React 19 Frontend (AquaPulse UI)                       |
|   - Resident Portal      - Admin Dashboard     - Super Admin Console    |
|   - Real-time Telemetry  - Interactive Modals  - Responsive Buttons     |
+-------------------------------------------------------------------------+
                                    |
                                    | REST (JSON / Multipart CSV)
                                    v
+-------------------------------------------------------------------------+
|                Spring Boot 3.3.x REST API Backend                       |
|   +-----------------------------------------------------------------+   |
|   | Spring Security 6 (Stateless JWT Filter with Role Authorization)|   |
|   +-----------------------------------------------------------------+   |
|   |  - AuthController          - ApartmentController                |   |
|   |  - HouseholdController     - WaterUsageLogController            |   |
|   |  - BillingCycleController  - BulkPurchaseController             |   |
|   |  - TariffController        - AlertController                    |   |
|   +-----------------------------------------------------------------+   |
|   | Core Business Services:                                         |   |
|   |  * TieredBillingEngineService (Slab rate calculations)          |   |
|   |  * ConsumptionDistributionService (Proportional Apportionment)  |   |
|   |  * AlertEngineScheduler (Spring @Scheduled 2-Sigma Outliers)    |   |
|   |  * CsvWaterLogParserService (Batch Streaming & Deduplication)   |   |
|   +-----------------------------------------------------------------+   |
|   | Spring Data JPA Entities & Cascades                             |   |
+-------------------------------------------------------------------------+
                                    |
                                    | JDBC / Flyway Database Migrations
                                    v
+-------------------------------------------------------------------------+
|                     PostgreSQL Database Engine                          |
|   apartments  |  households  |  users  |  water_meters                 |
|   water_usage_logs  |  tariff_plans  |  tariff_slabs                   |
|   bulk_water_purchases  |  billing_cycles  |  invoices & items          |
|   alert_notifications                                                   |
+-------------------------------------------------------------------------+
```

---

### Core Mathematical Formulations

#### 1. Tiered Tariff Calculation Formula
Water charges are computed progressively using slab bounds $[L_s, U_s]$ and rate $R_s$:
$$\text{Tiered Charge} = \sum_{s \in \text{Slabs}} \max\Big(0, \min(\text{Volume}, U_s) - L_s\Big) \times R_s$$
- **Tier 1 (Lifeline baseline)**: 0 – 10 kL @ ₹25.00/kL
- **Tier 2 (Standard usage)**: 10.01 – 20 kL @ ₹30.00/kL
- **Tier 3 (High usage penalty)**: > 20 kL @ ₹40.00/kL
*Example:* A household consuming **16.0 kL**:
$$\text{Cost} = (10 \times 25) + (6 \times 30) = 250 + 180 = ₹430.00$$

#### 2. Consumption-Based Common Water Distribution (with Flat Area Fallback)
Shared apartment water costs (tanker deliveries and municipal supply) are apportioned:
- **Metered Units**: Proportional to metered consumption:
  $$\text{Share}_i = \text{Metered Pool Cost} \times \frac{\text{Metered Consumption}_i}{\sum_{j} \text{Metered Consumption}_j}$$
- **Unmetered Units (Fallback)**: Proportional to flat area (sq ft):
  $$\text{Share}_k = \text{Unmetered Pool Cost} \times \frac{\text{Area}_k}{\sum_{m} \text{Area}_m}$$

#### 3. Statistical Leak Detection ($2\sigma$ Outlier Rule)
Spring `@Scheduled` task analyzes historical 14-day telemetry:
- Mean: $\mu = \frac{1}{N} \sum_{i=1}^N x_i$
- Standard Deviation: $\sigma = \sqrt{\frac{1}{N-1} \sum_{i=1}^N (x_i - \mu)^2}$
- Anomaly Flag: If current reading $x > \mu + 2\sigma$ ($Z = \frac{x - \mu}{\sigma} > 2.0$), the household is flagged as a **potential continuous pipe/tap leak**.

---

## 2. Directory Structure of the Backend

The backend code is organized under `water-monitoring-frontend/backend/`:
```
backend/
├── pom.xml                                      <- Spring Boot 3, Security 6, PostgreSQL, Flyway, JJWT
└── src/
    ├── main/
    │   ├── resources/
    │   │   ├── application.properties           <- Datasource, JWT secret, 2-sigma thresholds
    │   │   └── db/migration/
    │   │       ├── V1__init_schema.sql          <- DDL schema, foreign keys, cascades, unique constraints
    │   │       └── V2__seed_demo_data.sql       <- Akshaya Residency, Admin, Residents, Slabs, Logs
    │   └── java/com/aquapulse/
    │       ├── WaterMonitoringApplication.java  <- @SpringBootApplication, @EnableScheduling
    │       ├── entity/                          <- JPA Entities with relationships & cascades
    │       │   ├── Apartment.java
    │       │   ├── Household.java
    │       │   ├── User.java & Role.java
    │       │   ├── WaterMeter.java
    │       │   ├── WaterUsageLog.java
    │       │   ├── TariffPlan.java & TariffSlab.java
    │       │   ├── BulkWaterPurchase.java
    │       │   ├── BillingCycle.java
    │       │   ├── Invoice.java & InvoiceItem.java
    │       │   └── AlertNotification.java
    │       ├── security/                        <- Spring Security 6 & JWT
    │       │   ├── JwtService.java
    │       │   ├── JwtAuthenticationFilter.java
    │       │   ├── SecurityConfig.java
    │       │   └── CustomUserDetailsService.java
    │       ├── dto/                             <- DTOs with Jakarta Bean Validation
    │       ├── repository/                      <- Spring Data JPA Repositories
    │       ├── service/                         <- Core Business Logic Services
    │       │   ├── AuthService.java
    │       │   ├── ApartmentService.java
    │       │   ├── HouseholdService.java
    │       │   ├── WaterUsageLogService.java    <- Manual entry + CSV parsing + deduplication
    │       │   ├── TieredBillingEngineService.java
    │       │   ├── ConsumptionDistributionService.java
    │       │   ├── BillingCycleService.java
    │       │   ├── AlertEngineScheduler.java    <- @Scheduled 2-sigma outlier engine
    │       │   └── BulkWaterPurchaseService.java
    │       └── controller/                      <- REST Controllers
    │           ├── AuthController.java
    │           ├── ApartmentController.java
    │           ├── HouseholdController.java
    │           ├── WaterUsageLogController.java
    │           ├── BillingCycleController.java
    │           ├── BulkPurchaseController.java
    │           ├── TariffController.java
    │           └── AlertController.java
    └── test/                                    <- Test Suite
        ├── resources/application-test.properties
        └── java/com/aquapulse/
            ├── service/
            │   ├── TieredBillingEngineServiceTest.java       <- Unit tests for slab calculations
            │   ├── ConsumptionDistributionServiceTest.java   <- Unit tests for proportional allocation
            │   └── WaterUsageLogServiceTest.java             <- Mockito tests for duplicate detection
            └── controller/
                ├── AuthControllerIntegrationTest.java        <- Spring Boot MockMvc integration test
                └── WaterUsageLogControllerIntegrationTest.java
```

---

## 3. How to Run the System

### A. Run the Interactive Frontend Model (Ready for Demonstration Now!)
In your terminal inside `c:\Users\kiranmayee\Desktop\Water Monitoring\water-monitoring-frontend`:
```powershell
cmd.exe /c "npm run dev"
```
Open **http://localhost:5173** in your browser.

### B. Run the Spring Boot Backend (When PostgreSQL is running)
Inside `water-monitoring-frontend/backend/`:
```bash
mvn spring-boot:run
```
Flyway will automatically execute `V1__init_schema.sql` and `V2__seed_demo_data.sql` on startup!

---

## 4. Live Demo Walkthrough Script for Today's Presentation

Use this step-by-step presentation script to impress evaluators:

### Step 1: Login & Quick Demo Switcher
1. Open the app at `http://localhost:5173`.
2. Point out the clean landing screen with Gmail validation and password security.
3. Show the **"OR PRESENTATION QUICK DEMO"** section:
   - Click **"🛡 Quick Login as Apartment Admin (Akshaya)"**.
   - Notice how you are instantly logged in without typing errors!
   - Highlight the **AquaPulse Live Model Switcher Bar** pinned at the top, allowing 1-click role swapping.

### Step 2: Apartment Admin - Water Usage Logging (Weeks 1–2)
1. In the sidebar, click **Meter Readings**:
   - Select **Flat 201 (Kiranmai)**.
   - Notice the previous reading (12.45 kL) displayed automatically.
   - In **Cumulative Index**, type `12.72`.
   - The UI immediately displays: `⚡ Computed Daily Consumption: 0.270 kL (270 Liters)`.
   - Click **"Compute & Log Reading"**.
   - A success toast appears and the new log is added to the table with audit details!
2. Test **Bulk CSV Ingestion & Duplicate Detection**:
   - Click **"☁ Bulk CSV Batch Ingestion"** at the top right.
   - Click **"🔍 Run Validation & Duplicate Detection"**.
   - Show the dry-run results:
     - 5 valid records marked in green `✔ Valid Record`.
     - 1 duplicate row marked in red: `❌ DUPLICATE: MTR-1101 already in batch for 2026-09-09`.
   - Click **"Ingest 5 Valid Records"** to commit.

### Step 3: Tariff Configuration & Tiered Billing Engine (Weeks 3–4)
1. Click **Tariff Plans** in the sidebar:
   - Show the 3 slabs:
     - Tier 1: 0 – 10 kL @ ₹25
     - Tier 2: 10 – 20 kL @ ₹30
     - Tier 3: > 20 kL @ ₹40
   - Show the **Interactive Tiered Billing Calculator (Simulator)**:
     - Click `16 kL` $\rightarrow$ watch the live breakdown: 10 kL @ ₹25 (₹250) + 6 kL @ ₹30 (₹180) + ₹50 fee = ₹480.00.
2. Click **Bulk Purchases** in the sidebar:
   - Point out the tanker deliveries and municipal feeder supply logs.
   - Click **"+ Add Tanker / Municipal Purchase"** $\rightarrow$ add a 12 kL tanker @ ₹150 $\rightarrow$ automatically recalculates the apartment's common water cost pool.
3. Click **Billing Management**:
   - Point out the cycle status: `August 2026 (OPEN)`.
   - Click **"⚡ Run Tiered Billing Engine"**.
   - The engine automatically:
     - Computes tiered charges per metered flat.
     - Apportions the ₹7,200 tanker pool proportionally to metered usage.
     - Falls back to flat area distribution for Flat 401 (unmetered).
   - Click **"View Invoice"** on Flat 201 $\rightarrow$ shows the itemized printable invoice modal with tiered breakdown and common tanker share!

### Step 4: Statistical Outlier Leak Detection Engine (Weeks 3–4)
1. Click **Water Leakage** in the sidebar:
   - Highlight the statistical cards:
     - Household Mean ($\mu = 248\text{ L/day}$)
     - Standard Deviation ($\sigma = 42\text{ L/day}$)
     - $2\sigma$ Anomaly Threshold ($> 332\text{ L/day}$)
   - Point out **Flat 204** (820 L/day, $Z = +13.6\sigma$, **CRITICAL LEAK RISK**).
   - Click **"📲 Send Warning Notice"** $\rightarrow$ dispatches simulated SMS/Email alert with toast confirmation.
   - Click **"⚡ Trigger @Scheduled Scan"** $\rightarrow$ simulates the Spring cron execution.

### Step 5: Switch to Resident Portal
1. From the top bar, click **"👤 Resident (Kiranmai)"**:
   - The UI switches to Kiranmai's resident portal instantly.
   - View **My Bills**: Shows the outstanding due of ₹525.00.
   - Click **"Pay Online"**:
     - Choose payment method: **UPI / QR Code**.
     - Click **"Confirm & Pay ₹525.00"**.
     - Instant payment receipt `#TXN-AQ-...` is generated.
     - Status updates to **PAID** and outstanding balance drops to ₹0.00!
   - Click **Support / Concerns**:
     - Click **"+ Raise Concern"** $\rightarrow$ report a leak concern $\rightarrow$ logged to maintenance tickets!

---

## 5. Summary of Completed Requirements

| Requirement | Module / Component | Status |
| :--- | :--- | :---: |
| **PostgreSQL Schema & Flyway** | `V1__init_schema.sql` & `V2__seed_demo_data.sql` | Completed |
| **Spring Security 6 with JWT** | `JwtService`, `JwtAuthenticationFilter`, `SecurityConfig` | Completed |
| **Registration, Login & Profiles** | `AuthController`, `AuthService`, Jakarta `@Valid` | Completed |
| **Apartment Onboarding & Meters** | `ApartmentController`, `HouseholdController` | Completed |
| **Manual & Bulk CSV Water Logging** | `WaterUsageLogController`, Commons CSV + Duplicate Check | Completed |
| **Tiered Tariff Billing Engine** | `TieredBillingEngineService` (Slab progressive rates) | Completed |
| **Bulk Tanker Purchase Tracking** | `BulkPurchaseController`, `BulkWaterPurchaseService` | Completed |
| **Consumption Distribution Algorithm**| `ConsumptionDistributionService` (Proportional + Flat Area) | Completed |
| **2-Sigma Leak Detection Alert Engine**| `AlertEngineScheduler` (`@Scheduled` $x > \mu + 2\sigma$) | Completed |
| **Billing Cycle & Itemized Invoices** | `BillingCycleController`, `Invoice`, `InvoiceItem` | Completed |
| **JUnit 5 & Mockito Unit Tests** | `TieredBillingEngineServiceTest`, `ConsumptionDistributionServiceTest`, `WaterUsageLogServiceTest` | Completed |
| **MockMvc Integration Tests** | `AuthControllerIntegrationTest`, `WaterUsageLogControllerIntegrationTest` | Completed |
| **Interactive Responsive Frontend** | All buttons active with modals, live formulas & 1-click switcher | Completed |
