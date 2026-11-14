# Flight Booking System (SkyWings)

A microservices-based flight booking platform built with Spring Boot and Angular.

> **Status:** 🚧 Work in progress. The source code will be uploaded once the project is ready.

## Planned Architecture

| Component | Purpose |
|-----------|---------|
| `config-server` | Centralized configuration |
| `eureka-server` | Service discovery |
| `api-gateway` | Single entry point and routing |
| `auth-service` | Authentication and user management |
| `profile-service` | Passenger profiles |
| `flight-service` / `search-service` | Flight data and search |
| `fare-service` | Fare calculation |
| `seat-service` | Seat selection |
| `booking-service` | Booking management |
| `payment-service` | Payments |
| `checkin-service` | Online check-in |
| `invoice-service` | Invoices |
| `notification-service` | Notifications |
| `flight-tracking-service` | Live flight tracking |
| `frontend` | Angular web application |

## Tech Stack

- Java, Spring Boot, Spring Cloud (Eureka, Config Server, API Gateway)
- Spring Security, Spring Data JPA
- MySQL
- Angular
- Maven

## Author

[ashadlo225-stack](https://github.com/ashadlo225-stack)
