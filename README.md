# 🗺️ Shadow Economy Map

> **Planning and specification for a geospatial economic-growth detection platform for Addis Ababa.**

**Shadow Economy Map** is designed to identify emerging and under-visible economic growth before it becomes apparent in traditional or official statistics.

The platform combines multiple free and publicly available signals—including **VIIRS nighttime lights**, **GHSL settlement growth**, **Meta's Relative Wealth Index**, and **infrastructure/news-derived case studies**—into a composite **economic-growth score** for geographic grid cells across Addis Ababa.

The goal is to make patterns of growth easier to explore, validate, and understand—particularly in areas outside well-documented central districts.

---

## 📌 Project Status

> **Current version: V1 — MVP**

The V1 specification is complete and ready for implementation.

The repository currently contains **documentation only**. Application code and implementation test files will live in separate repositories or project directories.

For the full staged product plan, see [`roadmap.md`](./roadmap.md).

**Important:** Only **V1** is currently a committed implementation target. **V2–V5** represent future planning and should not be treated as part of the MVP scope.

---

## 🎯 What This Project Is Building

At its core, Shadow Economy Map answers a simple question:

> **Where is economic growth emerging that traditional data sources may not yet clearly show?**

The platform analyzes multiple spatial and contextual signals to surface areas that may indicate:

* 📈 Emerging economic activity
* 🏗️ Settlement and infrastructure expansion
* 🌃 Changes in nighttime economic intensity
* 🗺️ Under-observed growth outside central districts
* 🔍 Locations worth further investigation or validation

Rather than claiming to directly measure economic activity, the system produces a **composite growth-detection signal** that helps users identify areas requiring closer attention.

---

# 📚 Documentation Architecture

The documentation is intentionally structured as a dependency chain.

New contributors should read the numbered documents **from top to bottom**, as each layer builds on decisions established in the previous one.

| #      | Document                                                                         | Purpose                                                                           |
| ------ | -------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **01** | [`01-problem-and-solution-statement.md`](./01-problem-and-solution-statement.md) | Defines the problem, users, proposed solution, V1 scope, and long-term vision     |
| **02** | [`02-requirements.md`](./02-requirements.md)                                     | Defines functional and non-functional requirements                                |
| **03** | [`03-usecases.md`](./03-usecases.md)                                             | Defines system use cases (`UC-01`, etc.) referenced throughout the specifications |
| **04** | [`04-database-and-data-model.md`](./04-database-and-data-model.md)               | Defines entities, schema structure, and relationships                             |
| **05** | [`05-api/`](./05-api/)                                                           | Defines the REST API specification, organized by feature                          |
| **06** | [`06-frontend-specification/`](./06-frontend-specification/)                     | Defines frontend types, hooks, components, and feature architecture               |
| **07** | [`07-folder-and-file-structure/`](./07-folder-and-file-structure/)               | Defines the concrete backend/frontend source and test structure                   |
| **08** | [`08-function-level-specification/`](./08-function-level-specification/)         | Defines individual files, functions, signatures, side effects, and edge cases     |
| **09** | [`09-test-file-specification/`](./09-test-file-specification/)                   | Defines the test-first implementation plan, test files, and required cases        |

---

## 🧭 Recommended Reading Order

For someone joining the project for the first time:

```text
01. Problem & Solution
        ↓
02. Requirements
        ↓
03. Use Cases
        ↓
04. Data Model
        ↓
05. API Specification
        ↓
06. Frontend Specification
        ↓
07. Folder & File Structure
        ↓
08. Function-Level Specification
        ↓
09. Test File Specification
```

After completing the initial read-through, the repository can be used as a reference during implementation.

---

# 🧩 Feature Map

The core product is organized around **six major features**, with **Case Study Curation** serving as the administrative counterpart to the public Case Studies feature.

These features are specified consistently across the API, frontend architecture, function specifications, and test specifications.

| Feature                       |  API | Frontend | Backend Functions | Frontend Functions | Backend Tests | Frontend Tests |
| ----------------------------- | :--: | :------: | :---------------: | :----------------: | :-----------: | :------------: |
| **Shared / Configuration**    |   —  |     —    |       `8-1`       |        `9-1`       |     `9-1`     |      `9-1`     |
| **Growth Map & Exploration**  | `01` |   `01`   |       `8-2`       |        `9-2`       |     `9-2`     |      `9-2`     |
| **Case Studies & Validation** | `02` |   `02`   |       `8-3`       |        `9-3`       |     `9-3`     |      `9-3`     |
| **Site Content & Onboarding** | `03` |   `03`   |       `8-4`       |        `9-4`       |     `9-4`     |      `9-4`     |
| **Admin Access**              | `04` |   `04`   |       `8-5`       |        `9-5`       |     `9-5`     |      `9-5`     |
| **Data Pipeline Management**  | `05` |   `05`   |       `8-6`       |        `9-6`       |     `9-6`     |      `9-6`     |
| **Case Study Curation**       | `06` |   `06`   |       `8-7`       |        `9-7`       |     `9-7`     |      `9-7`     |

