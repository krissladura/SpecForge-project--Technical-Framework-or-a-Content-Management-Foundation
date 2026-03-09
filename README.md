
SpecForge (SF) Car Management Engine
I. Technical Motive
The SpecForge Automotive Engine is a backend architectural framework designed to solve the problem of Data Entropy in vehicle inventory systems. Unlike standard CRUD (Create, Read, Update, Delete) applications, this engine treats vehicle data as a Strict State Machine.

The primary objective is to decouple the Business Logic (The Specification) from the Persistence Layer (The Database). This ensures that only "Pure State" data—scrubbed of security vulnerabilities and logical inconsistencies—is ever committed to memory [cite: 2026-03-09].

II. System Logic & Architecture
The system operates on a Tiered Validation Stack:

Schema Enforcement: Using SpecForge logic to define mandatory fields (VIN, Mileage, Pricing) before object instantiation.

State Transition Logic: Automates the lifecycle of a vehicle entity:

IN_TRANSIT -> INSPECTION -> READY_FOR_SALE -> RESERVED -> SOLD.

Integrity Hashing: Every record generates a unique SHA-256 hash based on its metadata. If the database is tampered with externally, the hash mismatch triggers a security flag [cite: 2026-01-27, 2026-03-09].

III. Core Features (No Cliches)
Deterministic VIN Validation: Rejects any input not matching the ISO 3779 standard (17-character alphanumeric).

Financial Precision Layer: Handles currency as integers (in cents/stotinki) to avoid the floating-point errors common in mid-tier retail software [cite: 2026-03-09].

Role-Based Access Control (RBAC): Integrated logic for SUPER_ADMIN, DEALER, and VIEWER permissions.

IV. Cybersecurity Specifications
This engine is built with a Zero-Trust mindset:

Input Sanitization: All string inputs are stripped of special characters to prevent SQL Injection and Cross-Site Scripting (XSS).

UUID Persistence: Uses version 4 UUIDs instead of auto-incrementing integers to prevent ID Enumeration attacks [cite: 2026-03-09].

Rate Limiting Logic: Prevents "Lobby-hopping" style brute-force attacks on the inventory API.

V. Development Setup
Environment: Requires Python 3.10+ (SoftUni Standard) [cite: 2026-01-27].

Logic Injection: Define your specs in specs/vehicle_def.py.

Execution: Run the internal test suite to verify the state machine transitions.
