# 🪪 02: eID Platform (.NET 8) & Integration — Production Requirements Specification

**System Name:** Sint Maarten National eID Platform Orchestrator  
**Core Technology:** ASP.NET Core 8 Web API (.NET C#)  
**Jurisdiction:** Land Sint Maarten (Kingdom of the Netherlands)  
**Standard Assurance:** EU eIDAS Level High & UN/CEFACT Identity Framework  

---

## 1. Overview & Architecture Scope

The eID Platform (.NET 8 C# Web API) acts as the high-assurance identity resolution and orchestration vault. It interfaces between WSO2 IS (IDP), the external Civil Registry (UIN system of record), and Relying Parties (RPs), maintaining Annex E cryptographic audit evidence, Polly resilience circuit breakers, and configuration-driven claim transformations.

```
+-------------------+             +------------------------+             +-------------------------+
| Relying Party (RP)| <=========> | WSO2 Identity Server   | <=========> | eID Platform (.NET 8)   |
| (mygov.test)      |    OIDC     | (IDP / SSO Gateway)    |   REST/OIDC | (Identity Orchestrator) |
+-------------------+             +------------------------+             +-------------------------+
                                                                                      |
                                                                               REST / mTLS
                                                                                      v
                                                                         +-------------------------+
                                                                         | Civil Registry (UIN)    |
                                                                         | (External System)       |
                                                                         +-------------------------+
```

---

## 2. Detailed Functional Requirements

### 2.1 Token Validation & Access Control (EID-F01)
* **EID-F01.1 Bearer JWT Validation:** Validate token signature using WSO2 IS JWKS endpoint (`/oauth2/jwks`), checking `iss` (issuer), `aud` (audience), and expiration (`exp`).
* **EID-F01.2 System Key Authentication:** Allow authorized internal microservices to authenticate via `X-EID-System-Key` header with high-entropy pre-shared secrets.

### 2.2 Civil Registry UIN Resolution & Stub Engine (EID-F02, EID-F03, EID-F04, EID-F08)
* **EID-F03.1 Citizen Resolution:** Query the Civil Registry using 9-digit CRIB / Citizen ID numbers or UIN string (`UN-BGD-xx-xxxxxx-x`).
* **EID-F05.1 Taxonomies & Error Taxonomy:** Return standardized error codes (`REGISTRY_NOT_FOUND`, `IDENTITY_MISMATCH`, `BIOMETRIC_FAILED`) when matching fails.
* **EID-F08.1 Mock/Stub Registry Engine:** Provide a configurable mock registry engine for local development and sandbox testing before production registry interfaces go live.

### 2.3 Per-RP Claim Transformation (EID-F06, EID-F11, EID-F12)
* **EID-F06.1 Claim Transformation:** Map raw Civil Registry payloads into schema-specific JSON response models for each RP tier.
* **EID-F12.1 Configuration-Driven Mapping:** Claim transformations must be driven via `appsettings.json` or database configurations — **zero code re-deployment required** for onboarding new RPs.

### 2.4 Annex E Cryptographic Evidence & Audit Logger (EID-F07)
* **EID-F07.1 Immutable Transaction Logging:** Generate an Annex E compliant audit record for every transaction (success or failure).
* **EID-F07.2 Cryptographic Integrity:** Compute a SHA-256 digital digest over the identity assertion, timestamp, subject ID, and correlation ID.

---

## 3. High-Level Data Contracts (Section 5 Spec)

### 3.1 WSO2 IDP ➔ eID Platform (`POST /api/v1/oauth/authorize`)
```json
{
  "correlation_id": "corr-sx-2026-991204a",
  "client_id": "sintmaarten-tax-portal",
  "redirect_uri": "http://mygov.test/login",
  "state": "xyz123",
  "auth_context": {
    "mfa_enforced": true,
    "consent_granted": true,
    "timestamp": "2026-07-21T16:35:00Z"
  }
}
```

### 3.2 eID Platform ➔ Civil Registry (UIN Request Payload)
```json
{
  "correlation_id": "corr-sx-2026-991204a",
  "search_type": "CRIB_OR_UIN",
  "identifier": "123456782",
  "requested_attributes": ["given_name", "family_name", "dob", "district", "bsn_number"]
}
```

### 3.3 eID Platform ➔ Relying Party / WSO2 (Mapped Mapped Claim Output)
```json
{
  "correlation_id": "corr-sx-2026-991204a",
  "evidence_reference_id": "EVID-ANNEX-E-2026-88391029",
  "verification_status": "VERIFIED_HIGH_ASSURANCE",
  "citizen": {
    "uuid": "baac1f67-dae8-4df2-9fd9-3e70f5b5d1f0",
    "uin": "UN-BGD-26-219920-1",
    "eid": "EID-BGD-26-019026-9",
    "bsn_number": "123456782",
    "full_name": "Ibrahim Islam",
    "first_name": "Ibrahim",
    "last_name": "Islam",
    "dob": "1994-10-15",
    "municipality": "Philipsburg",
    "country": "Sint Maarten (Kingdom of the Netherlands)",
    "international_standard": {
      "authority": "Government of Sint Maarten • Kingdom of the Netherlands • EU eIDAS & UN/CEFACT Standard",
      "country": "Sint Maarten (Kingdom of the Netherlands)"
    }
  }
}
```

---

## 4. Production Non-Functional Requirements (NFRs)

### 4.1 Resilience & Circuit Breaking with Polly (EID-F10, EID-NF04)
The .NET 8 API must use Microsoft’s `Polly` resilience framework to manage Civil Registry calls:

```csharp
// Program.cs - Polly Circuit Breaker Configuration
builder.Services.AddHttpClient<ICivilRegistryClient, CivilRegistryClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["CivilRegistry:BaseUrl"]!);
    client.Timeout = TimeSpan.FromSeconds(3);
})
.AddStandardResilienceHandler(options =>
{
    options.Retry.MaxRetryAttempts = 3;
    options.Retry.BackoffType = DelayBackoffType.Exponential;
    options.CircuitBreaker.FailureRatio = 0.5;
    options.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(10);
    options.CircuitBreaker.MinimumThroughput = 8;
});
```

### 4.2 End-to-End Correlation ID Tracing (EID-NF05)
* **EID-NF05.1 `X-Correlation-ID` Propagation:** Read `X-Correlation-ID` header from incoming requests or auto-generate a UUIDv4. Pass this header to all downstream Registry calls and log messages.

### 4.3 PII Data Protection & Security (EID-NF01, EID-NF02)
* **EID-NF02.1 Zero Plaintext PII Logging:** PII attributes (UIN, DOB, BSN, Address) MUST NEVER be logged in plaintext application logs. Logs must contain masked identifiers (`UN-BGD-***-1`).
