# Testing Architecture by Environment (SMART on FHIR / Epic)

This document describes the testing architecture for the application across **Local**, **Dev**, and **Staging** environments.

Each section includes:
- A Mermaid diagram for that environment
- A **Nodes** section listing any *new* nodes introduced in that environment
- A note explaining **dotted vs solid arrows**

---

## Arrow semantics

- **Solid arrows (`-->`)**  
  Default, expected execution paths for that environment.

- **Dotted arrows (`-.->`)**  
  Optional, gated, or non-default paths (e.g. canary tests, fallbacks, or explicitly enabled behavior).

---

# Local environment

**Intent:** fast, deterministic development and testing with no dependency on external systems. All authentication and EHR interactions are mocked or emulated and fully resettable.

~~~mermaid
flowchart LR;

subgraph EXEC["Local - Test Execution"]
direction TB;
  TestRunner["Test Runner (Local)"];
  Contract["Contract Bundle (FHIR + SMART)"];
end;

subgraph APP["Your Application (Local)"]
direction TB;
  UI["Web/Mobile UI"];
  API["Backend API"];
  AuthBroker["AuthBroker"];
  EHRGateway["EHRGateway (vendor-agnostic)"];
  TestAuthIssuer["TestAuthIssuer (test-only)"];
end;

subgraph EXT["External Systems (Local)"]
direction TB;
  subgraph MOCK["Mocked / Emulated Systems"]
  direction TB;
    Emulator["FHIR + SMART Emulator (resettable)"];
  end;
end;

Clinician["Clinician - EHR Launch"];
Patient["Patient - Web/Mobile App"];

Clinician --> UI;
Patient --> UI;

UI --> API;
API --> AuthBroker;

UI --> TestAuthIssuer;
TestRunner --> TestAuthIssuer;

API --> EHRGateway;
EHRGateway --> Emulator;

TestRunner --> UI;
TestRunner --> API;
TestRunner --> Emulator;

Contract --> Emulator;
Contract --> TestRunner;
~~~

## Nodes (Local)

- **Test Runner**: Executes integration and end-to-end tests locally or in CI.
- **Contract Bundle (FHIR + SMART)**: Canonical specification of expected FHIR and SMART behavior, used by both tests and emulator.
- **TestAuthIssuer**: Test-only component that issues deterministic app-auth tokens, bypassing real IdP flows.
- **FHIR + SMART Emulator**: Resettable local implementation of FHIR APIs and SMART auth semantics, driven by the contract bundle.

---

# Dev environment

**Intent:** emulator-backed testing by default, with explicit opt-in live checks against Epic’s public sandbox to detect drift early.

~~~mermaid
flowchart LR;

subgraph EXEC["Dev - Test Execution"]
direction TB;
  TestRunner["Test Runner (CI/Dev)"];
  Contract["Contract Bundle (FHIR + SMART)"];
  Canary["Canary Suite"];
end;

subgraph APP["Your Application (Dev)"]
direction TB;
  UI["Web/Mobile UI"];
  API["Backend API"];
  AuthBroker["AuthBroker"];
  EHRGateway["EHRGateway (vendor-agnostic)"];
  TestAuthIssuer["TestAuthIssuer (test-only)"];
end;

subgraph AUTH["Authentication (Dev)"]
direction TB;
  AppAuth["App Auth (your IdP)"];
  SmartAuth["SMART on FHIR OAuth/OIDC (Epic AS)"];
end;

subgraph EXT["External Systems (Dev)"]
direction TB;

  subgraph MOCK["Mocked / Emulated"]
  direction TB;
    Emulator["FHIR + SMART Emulator (resettable)"];
  end;

  subgraph REAL["Real External"]
  direction TB;
    EpicSandbox["Epic Public Sandbox"];
  end;
end;

Clinician["Clinician - EHR Launch"];
Patient["Patient - Web/Mobile App"];

Clinician --> UI;
Patient --> UI;

UI --> API;
API --> AuthBroker;
AuthBroker --> AppAuth;

AuthBroker -.-> SmartAuth;

UI --> TestAuthIssuer;
TestRunner --> TestAuthIssuer;

API --> EHRGateway;
EHRGateway --> Emulator;

EHRGateway -.-> EpicSandbox;

TestRunner --> UI;
TestRunner --> API;
TestRunner --> Emulator;

TestRunner -.-> Canary;
Canary -.-> EpicSandbox;

Contract --> Emulator;
Contract --> TestRunner;
~~~

## Nodes (Dev)

- **Canary Suite**: Small, explicitly triggered test suite that runs against the live Epic sandbox to detect API/behavior drift.
- **SMART on FHIR OAuth/OIDC (Epic AS)**: Epic’s authorization server, exercised only via dotted paths in dev.
- **Epic Public Sandbox**: Epic-hosted non-production environment reached only via dotted (opt-in) paths in dev.

---

# Staging environment

**Intent:** validate primarily against the real Epic public sandbox, with emulator fallback only for targeted tests/diagnostics.

~~~mermaid
flowchart LR;

subgraph EXEC["Staging - Test Execution"]
direction TB;
  TestRunner["Test Runner (CI/Staging)"];
  Contract["Contract Bundle (FHIR + SMART)"];
  Canary["Canary Suite"];
end;

subgraph APP["Your Application (Staging)"]
direction TB;
  UI["Web/Mobile UI"];
  API["Backend API"];
  AuthBroker["AuthBroker"];
  EHRGateway["EHRGateway (vendor-agnostic)"];
  TestAuthIssuer["TestAuthIssuer (test-only for app login)"];
end;

subgraph AUTH["Authentication (Staging)"]
direction TB;
  AppAuth["App Auth (your IdP)"];
  SmartAuth["SMART on FHIR OAuth/OIDC (Epic AS)"];
end;

subgraph EXT["External Systems (Staging)"]
direction TB;

  subgraph REAL["Real External"]
  direction TB;
    EpicSandbox["Epic Public Sandbox"];
  end;

  subgraph MOCK["Mocked / Emulated"]
  direction TB;
    Emulator["FHIR + SMART Emulator (resettable)"];
  end;
end;

Clinician["Clinician - EHR Launch"];
Patient["Patient - Web/Mobile App"];

Clinician --> UI;
Patient --> UI;

UI --> API;
API --> AuthBroker;
AuthBroker --> AppAuth;
AuthBroker --> SmartAuth;

UI --> TestAuthIssuer;
TestRunner --> TestAuthIssuer;

API --> EHRGateway;
EHRGateway --> EpicSandbox;

EHRGateway -.-> Emulator;

TestRunner --> UI;
TestRunner --> API;

TestRunner --> Canary;
Canary --> EpicSandbox;

Contract --> TestRunner;
Contract -.-> Emulator;
~~~

## Nodes (Staging)

- **TestAuthIssuer (staging)**: Used only to bypass *your* patient-login IdP flows in tests; never substitutes for SMART auth.
- **Emulator (fallback)**: Available only via dotted paths for targeted tests/diagnostics; not the default staging target.

---

## Summary

- **Local:** fully mocked, resettable, deterministic.
- **Dev:** emulator by default, with opt-in live canary checks.
- **Staging:** real Epic sandbox by default, emulator as a controlled fallback.