### Public vs. Administrative Features

Case Study Curation is intentionally separated from the public Case Studies feature.

Both operate on the same underlying domain entity, but the API architecture deliberately separates:

```text
Public Case Studies
        │
        ├── Public routes
        ├── Public caching strategy
        └── Read-oriented access
```

```text
Administrative Case Study Curation
        │
        ├── Protected admin routes
        ├── Administrative workflows
        └── Content management operations
```

This separation is defined in the API specification and preserves a clear boundary between public exploration and administrative management.

---

# 🏗️ Engineering Conventions

The specifications assume a consistent architecture across the project.

## Backend

```text
Node.js
└── Express
    └── Prisma
        └── Layered Architecture

Schema
  ↓
Service
  ↓
Controller
  ↓
Routes
```

### Backend Testing

Tests use **Vitest**, following patterns such as:

* `describe`
* `it`
* `expect`
* `vi.fn()`
* `vi.mock()`

---

## Frontend

The frontend specification is built around:

* **React**
* **TanStack Query**
* **React Hook Form**
* **Zod**

The architecture separates responsibilities across:

```text
Types
  ↓
API / Data Access
  ↓
Hooks
  ↓
Components
  ↓
Pages / Features
```

### Frontend Testing

Tests use:

* **Vitest**
* **React Testing Library**
* `render`
* `renderHook`
* `userEvent`

---

# 🧪 Test-First Development

The project follows a **test-first specification approach**.

The `09-test-file-specification` documentation defines the tests that should exist before or alongside implementation.

### Test Structure

Test files must mirror the application source structure:

```text
src/
└── services/
    └── growth.service.ts

tests/
└── services/
    └── growth.service.test.ts
```

Tests are **never co-located** with implementation files.

Every major implementation unit—including:

* Services
* Controllers
* Routes
* Hooks
* Components
* Pages

has a corresponding test specification.

The exact file map and required test cases are defined in each feature's `09` specification.

---

# 🔒 High-Scrutiny Modules

Certain parts of the system require stronger regression protection than standard happy-path coverage.

These include:

### 🔐 Administrative Authentication

Authentication and authorization flows must include explicit regression tests around access boundaries and protected functionality.

### ⚠️ AI-Assisted or Auto-Publish-Adjacent Flows

Any flow involving:

* Automated discovery
* AI-assisted processing
* Warning or caution branches
* Publication-adjacent actions

receives additional explicit test coverage.

These requirements are defined in the **Testing Guide — Section 12** and referenced throughout the `09` specifications.

The goal is to protect against subtle regressions where a system may technically function while bypassing important review, warning, or authorization boundaries.

---

# 📖 Documentation Completeness

All numbered documentation layers are complete for every defined feature.

This includes both backend and frontend specifications where applicable.

Each subtree under:

* `08-function-level-specification`
* `09-test-file-specification`

ends with:

1. An explicit **completion marker**
2. A **Next: proceed to →** reference

If you're checking whether documentation is missing or incomplete, start from those markers and follow the dependency chain.

---

# 🚫 What Is Not Included

This repository intentionally does **not** contain:

* Application source code
* Production services
* Frontend implementation
* Backend implementation
* Actual test files
* Generated datasets
* Deployment configuration

The files under `09-test-file-specification` are **specifications for tests**, not the tests themselves.

---

# 🛣️ Roadmap

The product is planned across multiple stages:

```text
V1  →  MVP Implementation
 ↓
V2  →  Planned Expansion
 ↓
V3  →  Advanced Capabilities
 ↓
V4  →  Broader Intelligence & Automation
 ↓
V5  →  Long-Term Vision
```

Only **V1** should currently be considered part of the active build scope.

For the complete roadmap and version boundaries:

➡️ [`roadmap.md`](./roadmap.md)

---

## 🗂️ Repository Notes

The repository is documentation-focused.

```text
Shadow Economy Map
│
├── 01-problem-and-solution-statement.md
├── 02-requirements.md
├── 03-usecases.md
├── 04-database-and-data-model.md
│
├── 05-api/
├── 06-frontend-specification/
├── 07-folder-and-file-structure/
├── 08-function-level-specification/
├── 09-test-file-specification/
│
├── roadmap.md
│
└── packages.microsoft.gpg
    └── Unrelated infrastructure artifact
```

---

## 🤝 Contributing

Before implementing or modifying any part of the project:

1. Read the numbered documentation in order if you are new to the project.
2. Identify the relevant feature and use case.
3. Check the API and frontend specifications.
4. Follow the defined folder and file structure.
5. Review the function-level specification.
6. Review the corresponding test specification.
7. Implement without silently expanding the committed V1 scope.

The documentation is intended to act as the project's **shared engineering contract**.

---

<div align="center">

### Shadow Economy Map

**Making under-visible growth easier to detect, explore, and investigate.**

*V1 Specification Complete · Ready for Implementation*

</div>
