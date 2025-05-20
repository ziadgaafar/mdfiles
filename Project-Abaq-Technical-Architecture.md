
# 🚀 Project Abaq: Technical Architecture Rationale

## 🎯 Introduction: Building a Future-Proof LMS

This document outlines the key technical decisions made for Project Abaq, our Learning Management System. The choices of **React with Next.js**, a **Monorepo architecture powered by Turborepo**, and a **multi-app structure** were made strategically to ensure scalability, maintainability, developer efficiency, and a superior user experience.

Our goal is to build a robust platform that caters effectively to **students, teachers, and administrators**, today and in the future.

---

## 💡 Core Technology: React & Next.js - A Powerful Duo

### Why React?

- **Component-Based Architecture:**  
  Reusable UI components (buttons, forms, course cards) accelerate development and ensure UI consistency.

- **Rich Ecosystem & Community:**  
  Fast problem-solving, cutting-edge libraries, and availability of skilled developers.

- **Virtual DOM for Performance:**  
  Efficient updates = fast rendering.

- **Strong Industry Adoption:**  
  Proven stability and future support.

### Why Next.js?

- **Versatile Rendering Options:**  
  SSR for SEO, SSG for fast loading, CSR for interactive dashboards.

- **Optimized Performance:**  
  Code splitting, image optimization, and prefetching.

- **Enhanced Developer Experience (DX):**  
  TypeScript, file-system routing, fast refresh.

- **SEO Friendliness:**  
  Crucial for discoverability.

- **API Routes:**  
  Backend-for-Frontend (BFF) pattern support.

**Key Takeaway:** React provides the UI building blocks, Next.js brings the structure, performance, and full-stack capability.

---

## 🏗️ Architectural Strategy: Monorepo & Multi-App Structure

### Why a Monorepo (with Turborepo)?

- **Efficient Code Sharing:**  
  UI components, hooks, and logic shared across apps.

- **Streamlined Development & Collaboration:**  
  Atomic PRs, better refactoring.

- **Optimized Build & CI/CD Pipelines:**  
  Intelligent dependency tracking and remote caching.

- **Simplified Dependency Management:**  
  Single-point version management.

### Monorepo Structure

```mermaid
graph TD
    A["Monorepo (Turborepo)"] --> B["apps"]
    A --> C["packages"]
    B --> B1["student-app"]
    B --> B2["teacher-app"]
    B --> B3["admin-app"]
    B --> B4["auth-app"]
    C --> C1["ui-components"]
    C --> C2["shared-hooks"]
    C --> C3["localization-middleware"]
    C --> C4["api-client"]
    C --> C5["eslint-config-custom"]
    C --> C6["tsconfig-custom"]

    subgraph "Shared Code"
        direction LR
        C1; C2; C3; C4; C5; C6;
    end

    C1 --> B1; C1 --> B2; C1 --> B3; C1 --> B4;
    C2 --> B1; C2 --> B2; C3 --> B3; C2 --> B4;
    C3 --> B1; C3 --> B2; C3 --> B3; C3 --> B4;
    C4 --> B1; C4 --> B2; C4 --> B3; C4 --> B4;
    C5 -.-> B1; C5 -.-> B2; C5 -.-> B3; C5 -.-> B4; C5 -.-> C1; C5 -.-> C2; C5 -.-> C3; C5 -.-> C4;
    C6 -.-> B1; C6 -.-> B2; C6 -.-> B3; C6 -.-> B4; C6 -.-> C1; C6 -.-> C2; C6 -.-> C3; C6 -.-> C4;

    style A fill:#007acc,stroke:#fff,stroke-width:2px,color:#fff
    style B fill:#5cb85c,stroke:#fff,stroke-width:2px,color:#fff
    style C fill:#f0ad4e,stroke:#fff,stroke-width:2px,color:#fff
```

### Why Separate Applications?

- **Clear Separation of Concerns (SoC):**  
  Each app focuses on one role.

- **Independent Deployments & Scalability:**  
  Update and scale apps individually.

- **Tailored User Experiences (UX):**  
  Custom UI for students, teachers, and admins.

- **Enhanced Security Boundaries:**  
  Reduced attack surface.

- **Optimized Team Workflows:**  
  Different teams can manage different apps.

### Application Interaction & Authentication Flow

```mermaid
graph LR
    User[User Browser] -->|Visit abaq.com| StudentApp[Student App]
    User -->|Visit teacher.abaq.com| TeacherApp[Teacher App]
    User -->|Visit admin.abaq.com| AdminApp[Admin App]

    subgraph AuthFlow
        AuthApp[Auth App]
        API[API Service]
    end

    StudentApp -->|Unauthenticated| AuthApp
    TeacherApp -->|Unauthenticated| AuthApp
    AdminApp -->|Unauthenticated| AuthApp

    AuthApp -->|Login/Signup| API
    API -->|Token & Role| AuthApp
    AuthApp -->|Set Cookie| User

    User -->|Redirect to App| StudentApp
    User -->|Redirect to App| TeacherApp
    User -->|Redirect to App| AdminApp

    StudentApp -->|API Calls| API
    TeacherApp -->|API Calls| API
    AdminApp -->|API Calls| API
```

---

## ✨ Feature Highlight: Seamless Localization (ar/en)

### Shared Middleware Package

- **Functionality:**  
  - Detects user's language (browser/cookie/manual).  
  - Redirects non-localized URLs to `/ar/` or `/en/`.  
  - Supports Arabic & English.

- **Benefits:**  
  - Centralized logic.  
  - Consistent UX.  
  - Easy rollout across apps.

### Localization Flow

```mermaid
graph TD
    A[User Request: abaq.com/my-courses] --> B{Shared Localization Middleware}
    B -- Language: ar --> C[Redirect: abaq.com/ar/my-courses]
    B -- Language: en --> D[Redirect: abaq.com/en/my-courses]
    C --> E[Student App Renders Arabic Content]
    D --> F[Student App Renders English Content]

    subgraph "Monorepo: packages/localization-middleware"
        LM[middleware.ts]
    end

    StudentApp -.-> LM
    TeacherApp -.-> LM
    AdminApp -.-> LM
    AuthApp -.-> LM
```

---

## 🏆 Overall Benefits & Synergy

- **High Performance:** Fast, optimized load times.
- **Scalability:** Apps and teams can grow independently.
- **Maintainability:** Clean boundaries, reusable packages.
- **Developer Velocity:** Fast dev cycles with shared logic.
- **Robustness:** Reduced risk via separation.
- **Future-Ready:** Flexible to expand and innovate.

---

## ✅ Conclusion

The technical decisions for Project Abaq were made with clarity and foresight. The combination of **React, Next.js, Turborepo**, and **role-based apps** ensures we're building a **fast, scalable, maintainable**, and **future-proof LMS**.
