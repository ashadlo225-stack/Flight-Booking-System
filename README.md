# ✈️ Flight Booking System (SkyWings)

A microservices-based flight booking platform built with **Spring Boot**, **Spring Cloud** and **Angular**. Passengers can search flights, pick seats, pay, receive an invoice and e-mail, check in online and track their flight live. Admins manage flights, fleet, airports and support.

> **Status:** 🚧 Work in progress. The source code will be uploaded once the project is ready.

---

## 📸 Screenshots

Captured from the running application (Angular frontend talking to the live Spring Boot microservices through the API Gateway).

### Landing Page
Hero banner, quick flight search bar and navigation to flights, live radar, bookings and support.

![Landing page](docs/screenshots/01-landing.png)

### Sign In
![Login page](docs/screenshots/02-login.png)

### Flight Search
Search by route and date, with fares, airline filters, cabin class, price slider, in-flight meal info and seat availability, all loaded live from the search, flight and fare services.

![Flight search results](docs/screenshots/03-flight-search.png)

### Seat Selection
Aircraft cabin map with Business, Premium Economy and Economy cabins, exit rows, a per-passenger seat allocation panel, live price total and temporary seat holds.

![Seat selection map](docs/screenshots/04-seat-selection.png)

### Live Flight Radar
Real-time tracking showing altitude, ground speed, heading, gate and estimated arrival, refreshed every few seconds from the flight-tracking service.

![Live flight radar](docs/screenshots/05-live-radar.png)

### My Bookings and E-Tickets
Passengers see every booking (PNR), flight, route, fare and seat, with options to view boarding passes, reschedule, or cancel with an instant refund.

![My bookings](docs/screenshots/06-my-bookings.png)

### Boarding Pass
Confirmed and paid bookings get an electronic boarding pass with ticket number, seat, gate, boarding time, in-flight meal and barcode. The pass is only issued after the booking service verifies the booking and payment.

![Boarding pass](docs/screenshots/07-boarding-pass.png)

### Customer Support
Help desk with a support assistant, suggested topics and searchable FAQs.

![Customer support](docs/screenshots/08-customer-support.png)

### Admin Dashboard
Operations view for administrators: KPI cards (flights, customers, bookings, revenue, pending payments), passenger bookings table, flight fleet and registered users. Passenger details in this screenshot are masked.

![Admin dashboard](docs/screenshots/09-admin-dashboard.png)

### Fleet and Flight Management Console
Airline operations can add and edit flights, manage aircraft, in-flight meals, weekly schedules and the cabin seat configuration.

![Fleet console](docs/screenshots/10-fleet-console.png)

---

## 📑 Table of Contents

