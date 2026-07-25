# 🏛️ 01: WSO2 Identity Server (IDP) — Production Requirements & Customization Specification

**System Name:** Government of Sint Maarten Enterprise Identity Provider (IDP)  
**Core Technology:** WSO2 Identity Server 7.x  
**Jurisdiction:** Land Sint Maarten (Kingdom of the Netherlands)  
**Standard Assurance:** EU eIDAS Level High & UN/CEFACT Identity Framework  

---

## 1. Overview & Architecture Scope

WSO2 Identity Server (WSO2 IS) functions as the Centralized Identity Provider (IDP), Authentication Gateway, and Single Sign-On (SSO) Controller for Sint Maarten’s e-Government ecosystem. WSO2 IS handles protocol federation (OIDC / SAML2), Relying Party (RP) onboarding, Adaptive Multi-Factor Authentication (MFA), user consent enforcement, and token issuance.

```
+---------------------------------------------------------------------------------+
|                          WSO2 IDENTITY SERVER (IDP)                             |
|                                                                                 |
|  +---------------------+   +-----------------------+   +---------------------+  |
|  | OIDC / SAML2 Gateway |   | Adaptive MFA Engine   |   | Consent Controller  |  |
|  +---------------------+   +-----------------------+   +---------------------+  |
|  | Per-RP Claim Filter |   | Asymmetric JWT Signer |   | Audit Event Logger  |  |
|  +---------------------+   +-----------------------+   +---------------------+  |
+---------------------------------------------------------------------------------+
```

---

## 2. Detailed Functional Requirements

### 2.1 Protocol Federation & Onboarding (IDP-F01, IDP-F02)
* **IDP-F01.1 Protocol Support:** WSO2 IS must support OpenID Connect (OIDC 1.0 Core & Discovery) and SAML 2.0 Web Browser SSO profiles for all Relying Parties (RPs).
* **IDP-F01.2 Client Secret Management:** Client secrets must be generated with high entropy (minimum 256-bit) and stored using salted bcrypt or PBKDF2 hashing.
* **IDP-F02.1 Dynamic & Static RP Registration:** Provide an administrative interface and REST API (`/api/identity/config-mgt/v1.0`) for onboarding new RPs.
* **IDP-F02.2 Per-RP Redirect URI Validation:** Strict regex matching on registered `redirect_uri` values. Wildcard redirect URIs are strictly prohibited in production.

### 2.2 Claim Dialects & Filtering Engine (IDP-F09)
WSO2 IS must enforce attribute release policies under the `http://wso2.org/claims` dialect to ensure RPs receive only entitled attributes.

