# SkinTwin-AI Ecosystem Design

This repository contains the comprehensive ecosystem design for the **skintwin-ai** organization. It provides a high-level overview of the integrated architecture, including a Shopify app foundation, backend services, and network layers that connect all repositories into a cohesive and scalable platform with federated enterprise ERP integration.

## 1. Introduction

The SkinTwin-AI ecosystem is a collection of repositories that together form a revolutionary, AI-driven beauty-tech platform. This document outlines the architecture that integrates these components, enabling seamless data flow, shared services, and a unified user experience.

## 2. Repository Ecosystem

The ecosystem is composed of several repositories, each with a specific role. The following diagram illustrates the relationships and dependencies between them:

![Repository Mapping](./diagrams/repository_mapping.png)

## 3. High-Level Architecture

The platform is designed around a Shopify-first microservices architecture, with a clear separation between Shopify app surface, backend domain services, and AI/cognitive layers. A federated enterprise-wide ERP integration layer synchronizes master data, orders, inventory, and finance across business units. The following diagram provides a high-level overview of the entire ecosystem:

![Ecosystem Architecture](./diagrams/ecosystem_architecture.png)

### 3.1. Frontend Architecture

The frontend layer is built as a monorepo containing four primary applications: the Shopify Embedded App, Customer Portal, Training LMS, and Business Directory. These applications share a common UI library and design system to ensure a consistent user experience.

[**Read more about the Frontend Architecture](./design/frontend_architecture.md)**

### 3.2. Backend Architecture

The backend is composed of several microservices, each responsible for a specific business domain. These services are designed to be independently scalable and deployable.

[**Read more about the Backend Architecture](./design/backend_architecture.md)**

### 3.3. Network & API Gateway Architecture

A secure and scalable network layer connects all the components of the ecosystem. An API Gateway serves as the single entry point for all frontend applications, providing routing, authentication, and rate limiting.

[**Read more about the Network & API Gateway Architecture](./design/network_architecture.md)**

## 4. Data Flow

The following diagram illustrates the flow of data between Shopify app interactions, backend processing, federated ERP synchronization, and external integrations:

![Data Flow Diagram](./diagrams/data_flow.png)

## 5. Design Documents

This repository contains detailed design documents for each architectural layer:

- [**Repository Analysis**](./analysis/repository_analysis.md)
- [**Integration Mapping**](./analysis/integration_mapping.md)
- [**Backend Architecture**](./design/backend_architecture.md)
- [**Frontend Architecture**](./design/frontend_architecture.md)
- [**Network & API Gateway Architecture**](./design/network_architecture.md)

## 6. CI/CD & Testing

This repository also serves as the central source for CI/CD infrastructure and E2E testing specifications across the ecosystem.

### 6.1. Reusable GitHub Actions Workflows

The `.github/workflows/` directory contains reusable workflow templates that can be called from any repository:

| Workflow | Purpose |
|----------|---------|
| [`ci-node.yml`](./.github/workflows/ci-node.yml) | Node.js/TypeScript build, test, lint |
| [`ci-python.yml`](./.github/workflows/ci-python.yml) | Python build, test, lint |
| [`ci-container.yml`](./.github/workflows/ci-container.yml) | Container build and security scan |
| [`ci-e2e.yml`](./.github/workflows/ci-e2e.yml) | End-to-end test execution |
| [`ci-release.yml`](./.github/workflows/ci-release.yml) | Release validation and publishing |
| [`ci-contracts.yml`](./.github/workflows/ci-contracts.yml) | Cross-repo contract validation |
| [`ci-security.yml`](./.github/workflows/ci-security.yml) | Security scanning (CodeQL, deps, secrets) |

[**Read the CI/CD Standards documentation**](./ci/README.md)

### 6.2. E2E Testing Strategy

The `/e2e/` directory contains comprehensive E2E test specifications:

- [**E2E Strategy Overview**](./e2e/README.md) - Test tiers, execution strategy, reliability engineering
- [**Shopify App Scenarios**](./e2e/scenarios/shopify-scenarios.md) - Installation, session, webhook tests
- [**Portal & LMS Scenarios**](./e2e/scenarios/portal-lms-scenarios.md) - Auth, booking, purchase, course tests
- [**ERP & Events Scenarios**](./e2e/scenarios/erp-events-scenarios.md) - Event propagation, ERP sync tests
- [**Test Fixtures**](./e2e/fixtures/README.md) - Deterministic test data specifications

### 6.3. Implementation Rollout

The CI/CD and E2E infrastructure is being rolled out in phases:

[**View the Rollout Plan**](./ci/ROLLOUT.md)

## 7. Diagrams

All architectural diagrams are available in the `/design/diagrams` directory:

- [Ecosystem Architecture Diagram](./design/diagrams/ecosystem_architecture.png)
- [Data Flow Diagram](./design/diagrams/data_flow.png)
- [Repository Mapping Diagram](./design/diagrams/repository_mapping.png)
