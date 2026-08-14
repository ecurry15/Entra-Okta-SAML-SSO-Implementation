# Entra-Okta-SAML-SSO-Implementation

![Architecture Diagram](https://img.shields.io/badge/Architecture-Entra%20ID%20%7C%20Okta%20%7C%20Salesforce-blue?style=for-the-badge)
![Security](https://img.shields.io/badge/IAM-SAML%202.0%20%7C%20SCIM-orange?style=for-the-badge)

## 📌 Executive Summary
This project demonstrates the design, deployment, and troubleshooting of a multi-platform enterprise identity federation architecture. The objective was to configure **Microsoft Entra ID** as the Primary Identity Provider (IdP), **Okta** as the central Identity Hub / Service Provider (SP), and **Salesforce** as a downstream SaaS application.

The end-to-end flow enables centralized identity lifecycle management and Single Sign-On (SSO): users authenticate via Entra ID, are federated into Okta via custom SAML attribute assertions, and are seamlessly signed into Salesforce through Okta-managed SSO and provisioned accounts.

---

## 🏗️ Technical Architecture & Authentication Flow

![Identity Federation Architecture Diagram](https://mermaid.ink/img/p3e4e1bea8ba57434)

### Authentication Flow:
1. User accesses the **Okta End-User Portal** or **Salesforce App Tile**.
2. Okta evaluates IdP Routing Rules and redirects the browser to **Microsoft Entra ID** for authentication.
3. User authenticates against Entra ID (enforcing primary credentials and policy controls).
4. Entra ID issues a signed SAML 2.0 assertion containing custom user claims (`upn`, `email`, `givenName`, `surname`).
5. Okta validates the SAML response, parses the custom schema mappings, matches the user account, and establishes an Okta session.
6. Okta federates the assertion downstream to **Salesforce**, granting immediate SSO access.

---

## 🛠️ Implementation Phasing

### Phase 1: Directory Setup & User/Group Import
* Provisioned an enterprise **Okta Tenant** and structured organization units.
* Imported identity profiles from **Microsoft Entra ID** into Okta.
* Created granular **Okta User Groups** (e.g., `Salesforce-Users-Group`) to manage app assignments and role-based access controls (RBAC).

### Phase 2: Inbound SAML Federation (Entra ID ➔ Okta)
* Created an Enterprise Application in Entra ID for Okta Inbound Federation.
* Formatted SAML 2.0 endpoints (Issuer URI, Single Sign-On URL, ACS URL).
* Extended the Okta IdP Schema in **Profile Editor** by defining a custom `upn` string attribute (`appuser.upn`).
* Configured attribute mappings between Entra's `user.userprincipalname` and Okta's `user.login`.
* Configured Account Link Policies and enabled automatic user account matching.

### Phase 3: Outbound SAML SSO & Provisioning (Okta ➔ Salesforce)
* Integrated the **Salesforce Integration App** inside Okta.
* Configured SAML 2.0 SSO settings in Salesforce (Issuer, Identity Provider Certificate, SAML Identity Type).
* Enabled domain management in Salesforce and set Okta as the primary authentication service.
* Configured automated lifecycle provisioning to propagate Okta group assignments into Salesforce user accounts.

---

## 🚨 Deep Dive: Troubleshooting Inbound Federation & SAML Claims

During Phase 2 execution, inbound federation failed with the following critical error in Okta:
`FAILURE: User login to Okta via IdP - User Not Found / User Creation Disabled`

```
System Log Entry:
Target: Andre.Iggy@campbell400.onmicrosoft.com
DetailEntry: unknown
Reason: Invalid username transform / Account match failure
```

### Root Cause Analysis
1. **Entra NameID Namespace Lock:** By default, Microsoft Entra ID wraps system `NameID` claims in locked XML namespaces (`http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier`). Okta's simple matching engine failed to extract the raw string, evaluating the incoming subject as `unknown`.
2. **Schema Mismatch:** Okta's native IdP schema did not possess a parameter for raw `upn` strings, triggering `invalid property upn` transform exceptions when executing Okta Expression Language (OEL) queries.
3. **Case Sensitivity Mismatch:** Entra ID passed the authenticated UPN with camel-case syntax (`Andre.Iggy@...`), whereas strict string matching in Okta expected case alignment or explicit normalization.

### Technical Resolution Steps
1. **Defined Custom Attribute in Okta Profile Editor:**
   * Navigated to `Directory > Profile Editor > [Warriors Entra SSO Profile]`.
   * Added custom variable `upn` (Type: `string`, Display: `UPN`).
2. **Built Profile Mapping Rule:**
   * Mapped `appuser.upn` on the Entra IdP side to `user.login` on the Okta user side.
3. **Updated IdP Account Matching & Expression Language:**
   * Updated Account Matching to evaluate `idpuser.upn` against `Okta Username`.
   * Implemented Okta Expression Language string normalization:
     ```text
     String.toLowerCase(idpuser.upn)
     ```
4. **Enabled Account Linking Policy:**
   * Configured **Enable automatic linking** under IdP Account Matching settings with **Match restriction: None**.

---

## 🔬 Verification & Testing

* **System Log Audit:** Confirmed `user.authentication.auth_via_IDP` entries switched from `FAILURE` to `SUCCESS`.
* **IdP Routing Validation:** Verified unauthenticated requests to Okta seamlessly redirect to Entra ID and back without user intervention.
* **Downstream SSO Test:** Authenticated via Entra ID, landed on Okta Dashboard, clicked Salesforce tile, and verified instant SSO session initiation into Salesforce.

---

## 💼 Key Technical Skills Demonstrated
* **Identity Providers & Protocols:** SAML 2.0, SCIM, Microsoft Entra ID (Azure AD), Okta Universal Directory, Salesforce Identity.
* **Claim Engineering & Schema Customization:** Custom SAML attribute claims, XML namespace navigation, Okta Profile Editor schema expansion.
* **Okta Expression Language (OEL):** String transformation functions (`String.toLowerCase()`, `idpuser` evaluation).
* **Identity Troubleshooting:** Analysis of Okta System Logs, SAML response assertions, and account matching policies.
* **Role-Based Access Control (RBAC):** Group-based application provisioning and entitlement governance.
