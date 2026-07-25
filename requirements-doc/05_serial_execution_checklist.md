# 📋 05: Serial Execution Checklist (eID & WSO2 IS Ecosystem)

This checklist outlines the exact, step-by-step tasks to complete the remaining phases of the Sint Maarten Federated eID & WSO2 SSO system.

---

## 🚀 Phase 1: WSO2 Identity Server Configuration

- [x] **1.1 Custom Claim Dialects Mapping:**
  - [x] Add `http://wso2.org/claims/bsn_number` mapping to capture CRIB numbers.
  - [x] Add `http://wso2.org/claims/uin` mapping for the Unique Identification Number.
  - [x] Add `http://wso2.org/claims/eid` mapping for the Electronic ID serial number.
  - [x] Add `http://wso2.org/claims/municipality` mapping for citizen districts.
- [x] **1.2 Service Provider Registrations:**
  - [x] Configure Service Providers in WSO2 for all 6 target portals (`mygov`, `upsheba`, `fintech`, `health`, `passport`, `tax`).
  - [x] Map client callback redirect URIs strictly with exact domains (no wildcards).
- [x] **1.3 Adaptive JavaScript MFA Policy:**
  - [x] Attach adaptive auth scripts in WSO2 to enforce Step 2 verification for high-risk portals (`tax-portal`, `health-portal`).
  - [x] Configure tenant logs to output OIDC assertion mapping events to `/repository/logs/audit.log`.

---

## 💻 Phase 2: Next.js Frontend Integration (`eid-frontend`)

- [x] **2.1 API Endpoint Target Alignment:**
  - [x] Update client fetch parameters and environment configs to route all API calls to the new .NET 8 backend (`http://localhost:8082`).
- [x] **2.2 Multi-Step OIDC Login & 2FA View:**
  - [x] Modify `src/app/oauth/authorize/page.tsx` to handle multi-step login transitions (`CREDENTIALS` ➔ `MFA_CHALLENGE`).
  - [x] Render 6-digit OTP verification card with countdown resend timer.
  - [x] Link submission forms to submit `otp` payload to the backend OIDC authorization endpoints.
- [x] **2.3 Wizard Submission Routing:**
  - [x] Hook step 4 (biometric upload + document verification) of the registration wizard to submit JSON payloads to `.NET` citizen registration endpoint.

---

## 🔑 Phase 3: Passwordless FIDO2 & WebAuthn Integration

- [x] **3.1 Backend WebAuthn API Handlers:**
  - [x] Implement FIDO2 registration challenge endpoints (`POST /api/v1/webauthn/register/challenge`) in `.NET`.
  - [x] Implement FIDO2 assertion verification endpoints (`POST /api/v1/webauthn/login/verify`).
- [x] **3.2 Frontend Touch ID / Face ID Prompts:**
  - [x] Add navigator credentials prompt wrappers in eID Frontend to trigger browser biometrics.
  - [x] Build a "Security Keys & Passkeys" tab in the Citizen Dashboard for device registration.

---

## 🇸🇽 Phase 4: Relying Party Dashboard & Session Completion

- [x] **4.1 Single Logout (SLO) Handlers:**
  - [x] Bind logout route endpoints in MyGov Laravel controller to clean sessions and call WSO2 IS logout endpoints.
- [x] **4.2 Claims Dashboard UI:**
  - [x] Design and map custom Sint Maarten citizen profile dashboard views in MyGov displaying CRIB number, district, and eID photo.

---

## 🛡️ Phase 5: Production Hardening & Compliance Auditing

- [x] **5.1 OWASP Top 10 Auditing:**
  - [x] Audit all input parameters for SQLi, XSS, and CSRF protection.
  - [x] Verify CORS policy origins list to ensure production constraints.
- [x] **5.2 Load Testing:**
  - [x] Run benchmark scripts using Locust/k6 to verify latency remains < 300ms under 1,000 req/sec load.
