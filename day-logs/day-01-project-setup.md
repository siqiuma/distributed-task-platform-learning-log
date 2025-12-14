# 📓 Learning Journal — Backend SDE Project

**Date:** 2025-12-13  
**Project:** Distributed Task Platform  
**Lesson:** 1 — Environment Setup & Tooling  

---

## 🎯 Objective

Establish a **production-quality Java backend development environment** that is stable, modern, and aligned with real-world backend systems, so future work can focus on engineering problems rather than tooling issues.

---

## 🛠 Work Completed

### 1. System Architecture Alignment
- Confirmed development machine architecture is **ARM64 (Apple Silicon)**
- Identified and fixed architecture mismatches caused by:
  - Intel (x86_64) Java installations
  - Old Maven versions
  - Rosetta-based terminal execution

This prevents subtle runtime issues later (e.g., Netty, Kafka, Docker).

---

### 2. Java Runtime Setup
- Installed **Temurin (Eclipse Adoptium) OpenJDK 21 (ARM64)**
- Configured `JAVA_HOME` globally via `.zshrc`
- Verified Java runtime version and architecture via terminal

**Key takeaway:**  
> Java version, architecture, and build tools must be aligned early to avoid hard-to-debug issues later.

---

### 3. Maven Modernization
- Discovered system Maven (3.8.1) was running with Java 13 (x86_64)
- Added **Maven Wrapper (`mvnw`)** to the project
- Pinned Maven version to **3.9.9**

Benefits:
- Reproducible builds
- CI-friendly setup
- No dependency on local Maven installation

**Decision:**  
> Always use `./mvnw` instead of `mvn` for this project.

---

### 4. Spring Boot Project Initialization
- Created Spring Boot 3.x project with:
  - Java 21
  - Maven
  - Spring Web
  - Spring Data JPA
  - Spring Boot Actuator
  - H2 (local development)

- Added missing dependency:
  - `spring-boot-starter-validation`

Validated setup by running:
```bash
./mvnw clean test
