# 📋 04: Master Production Development Checklist

**System Name:** Government of Sint Maarten eID & WSO2 Ecosystem  
**Target Environment:** Production (Sint Maarten Government Cloud / Kubernetes)  
**Assurance Standard:** EU eIDAS Level High & UN/CEFACT Identity Framework  

---

## 1. WSO2 Identity Server (IDP) Setup & Customization Checklist

### 1.1 Deployment & Core Configuration (`IDP-F01, IDP-NF01, IDP-NF03`)
- [ ] **Deployment Topology:** Deploy containerized WSO2 IS 7.x in active-active high-availability configuration across 2+ availability zones.
- [ ] **Transport Security:** Enforce TLS 1.3 (or TLS 1.2 minimum with AEAD cipher suites).
- [ ] **Asymmetric Token Signing:** Configure RS256 (RSA 2048-bit + SHA-256) keypair in `wso2carbon.jks`.
- [ ] **JWKS Endpoint:** Publish public keys via `/.well-known/jwks.json` with 90-day automated rotation policy.
- [ ] **Single Logout (SLO):** Set `prompt_on_logout = false` in `deployment.toml` for seamless OIDC logout prompt bypass when `id_token_hint` is provided.

### 1.2 Claim Dialects & RP Onboarding (`IDP-F02, IDP-F09`)
- [x] **Custom Dialects:** Map Sint Maarten claims under `http://wso2.org/claims`:
  - [x] `http://wso2.org/claims/bsn_number` (CRIB / Citizen ID Number)
  - [x] `http://wso2.org/claims/uin` (Unique Identification Number)
  - [x] `http://wso2.org/claims/eid` (Electronic ID Serial Number)
  - [x] `http://wso2.org/claims/municipality` (Sint Maarten District)
- [x] **Service Provider Registration:** Register RPs (`mygov-portal`, `tax-portal`, `health-portal`, etc.) with strict regex `redirect_uri` validation (no wildcards).
- [ ] **Claim Filtering Tiers:** Enforce least-privilege claim release policies per RP entitlement tier.

### 1.3 Adaptive MFA & Consent Management (`IDP-F04, IDP-F05, IDP-F07, IDP-F11`)
- [x] **Adaptive MFA Scripting:** Attach JavaScript adaptive auth scripts to enforce Step 2 TOTP for high-risk RPs (`tax-portal`, `health-portal`).
- [ ] **Explicit Consent UI:** Render un-bypassable consent prompt displaying exact requested claims prior to issuing tokens.
- [ ] **Audit Event Logger:** Configure structured JSON audit logging to `/repository/logs/audit.log` containing `CorrelationID`, `TenantID`, `SPName`, `SubjectUUID`, `EventType`, and `ClientIP`.

---

## 2. eID Platform (.NET 8 C# Web API) Build Checklist

### 2.1 Solution Scaffold & Security (`EID-F01, EID-NF01`)
- [x] **Solution Structure:** Create ASP.NET Core 8 Web API projects (`SintMaarten.EID.Api`, `Core`, `Infrastructure`, `Services`).
- [x] **JWKS Token Validation:** Implement Bearer token validation middleware against WSO2 IS JWKS endpoint.
- [x] **System Key Middleware:** Build `X-EID-System-Key` header authentication for secure server-to-server microservice lookups.

### 2.2 Civil Registry UIN Resolution & Stub Engine (`EID-F02, EID-F03, EID-F04, EID-F08`)
- [x] **Citizen Resolution Engine:** Implement resolution lookup by 9-digit CRIB / Citizen ID or UIN (`UN-BGD-xx-xxxxxx-x`).
- [x] **Error Taxonomies:** Standardize error responses (`REGISTRY_NOT_FOUND`, `IDENTITY_MISMATCH`, `BIOMETRIC_FAILED`).
- [x] **Mock Registry Engine:** Implement mock registry stub for offline development and sandbox testing.

### 2.3 Polly Resilience & Fault Tolerance (`EID-F10, EID-NF04`)
- [x] **Retries Policy:** Configure Polly exponential backoff retries (3 attempts).
- [x] **Circuit Breaker Policy:** Configure circuit breaker to trip if 50% of requests fail over a 10s sampling window.
- [x] **Timeout Policy:** Set 3-second hard timeout for all external Civil Registry calls.

### 2.4 Governance & Audit Logging (`EID-F07, EID-NF05, EID-NF06`)
- [x] **Annex E Cryptographic Logger:** Build `AnnexEEvidenceLogger` generating SHA-256 digital digests of transactions.
- [x] **Correlation ID Middleware:** Extract or generate `X-Correlation-ID` headers and propagate across all downstream calls.
- [x] **PII Log Masking:** Enforce masking policy (`UN-BGD-***-1`) to prevent plaintext PII writing to application logs.

---

## 3. Relying Party (`mygov.test`) Integration Checklist

### 3.1 OIDC Integration & Callback (`SVC-F01, SVC-F02, SVC-F03`)
- [ ] **Unauthenticated Redirect:** Redirect unauthenticated users to WSO2 IS OIDC authorize endpoint.
- [x] **Token Exchange (`Wso2Controller.php`):** Exchange authorization code for tokens and store enriched `$user_profile` in Laravel session.
- [ ] **Single Logout Handling:** Handle session termination and state cleanup on logout.

### 3.2 Registration & User Experience (`SVC-F04, SVC-F05`)
- [ ] **Registration Entry Point:** Place `🇸🇽 Register Your Sint Maarten eID` button on `mygov.test/login`.
- [x] **4-Step Wizard:** Render registration wizard collecting CRIB/BSN, demographics, Sint Maarten district, and document uploads (Passport photo + ID scan).
- [x] **Authentication Options:** Enable Password, Magic Link, FIDO2/Touch ID Passkey, and Google Authenticator TOTP QR code options.

---

## 4. Production Hardening & Compliance Checklist

### 4.1 Security & Quality Assurance
- [x] **OWASP Top 10 Audit:** Verify protection against SQL injection, XSS, CSRF, and broken access control.
- [x] **Secret Scrubbing:** Ensure zero passwords or private keys are hardcoded in source repositories.
- [x] **Load Testing:** Benchmark performance to ensure p95 latency < 300ms under 1,000 peak req/sec load.
- [x] **Documentation Verification:** Verify all API contracts and Swagger/OpenAPI documentation.
