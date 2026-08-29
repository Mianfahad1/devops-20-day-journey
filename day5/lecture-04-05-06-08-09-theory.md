# Theory Notes — Lectures 4, 5, 6, 8, 9

## Lecture 4 — Software Applications

**Definition:** Software applications are programs designed to perform specific tasks for end-users, running on top of the operating system.

### Types of Software
| Type | Example | Role in DevOps |
|------|---------|----------------|
| **System Software** | OS (Linux, Windows), Drivers | The foundation everything runs on |
| **Application Software** | Nginx, Apache, MySQL, Jenkins | The tools we install, configure, and manage |
| **Middleware** | Message queues (RabbitMQ), API gateways | Connects different applications |

---

## Lecture 5 — DevOps CI/CD (⭐ Interview Critical)

### What is CI (Continuous Integration)?
- Developers frequently merge code changes into a **shared repository** (multiple times a day).
- Each merge triggers an **automated build** and **automated tests**.
- **Goal:** Catch bugs early, before they reach production.

### What is CD? (Continuous Delivery vs Continuous Deployment)

| Concept | Meaning |
|---------|---------|
| **Continuous Delivery** | Code is automatically built, tested, and **prepared** for release. Deployment to production is **manual** (one-click approval). |
| **Continuous Deployment** | Every change that passes the pipeline is **automatically deployed** to production. **No human intervention** is required. |

### The CI/CD Pipeline Stages
Source Code → Build → Test → Package → Deploy → Monitor

---

## Lecture 6 — Virtualization

### Virtual Machines (VMs) vs Containers
| Feature | Virtual Machine (VM) | Container |
|---------|----------------------|-----------|
| **Abstraction** | Hardware (CPU, RAM, Disk) | Operating System |
| **Guest OS** | Each VM has its own full OS | Shares the host OS kernel |
| **Boot Time** | Minutes | Seconds |
| **Resource Overhead** | Heavy (GBs) | Lightweight (MBs) |

---

## Lecture 8 — Scripting and Programming

### Compiled vs Interpreted Languages
| Type | Compilation | Examples | DevOps Use |
|------|-------------|----------|------------|
| **Compiled** | Converted to machine code *once* | Go, Java, C++ | High-performance tools |
| **Interpreted** | Executed line-by-line at runtime | Python, Ruby, JavaScript | Automation scripts |

---

## Lecture 9 — Software Development Life Cycle (SDLC)

### Waterfall vs Agile vs DevOps
| Aspect | Waterfall | Agile | DevOps |
|--------|-----------|-------|--------|
| **Approach** | Sequential | Iterative (Sprints) | Continuous |
| **Focus** | Planning & Design | Development & Feedback | Deployment & Operations |
| **Deployment Frequency** | Months/Years | Weeks | Hours/Days |

---

*Theory notes completed: 2026-08-29*
