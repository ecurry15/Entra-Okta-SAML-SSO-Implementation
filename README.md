# Federated SSO Implementation: Entra ID > Okta > Salesforce

<p align="center">
  <img src="https://github.com/user-attachments/assets/f57a20fb-dc44-4b8b-9881-48f63dda9de1" alt="Federation Diagram" width="900"/>
</p>

## 📌 Summary

This project is a continuation of my previous [Enterprise Identity & Access Management Lab](https://github.com/ecurry15/Enterprise-Identity-Access-Management-Lab-Microsoft-Entra), where I built an enterprise-level Microsoft Entra IAM environment.

I built on that environment by importing users from Entra ID into Okta and then designing and deploying a multi-platform identity federation architecture. **Microsoft Entra ID** serves as the Primary Identity Provider (IdP), **Okta** serves as the Central Hub/Downstream IdP, and **Salesforce** serves as the downstream SaaS application.

The end-to-end flow enables Single Sign-On (SSO):

**Entra ID → Okta → Salesforce**

Users authenticate through Entra ID, are federated to Okta using SAML, and then access Salesforce through Okta-managed SSO and account provisioning.

---

## 🏗️ Authentication Flow

**Entra ID → Okta → Salesforce**

### 1. User Authentication: Entra ID → Okta

1. The user accesses the **Okta End-User Portal**.
2. Okta identifies the user as belonging to the **Entra ID federation** and redirects the browser to Microsoft Entra ID.
3. The user authenticates against **Microsoft Entra ID**, where primary credentials and Conditional Access policies are enforced.
4. Entra ID generates a signed **SAML 2.0 assertion** containing the user's identity and configured claims.
5. Okta receives and validates the SAML response, then uses the configured account-matching rules to identify the corresponding Okta user.
6. Okta establishes a session and signs the user into the **Okta End-User Dashboard**.

### 2. Application SSO: Okta → Salesforce

7. The user selects **Salesforce** from the Okta End-User Dashboard.
8. Okta generates a signed **SAML 2.0 assertion** containing the required user attributes and claims.
9. Salesforce validates the SAML assertion and verifies the user's identity.
10. Salesforce establishes the user session and signs the user into the application.

### Federation Architecture

| Component | Role |
|---|---|
| **Microsoft Entra ID** | Primary Identity Provider (IdP) |
| **Okta** | Central Identity Hub / Downstream IdP |
| **Salesforce** | Downstream SaaS Application / Service Provider (SP) |
| **SAML 2.0** | Federation protocol used between Entra ID, Okta, and Salesforce |

---

## 🛠️ Implementation Phases

### Phase 1: Directory Setup & User/Group Import

- Provisioned an enterprise **Okta Tenant**.
- Imported identity user profiles from **Microsoft Entra ID** into Okta. Only 9 are active due to license restrictions.
- Created **Okta User Groups** to match the Entra ID security groups.

<p align="center">
  <img src="https://github.com/user-attachments/assets/fc60b23e-79d5-40a4-8d51-37395e8c2566" alt="Okta Entra Users" width="850"/>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/b6a2c37a-d619-45c5-aefd-d27f49ecb193" alt="Okta Groups" width="650"/>
</p>

---

### Phase 2: Inbound SAML Federation (Entra ID ➔ Okta)

- Created a SAML 2.0 Identity Provider named **Warriors Entra SSO** in Okta.
- Created an Enterprise Application in Entra ID named **Warriors Okta SSO** for Okta inbound federation.
- Assigned the **IT-Users** group to the **Warriors Okta SSO** application.
- Configured Single Sign-On with SAML 2.0 in Entra ID using the Okta Entity ID and ACS URL.
- Configured the SAML protocol settings in Okta using the Entra identifier, login URL, and certificate.
- Matched **IdP username** to `idpuser.upn`, which takes the Entra username entered as the `upn` and uses it to look up the corresponding user in the Okta directory.

<p align="center">
  <img src="https://github.com/user-attachments/assets/f3421a1f-604d-425f-89ac-a1276668bfe6" alt="Warriors SSO Okta App" width="750"/>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/b7ac5590-8ef3-4b82-9807-b0e704956516" alt="Entra Okta SAML Configuration" width="900"/>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/3f4d93a3-a28d-4748-b290-981a66f9baae" alt="IT Group Added to Application" width="750"/>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/ef7a4ba2-74a8-4182-99e4-4cbe0fbf880f" alt="SSO Routing Rule" width="750"/>
</p>

---

### Phase 3: Outbound SAML SSO & Provisioning (Okta ➔ Salesforce)

- Created a Salesforce account.
- Provisioned a user, **Andre Iggy**, who is part of the **IT group**.
- Integrated the **Salesforce Integration App** inside Okta.
- Configured SAML 2.0 SSO settings in Salesforce, including the Issuer, Identity Provider Certificate, and SAML Identity Type.
- Enabled domain management in Salesforce and set Okta as the primary authentication service.
- Assigned the Salesforce application to the **IT group**.

<p align="center">
  <img src="https://github.com/user-attachments/assets/451ccb0d-f2ad-48c5-aaab-4c8c017c44c5" alt="Salesforce Integration" width="800"/>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/f40bf318-4b97-46f6-a650-c88263cc91f4" alt="Salesforce SAML Configuration" width="750"/>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/34b847f3-80bc-4697-ab2a-4d522794181e" alt="Salesforce Assigned to IT Group" width="800"/>
</p>

---

## 🚨 Deep Dive: Troubleshooting Inbound Federation & SAML Claims

During Phase 2, inbound federation initially failed with the following error in Okta:

`FAILURE: User login to Okta via IdP - User Not Found / User Creation Disabled`

```text
System Log Entry:
Target: Andre.Iggy@campbell400.onmicrosoft.com
DetailEntry: unknown
Reason: Invalid username transform / Account match failure
```
<p align="center"> <img src="https://github.com/user-attachments/assets/74645fb9-038c-46ee-bceb-f0012eb1a051" alt="Failed SSO" width="800"/> </p>

## Root Cause Analysis

1. **Schema Mismatch:** While using the **Name ID** claim, Entra ID was sending the User Principal Name wrapped inside an XML string to Okta. Okta was unable to match the XML value to a user in its directory.
2. **Account Link Policy:** Automatic account linking was disabled.

### Resolution Steps

#### 1. Defined a Custom Attribute in Okta Profile Editor

- Navigated to `Directory > Profile Editor > [Warriors Entra SSO Profile]`.
- Added the custom variable `upn` with the type `string` and display name `UPN`.

#### 2. Built a Profile Mapping Rule

- Mapped `appuser.upn` on the Entra IdP side to `user.login` on the Okta user side.

#### 3. Updated IdP Account Matching & Expression Language

- Updated Account Matching to evaluate `idpuser.upn` against the **Okta Username**.

#### 4. Enabled Account Linking Policy

- Set **Enable automatic linking** under the IdP Account Matching settings.

<p align="center">
  <img src="https://github.com/user-attachments/assets/88079188-bce8-431c-8c9f-e6be5890419a" alt="Okta Profile Mapping" width="800"/>
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/ee329f0a-aa3d-4d22-a23e-817b942d416b" alt="Entra UPN Claim" width="800"/>
</p>

---

## 🔬 Verification & Testing

- **IT User Andre Iggy completed the authentication flow successfully.**
- **System Log Audit:** Confirmed that `user.authentication.auth_via_IDP` entries changed from `FAILURE` to `SUCCESS`.

<p align="center">
  <img src="https://github.com/user-attachments/assets/84922dc5-cbe4-45b7-8a2f-3a7d5446af0e" alt="Entra SSO Available" width="800"/>
</p>

<p align="center">
  <img width="800" alt="Andre SSO sign in" src="https://github.com/user-attachments/assets/61fb2cfe-2647-4ea3-a4b4-fb67d7a12da4" />
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/618fceb5-150c-4ce9-9555-aeb352cd088f" alt="Successful Entra Login Log" width="800"/>
</p>

<p align="center">
  <img width="800" alt="Andre Successful login" src="https://github.com/user-attachments/assets/3c43bb3d-c0f2-425d-aa79-613e77b054af" />
</p>

---

## 💼 Key Technical Skills Demonstrated

- **Identity Providers & Protocols:** SAML 2.0, SCIM, Microsoft Entra ID (Azure AD), Okta, Salesforce.
- **Claim Engineering:** Custom SAML attribute claims and Okta Profile Editor schema expansion.
- **Identity Troubleshooting:** Analysis of Okta System Logs, SAML response assertions, and account matching policies.
- **Role-Based Access Control (RBAC):** Group-based application provisioning and entitlement governance.