0. [Screenshots](#-screenshots)
1. [System Architecture](#1-system-architecture)
2. [Services and Ports](#2-services-and-ports)
3. [End-to-End Booking Workflow](#3-end-to-end-booking-workflow)
4. [Booking Sequence (Who Calls Whom)](#4-booking-sequence-who-calls-whom)
5. [Booking State Lifecycle](#5-booking-state-lifecycle)
6. [Event-Driven Messaging (RabbitMQ)](#6-event-driven-messaging-rabbitmq)
7. [Online Check-In Workflow](#7-online-check-in-workflow)
8. [Live Flight Tracking Workflow](#8-live-flight-tracking-workflow)
9. [Frontend Page Map](#9-frontend-page-map)
10. [Startup Order](#10-startup-order)
11. [Tech Stack](#11-tech-stack)

---

## 1. System Architecture

The Angular frontend talks only to the **API Gateway**. The gateway finds services through **Eureka** and forwards requests. All services read their settings from the **Config Server**. Services talk to each other in two ways: direct **Feign** calls when an answer is needed now, and **RabbitMQ** events when a reaction can happen later.

```mermaid
flowchart TB
    User(["👤 Passenger / Admin"]) --> FE["🖥️ Angular Frontend"]
    FE -->|"HTTP /api/**"| GW["🚪 API Gateway :8080"]

    subgraph Infra["Infrastructure"]
        CFG["⚙️ Config Server :8888"]
        EUR["📖 Eureka Server :8761"]
        MQ{{"🐇 RabbitMQ<br/>flight.exchange"}}
    end

    GW -. "discover (lb://)" .-> EUR
    GW --> AUTH["🔐 Auth"]
    GW --> PROF["🙍 Profile"]
    GW --> SEARCH["🔎 Search"]
    GW --> FLIGHT["🛫 Flight"]
    GW --> FARE["💲 Fare"]
    GW --> SEAT["💺 Seat"]
    GW --> BOOK["📋 Booking"]
    GW --> PAY["💳 Payment"]
    GW --> INV["🧾 Invoice"]
    GW --> CHK["🛂 Check-in"]
    GW --> NOTIF["📧 Notification"]
    GW --> TRACK["📡 Flight Tracking"]

    AUTH & PROF & SEARCH & FLIGHT & FARE & SEAT & BOOK & PAY & INV & CHK & NOTIF & TRACK -. "load config" .-> CFG
    AUTH & SEARCH & FLIGHT & BOOK & PAY & SEAT & INV & CHK & NOTIF & TRACK <-->|"publish / consume events"| MQ

    DB[("🗄️ MySQL<br/>one database per service")]
    AUTH & FLIGHT & FARE & SEAT & BOOK & PAY & INV & CHK & NOTIF & SEARCH --> DB
```

---

## 2. Services and Ports

| Service | Port | Responsibility |
|---------|------|----------------|
| `config-server` | 8888 | Central configuration for all services |
| `eureka-server` | 8761 | Service registry and discovery |
| `api-gateway` | 8080 | Single entry point, routes `/api/**` to services |
| `auth-service` | 8081 | Register, login (JWT and Google sign-in), password reset |
| `flight-service` | 8082 | Flights, schedules, aircraft, meals (admin managed) |
| `search-service` | 8083 | Fast flight search with fares |
| `booking-service` | 8084 | Creates and manages bookings, cancel, reschedule, refund |
| `fare-service` | 8085 | Prices per flight and class |
| `seat-service` | 8086 | Seat map, hold, book and release seats |
| `payment-service` | 8087 | Payments and refunds |
| `invoice-service` | 8088 | Generates invoices after payment |
| `notification-service` | 8089 | E-mail notifications |
| `checkin-service` | 8090 | Online check-in and boarding pass |
| `profile-service` | 8091 | Passenger profile and saved details |
| `flight-tracking-service` | 8092 | Live location, delay, gate and status updates |

---

## 3. End-to-End Booking Workflow

This is the main journey a passenger follows, from sign-up to boarding.

```mermaid
flowchart LR
    A["1️⃣ Register / Login"] --> B["2️⃣ Search flights"]
    B --> C["3️⃣ Choose flight and fare"]
    C --> D["4️⃣ Enter passenger details"]
    D --> E["5️⃣ Select seats and meals"]
    E --> F["6️⃣ Review and pay"]
    F --> G{"Payment OK?"}
    G -- "Yes" --> H["7️⃣ Booking CONFIRMED"]
    G -- "No" --> X["❌ Booking FAILED<br/>seats released"]
    H --> I["8️⃣ Invoice and e-mail ticket"]
    I --> J["9️⃣ Online check-in"]
    J --> K["🔟 Track flight and board"]
```

| Step | What the user does | What happens behind the scenes |
|------|-------------------|--------------------------------|
| 1 | Registers or logs in (e-mail or Google) | `auth-service` issues a JWT and publishes `auth.user.registered`, so the welcome e-mail goes out |
| 2 | Searches by route and date | `search-service` reads flights from `flight-service` and prices from `fare-service` |
| 3 | Picks a flight and class | `booking-service` checks the flight and fare through Feign |
| 4 | Enters passenger details | Details can be filled in from `profile-service` |
| 5 | Selects seats and meals | `seat-service` holds the seats temporarily (`seat.held`) |
| 6 | Pays | `payment-service` processes the payment and publishes `payment.success` or `payment.failed` |
| 7 | Sees the confirmation | `booking-service` marks the booking confirmed (`booking.confirmed`) and `seat-service` marks the seats as booked |
| 8 | Receives ticket and invoice | `invoice-service` creates the invoice (`invoice.generated`), `notification-service` sends the e-mail |
| 9 | Checks in online | `checkin-service` verifies the booking and seat, then publishes `checkin.completed` |
| 10 | Follows the flight | `flight-tracking-service` pushes live updates and delay or gate alerts |

---

## 4. Booking Sequence (Who Calls Whom)

Solid arrows are direct (Feign) calls. Dashed arrows are RabbitMQ events.

```mermaid
sequenceDiagram
    autonumber
    actor U as Passenger
    participant FE as Angular App
    participant GW as API Gateway
    participant BK as Booking Service
    participant FL as Flight Service
    participant FR as Fare Service
    participant ST as Seat Service
    participant PY as Payment Service
    participant MQ as RabbitMQ
    participant IV as Invoice Service
    participant NT as Notification Service

    U->>FE: Select flight, seats and pay
    FE->>GW: POST /api/bookings
    GW->>BK: route (lb://BOOKING-SERVICE)
    BK->>FL: Validate flight
    BK->>FR: Get fare
    BK->>ST: Hold selected seats
    BK-->>MQ: booking.created
    BK-->>FE: Booking PENDING

    FE->>GW: POST /api/payments
    GW->>PY: route
    PY->>BK: Fetch booking amount
    alt Payment succeeds
        PY-->>MQ: payment.success
        MQ-->>BK: confirm booking
        BK-->>MQ: booking.confirmed
        MQ-->>ST: mark seats BOOKED
        MQ-->>IV: generate invoice
        IV-->>MQ: invoice.generated
        MQ-->>NT: send ticket and invoice e-mail
        NT-->>U: 📧 Confirmation e-mail
    else Payment fails
        PY-->>MQ: payment.failed
        MQ-->>BK: mark booking FAILED
        BK-->>MQ: booking.failed
        MQ-->>ST: release seats
        MQ-->>NT: send failure e-mail
    end
```

---

## 5. Booking State Lifecycle

```mermaid
stateDiagram-v2
    [*] --> PENDING: Booking created, seats held
    PENDING --> CONFIRMED: payment.success
    PENDING --> FAILED: payment.failed or hold expired
    CONFIRMED --> RESCHEDULED: Passenger changes flight
    RESCHEDULED --> CONFIRMED: New seats confirmed
    CONFIRMED --> CANCELLED: Passenger cancels
    CANCELLED --> REFUNDED: payment.refunded
    CANCELLED --> REFUND_DENIED: Not eligible for refund
    CONFIRMED --> CHECKED_IN: checkin.completed
    CHECKED_IN --> [*]
    FAILED --> [*]
    REFUNDED --> [*]
    REFUND_DENIED --> [*]
```

> The exact state names are illustrative and match the booking events the service publishes (`booking.created`, `confirmed`, `cancelled`, `rescheduled`, `refund.denied`, `failed`).

---

## 6. Event-Driven Messaging (RabbitMQ)

All services publish to one topic exchange, `flight.exchange`. Each interested service has its own durable queue, so a slow or offline consumer never blocks the publisher.

```mermaid
flowchart LR
    subgraph Publishers
        AUTH["Auth"]
        FLIGHT["Flight"]
        SEAT["Seat"]
        BOOK["Booking"]
        PAY["Payment"]
        INV["Invoice"]
        CHK["Check-in"]
        TRK["Flight Tracking"]
    end

    EX{{"flight.exchange<br/>(topic)"}}

    AUTH -->|"auth.user.registered<br/>auth.password.reset"| EX
    FLIGHT -->|"flight.created / updated<br/>deleted / rescheduled"| EX
    SEAT -->|"seat.held / booked<br/>released"| EX
    BOOK -->|"booking.created / confirmed<br/>cancelled / rescheduled<br/>refund.denied / failed"| EX
    PAY -->|"payment.success / failed<br/>refunded"| EX
    INV -->|"invoice.generated"| EX
    CHK -->|"checkin.completed"| EX
    TRK -->|"location / delay / gate<br/>depart / land / cancel"| EX

    EX --> QS["search.events.queue"] --> SEARCH["Search<br/>(keeps index fresh)"]
    EX --> QSE["seat.events.queue<br/>+ flight created queue"] --> SEATC["Seat<br/>(builds seat map)"]
    EX --> QB["booking queue"] --> BOOKC["Booking<br/>(reacts to payment)"]
    EX --> QP["payment.events.queue"] --> PAYC["Payment<br/>(reacts to booking)"]
    EX --> QI["invoice queue"] --> INVC["Invoice<br/>(creates invoice)"]
    EX --> QN["notification.events.queue"] --> NOTC["Notification<br/>(sends e-mails)"]
```

---

## 7. Online Check-In Workflow

```mermaid
flowchart TD
    A["Passenger opens My Trips"] --> B["Requests check-in"]
    B --> C["checkin-service asks booking-service<br/>Is the booking CONFIRMED?"]
    C -->|"No"| X["Reject with a clear message"]
    C -->|"Yes"| D["checkin-service asks seat-service<br/>Is the seat assigned?"]
    D -->|"No"| E["Passenger picks a seat"]
    E --> F
    D -->|"Yes"| F["Check-in recorded"]
    F --> G["Publish checkin.completed"]
    G --> H["notification-service e-mails the boarding pass"]
    F --> I["Passenger downloads the boarding pass"]
```

---

## 8. Live Flight Tracking Workflow

```mermaid
flowchart LR
    SRC["Flight data and admin updates"] --> TRK["flight-tracking-service"]
    TRK -->|"location, delay, gate change,<br/>departed, landed, cancelled"| EX{{"tracking exchange"}}
    EX --> NOTIF["notification-service"]
    NOTIF --> MAIL["📧 Delay and gate alerts to passengers"]
    TRK -->|"REST /api/flight-tracking/**"| GW["API Gateway"]
    GW --> FE["🖥️ Tracking page in Angular"]
    FE --> USER(["👤 Passenger sees live status"])
```

---

## 9. Frontend Page Map

The Angular app is split into `core` (guards, interceptors, models, services), `shared`, and one folder per feature.

```mermaid
flowchart TD
    LAND["Landing"] --> LOGIN["Login"]
    LAND --> REG["Register"]
    LAND --> SEARCHP["Flight Search Dashboard"]
    LOGIN --> SEARCHP
    SEARCHP --> PASS["Passenger Details"]
    PASS --> SEATP["Seat Selection"]
    SEATP --> PAYP["Checkout / Payment"]
    PAYP --> TICKET["Ticket"]
    TICKET --> TRACKP["Flight Tracking"]
    LOGIN --> PROFILE["Profile"]
    LOGIN --> SUPPORT["Customer Support"]
    LOGIN -->|"Admin role"| ADMIN["Admin Dashboard"]
    ADMIN --> AIR["Airport Console"]
    ADMIN --> FLEET["Fleet Console"]
    ADMIN --> SUPC["Support Console"]
```

---

## 10. Startup Order

Services depend on each other, so start them in this order:

```mermaid
flowchart LR
    S0["MySQL and RabbitMQ"] --> S1["1. Config Server :8888"]
    S1 --> S2["2. Eureka Server :8761"]
    S2 --> S3["3. Business services<br/>auth, flight, fare, seat, search, booking,<br/>payment, invoice, notification, check-in,<br/>profile, flight-tracking"]
    S3 --> S4["4. API Gateway :8080"]
    S4 --> S5["5. Angular frontend"]
```

1. Run the SQL scripts in the `sql/` folder to create the databases.
2. Copy your secrets into a local `.env` file (database, RabbitMQ, mail, Google client). This file is **git-ignored** and never committed.
3. Start the services in the order above, then open the Eureka dashboard at `http://localhost:8761` to confirm they registered.
4. Use the Postman collections in `postman/` to test every endpoint through the gateway at `http://localhost:8080`.

---

## 11. Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Angular, TypeScript |
| Backend | Java, Spring Boot, Spring Security, Spring Data JPA |
| Microservices | Spring Cloud Gateway, Eureka, Config Server, OpenFeign |
| Messaging | RabbitMQ (topic exchange) |
| Database | MySQL (one schema per service) |
| API docs and testing | Swagger UI (springdoc), Postman |
| Build | Maven |

---

## 👤 Author

[ashadlo225-stack](https://github.com/ashadlo225-stack)
