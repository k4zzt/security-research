# Publicly Accessible `env.json` Exposing Unrestricted Google Maps API Key

<p align="center">
  <strong>Configuration Exposure & API Key Misconfiguration</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackerOne-black?style=flat-square">
  <img src="https://img.shields.io/badge/Category-Information%20Disclosure-red?style=flat-square">
  <img src="https://img.shields.io/badge/Technology-Google%20Maps%20API-blue?style=flat-square">
  <img src="https://img.shields.io/badge/Status-Duplicate-lightgrey?style=flat-square">
</p>

---

<h3 align="center">Report Information</h3>

<table align="center">
  <tr>
    <th>Field</th>
    <th>Details</th>
  </tr>
  <tr>
    <td><strong>Platform</strong></td>
    <td>HackerOne</td>
  </tr>
  <tr>
    <td><strong>Category</strong></td>
    <td>Information Disclosure / API Key Exposure</td>
  </tr>
  <tr>
    <td><strong>Technology</strong></td>
    <td>Google Maps API</td>
  </tr>
  <tr>
    <td><strong>Exposed Resource</strong></td>
    <td><code>/envs/env.json</code></td>
  </tr>
  <tr>
    <td><strong>Status</strong></td>
    <td>Duplicate</td>
  </tr>
</table>

---

## Overview

During a reconnaissance sweep for publicly accessible configuration files, I identified that the `/envs/` directory was improperly protected.

The file `env.json` was publicly accessible and exposed the production configuration for the `pyme-web` environment.

Among the exposed configuration values was a Google Maps API key.

The exposed key was not restricted by HTTP referrer and was authorized to access the **Distance Matrix API**, allowing external requests to be performed using the exposed credential.

The configuration file also disclosed internal application architecture, including references to microservices, authentication-related functionality and telemetry infrastructure.

---

## Vulnerable Resource

```text
https://empresa.bancoplata.mx/envs/env.json
```

The publicly accessible file exposed configuration values including a Google Maps API key.

For security reasons, the credential is redacted in this write-up:

```text
GoogleMaps.ApiKey = [REDACTED]
```

---

## Technical Analysis

### 1. Publicly Accessible Configuration

The configuration file could be accessed directly without authentication:

```bash
curl https://empresa.bancoplata.mx/envs/env.json
```

The response contained production configuration data belonging to the `pyme-web` environment.

This included the Google Maps API key as well as additional configuration values revealing information about the application's internal architecture.

---

### 2. API Key Restrictions

The exposed Google Maps API key did not enforce HTTP referrer restrictions.

As a result, the credential could be used from an external environment rather than being restricted to the intended application origin.

The key could therefore be supplied to Google Maps API requests originating outside the legitimate application.

---

### 3. Distance Matrix API

During testing, the exposed credential was used to make a request to the Distance Matrix API from an external environment.

The request returned a successful response:

```http
HTTP/1.1 200 OK
```

Conceptually:

```text
External Client
      │
      │ Exposed API Key
      ▼
Google Maps API
      │
      ▼
Distance Matrix API
      │
      ▼
Successful response
```

This demonstrated that the exposed credential was functional and could be used externally.

The original PoC used a Distance Matrix request with an origin and destination to verify that the key was accepted by the API.

The credential itself is intentionally omitted from this public write-up.

---

## Additional Information Disclosure

The `env.json` file exposed more than the Google Maps credential.

The configuration revealed aspects of the application's internal architecture, including references to:

* Internal microservices;
* Invoice management functionality;
* Authentication-related flows;
* Telemetry infrastructure;
* Sentry configuration.

One of the identified services was associated with:

```text
pymeFacturaManager
```

These values provided additional information about the internal service architecture and could assist further reconnaissance.

---

## Security Controls Assessment

As part of the investigation, I tested internal API endpoints referenced by the exposed configuration.

The identified gateways correctly enforced **Role-Based Access Control (RBAC)** and returned authorization errors when accessed without the required privileges.

This indicated that the internal APIs themselves had authorization controls in place.

The issue therefore centered specifically on the public exposure of the configuration file and the unrestricted Google Maps credential.

This is an important distinction:

```text
Internal APIs
     │
     └── RBAC enforced ✓


Configuration file
     │
     └── Publicly accessible ✗


Google Maps API key
     │
     └── External use allowed ✗
```

---

## Proof of Concept

The following video demonstrates the discovery and validation of the exposed configuration and Google Maps API key.

https://github.com/user-attachments/assets/8fcb7043-8f73-4c41-9f4f-131418423fcf


### PoC


The demonstration shows the configuration file being accessed externally and the exposed credential being used to perform a Google Maps API request.

---

## Impact

### Financial Abuse

An unrestricted API key can potentially be abused by third parties to consume the application's Google Cloud resources.

If an API associated with the credential is billed based on usage, unauthorized requests can result in unexpected costs for the account owner.

The Distance Matrix API was specifically validated during testing.

---

### Service Interruption

Unauthorized consumption of available quotas could potentially affect legitimate application functionality if configured quotas are exhausted.

This could result in legitimate requests being rejected once applicable limits are reached.

---

### Infrastructure Disclosure

The exposed configuration file also disclosed information about the application's internal architecture.

This included references to internal services and application components, providing additional information that could be useful during reconnaissance.

---

## Recommended Mitigation

### 1. Restrict the API Key

Apply appropriate application and API restrictions to the Google Maps credential.

For browser-based usage, configure HTTP referrer restrictions to limit where the key can be used.

---

### 2. Restrict Enabled APIs

Only enable the Google APIs and services that are actually required by the application.

High-cost or unnecessary APIs should not be enabled for a client-side credential.

---

### 3. Remove Public Access to Configuration Files

The `/envs/` directory and production configuration files should not be publicly accessible.

Web server or application-level access controls should prevent direct access to sensitive configuration files.

---

### 4. Rotate the Exposed Credential

Any credential exposed through a publicly accessible configuration file should be considered compromised and rotated.

---

## Responsible Disclosure

The vulnerability was responsibly reported through HackerOne.

After review, the report was closed as **Duplicate**.

The response stated that the same exposed configuration file and identical Google Maps API key had already been reported in another submission.

The duplicate report was identified as:

**HackerOne report #3642984**

The platform determined that both reports described the same publicly accessible `env.json` configuration file and the same exposed Google Maps API key.

---

## Conclusion

This research identified a publicly accessible production configuration file containing an unrestricted Google Maps API key.

The credential could be used from an external environment and was confirmed to successfully access the Distance Matrix API.

The same configuration file also disclosed information about internal application architecture and service components.

Although the report was ultimately classified as **Duplicate**, the investigation demonstrated a complete vulnerability research workflow:

```text
Reconnaissance
      ↓
Configuration File Discovery
      ↓
Sensitive Data Identification
      ↓
Credential Validation
      ↓
External Abuse Testing
      ↓
Impact Assessment
      ↓
Responsible Disclosure
```

The case provided practical experience in configuration exposure, API security, credential validation, information disclosure and vulnerability research.