#### Sint Maarten Custom Claim Mapping:
* `http://wso2.org/claims/bsn_number` ➔ Sint Maarten CRIB / Citizen ID Number (9-digit BSN format)
* `http://wso2.org/claims/uin` ➔ Unique Identification Number (`UN-BGD-xx-xxxxxx-x`)
* `http://wso2.org/claims/eid` ➔ Electronic ID Card Serial Number
* `http://wso2.org/claims/municipality` ➔ Sint Maarten District (*Philipsburg, Simpson Bay, Cole Bay, Cul de Sac, Lower & Upper Prince's Quarter, Mahoe*)

#### Entitlement Tiers:
1. **Tier 1 (Basic RP - e.g., News/Portal):** `sub`, `given_name`, `family_name`, `email`
2. **Tier 2 (High Assurance - e.g., Tax/Revenue):** Tier 1 + `bsn_number`, `permanent_address`, `postcode`, `municipality`
3. **Tier 3 (Maximum Assurance - e.g., Civil Status/Health):** Tier 2 + `uin`, `eid`, `dob`, `gender`, `place_of_birth`, `phone`

### 2.3 Adaptive MFA Policy Controller (IDP-F04, IDP-F07)
WSO2 IS controls *when* and *how* Multi-Factor Authentication is triggered using Adaptive Authentication JavaScript execution scripts.

```javascript
/**
 * Production Adaptive MFA Policy Script for Sint Maarten IDP
 */
function onLoginRequest(context) {
    executeStep(1, {
        onSuccess: function (context) {
            var serviceProvider = context.serviceProviderName;
            var isMFAEnabled = context.currentKnownUser.remoteClaims["http://wso2.org/claims/mfa_enabled"];
            
            // Rule 1: High-Risk RPs (Tax & Healthcare) MANDATE Step 2 TOTP
            if (serviceProvider === "sintmaarten-tax-portal" || serviceProvider === "sintmaarten-health-portal") {
                Log.info("MFA Mandatory for High-Assurance RP: " + serviceProvider);
                executeStep(2);
            } 
            // Rule 2: User Preference Step-Up MFA
            else if (isMFAEnabled === "true") {
                Log.info("MFA Triggered by User Profile Preference");
                executeStep(2);
            }
        }
    });
}
```

### 2.4 User Consent & Privacy Prompt (IDP-F05)
* **IDP-F05.1 Explicit Consent UI:** Present an un-bypassable consent modal listing exact requested claims prior to token issuance.
* **IDP-F05.2 Revocation Management:** Provide a self-service portal (`https://idp.sintmaartengov.org/myaccount`) allowing citizens to view and revoke active RP consents.

### 2.5 Single Sign-On (SSO) & Single Logout (SLO) (IDP-F08, IDP-F12)
* **IDP-F08.1 Session Sharing:** Share authenticated session state via encrypted session cookies across all RPs within a single browser session.
* **IDP-F12.1 Front-Channel & Back-Channel Logout:** Support OIDC Front-Channel Logout 1.0 and Back-Channel Logout 1.0.
* **IDP-F12.2 Prompt Bypass:** Support `prompt_on_logout = false` in `deployment.toml` to prevent redundant confirmation dialogs when `id_token_hint` is provided.

---

## 3. Production Non-Functional Requirements (NFRs)

### 3.1 Security & Cryptography (IDP-NF01, IDP-NF03)
* **IDP-NF01.1 Asymmetric Token Signing:** Tokens must be signed using **RS256** (RSA 2048-bit + SHA-256) or **ES256** (ECDSA P-256). Secret-based HMAC algorithms (HS256) are prohibited for public token issuance.
* **IDP-NF01.2 Automated Key Rotation:** Key Rotation Policy must rotate asymmetric signing keys every 90 days with a 14-day grace overlap published via JSON Web Key Set (`/.well-known/jwks.json`).
* **IDP-NF03.1 Transport Security:** All endpoints require **TLS 1.3** (or TLS 1.2 minimum with AEAD cipher suites: `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`).

### 3.2 High Availability & SLA (IDP-NF02)
* **IDP-NF02.1 SLA Guarantee:** High Availability active-active deployment across 2+ availability zones targeting **99.9% uptime** (< 8.76 hours downtime/year).
* **IDP-NF02.2 Throughput:** Support p95 latency < 300ms for OAuth token issuance under a load of 1,000 peak requests/sec.

### 3.3 Audit Logging & Governance (IDP-NF04, IDP-F11)
Every event must output structured JSON to `/repository/logs/audit.log` containing:
* `Timestamp` (ISO 8601 UTC)
* `CorrelationID` (`X-Correlation-ID` header value)
* `TenantDomain`
* `ServiceProviderName`
* `SubjectUUID`
* `EventType` (`AUTHENTICATION_SUCCESS`, `AUTHENTICATION_FAILURE`, `MFA_CHALLENGE`, `CONSENT_GRANTED`, `CONSENT_REVOKED`)
* `ClientIP` & `UserAgent`

---

## 4. Production `deployment.toml` Configuration File

```toml
[server]
hostname = "idp.sintmaartengov.org"
node_ip = "127.0.0.1"

[authentication.endpoints]
prompt_on_logout = false

[oauth.extensions]
token_issuer = "org.wso2.carbon.identity.oauth2.token.JWTTokenIssuer"

[oauth.jwks]
url = "https://idp.sintmaartengov.org/oauth2/jwks"

[keystore.primary]
file_name = "repository/resources/security/wso2carbon.jks"
type = "JKS"
alias = "wso2carbon"
key_alg = "RSA"

[identity_mgt]
events.listeners = ["org.wso2.carbon.identity.governance.listener.IdentityMgtEventListener"]

[transport.https.sslReport]
enable = true
tls_protocols = ["TLSv1.2", "TLSv1.3"]
```
