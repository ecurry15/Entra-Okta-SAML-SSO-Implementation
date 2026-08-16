# Federated SSO Implementation: Entra > Okta > Salesforce

<img width="1536" height="1024" alt="Federation Diagram" src="https://github.com/user-attachments/assets/f57a20fb-dc44-4b8b-9881-48f63dda9de1" />


## 📌 Summary
* This project is a continuation of my previous project https://github.com/ecurry15/Enterprise-Identity-Access-Management-Lab-Microsoft-Entra. In which I set up an enterprise-level Entra IAM environment.

* I built off that environment by importing the users from Entra into Okta. I then designed and deployed a multi-platform identity federation architecture. I configured **Microsoft Entra ID** as the Primary Identity Provider (IdP), **Okta** as the Central Hub/Downstream IdP, and **Salesforce** as a downstream SaaS application.

* The end-to-end flow enables Single Sign-On (SSO): users authenticate via Entra ID, are federated to Okta via SAML, and are signed in to Salesforce through Okta-managed SSO and account provisioning.

---

## 🏗️ Authentication Flow

### Authentication Flow:
1. The user accesses the **Okta End-User Portal**
2. Okta redirects the browser to **Microsoft Entra ID** for authentication.
3. The user authenticates against Entra ID (enforcing primary credentials and Conditional Access policies).
4. Entra ID issues a signed SAML 2.0 assertion containing user claims.
5. Okta validates the SAML response, matches the user account, and establishes an Okta session.
6. The user is signed into the Okta End-User Dashboard, which displays assigned applications such as Salesforce.
7. The user selects the Salesforce application in the Okta dashboard.
8. Okta generates a SAML 2.0 authentication assertion for Salesforce containing the required user attributes/claims.
9. Salesforce validates the SAML assertion, verifies the user identity and required claims, and the user is signed into Salesforce.


---

## 🛠️ Implementation Phasing

### Phase 1: Directory Setup & User/Group Import
* Provisioned an enterprise **Okta Tenant**.
* Imported identity user profiles from **Microsoft Entra ID** into Okta (Only 9 were imported due to license restrictions).
* Created **Okta User Groups** to match the Entra ID security groups.

<img width="1777" height="855" alt="okta entra users" src="https://github.com/user-attachments/assets/fc60b23e-79d5-40a4-8d51-37395e8c2566" />
<img width="807" height="645" alt="okta groups" src="https://github.com/user-attachments/assets/b6a2c37a-d619-45c5-aefd-d27f49ecb193" />




### Phase 2: Inbound SAML Federation (Entra ID ➔ Okta)
* Created a SAML2 Identity provider **Warriors Entra SSO** in Okta. 
* Created an Enterprise Application in Entra ID **Warriors Okta SSO** for Okta Inbound Federation.
* Assigned the IT-Users group to **Warriors Okta SSO** app
* Set up Single Sign-On with SAML 2.0 in Entra using the Okta Entity ID and ACS URL.
* Set up SAML Protocol settings in Okta with the Entra identifier, login URL, and certificate.
* Matched **IdP username** to **idpuser.upn**, which takes the Entra username entered **upn** and looks it up in the Okta directory.

<img width="938" height="873" alt="warriors SSO Okta app" src="https://github.com/user-attachments/assets/f3421a1f-604d-425f-89ac-a1276668bfe6" />
<img width="1772" height="810" alt="Entra Okta SAML" src="https://github.com/user-attachments/assets/b7ac5590-8ef3-4b82-9807-b0e704956516" />
<img width="936" height="582" alt="IT group added to app" src="https://github.com/user-attachments/assets/3f4d93a3-a28d-4748-b290-981a66f9baae" />
<img width="1772" height="810" alt="Entra Okta SAML" src="https://github.com/user-attachments/assets/8bc9781a-7e12-4395-ad34-4abcef865037" />
<img width="934" height="799" alt="SSO routing rule" src="https://github.com/user-attachments/assets/ef7a4ba2-74a8-4182-99e4-4cbe0fbf880f" />

### Phase 3: Outbound SAML SSO & Provisioning (Okta ➔ Salesforce)
* Created a Salesforce account.
* Provisioned a user **Andre Iggy* which is apart of the **IT group**
* Integrated the **Salesforce Integration App** inside Okta.
* Configured SAML 2.0 SSO settings in Salesforce (Issuer, Identity Provider Certificate, SAML Identity Type).
* Enabled domain management in Salesforce and set Okta as the primary authentication service.
* Assigned the Application to the **IT group**

<img width="987" height="725" alt="salesforce integration" src="https://github.com/user-attachments/assets/451ccb0d-f2ad-48c5-aaab-4c8c017c44c5" />
<img width="837" height="871" alt="SF SAML" src="https://github.com/user-attachments/assets/f40bf318-4b97-46f6-a650-c88263cc91f4" />
<img width="979" height="855" alt="SF assigned to IT group" src="https://github.com/user-attachments/assets/34b847f3-80bc-4697-ab2a-4d522794181e" />

---

## 🚨 Deep Dive: Troubleshooting Inbound Federation & SAML Claims

During Phase 2, inbound federation failed with the following error in Okta:
`FAILURE: User login to Okta via IdP - User Not Found / User Creation Disabled`

```
System Log Entry:
Target: Andre.Iggy@campbell400.onmicrosoft.com
DetailEntry: unknown
Reason: Invalid username transform / Account match failure
```

<img width="916" height="558" alt="failed sso" src="https://github.com/user-attachments/assets/74645fb9-038c-46ee-bceb-f0012eb1a051" />


### Root Cause Analysis
1. **Schema Mismatch:** While using the **Name Id** claim, Entra was sending the **User Principal Name** wrapped inside an XML string to Okta. Okta was then unable to match the XML to a user in its directory.
2. **Account Link Policy:** Enable automatic linking was disabled.

### Resolution Steps
1. **Defined Custom Attribute in Okta Profile Editor:**
   * Navigated to `Directory > Profile Editor > [Warriors Entra SSO Profile]`.
   * Added custom variable `upn` (Type: `string`, Display: `UPN`).
2. **Built Profile Mapping Rule:**
   * Mapped `appuser.upn` on the Entra IdP side to `user.login` on the Okta user side.
3. **Updated IdP Account Matching & Expression Language:**
   * Updated Account Matching to evaluate `idpuser.upn` against `Okta Username`.
4. **Enabled Account Linking Policy:**
   * Configured **Enable automatic linking** under IdP Account Matching settings.

<img width="960" height="285" alt="mapping" src="https://github.com/user-attachments/assets/88079188-bce8-431c-8c9f-e6be5890419a" />
<img width="952" height="435" alt="Entra UPN claim" src="https://github.com/user-attachments/assets/ee329f0a-aa3d-4d22-a23e-817b942d416b" />

---

## 🔬 Verification & Testing
* **IT User Andre Iggy completes the Authentication flow**
* **System Log Audit:** Confirmed `user.authentication.auth_via_IDP` entries switched from `FAILURE` to `SUCCESS`.

<img width="997" height="820" alt="Entra SSO available" src="https://github.com/user-attachments/assets/84922dc5-cbe4-45b7-8a2f-3a7d5446af0e" />
<img width="867" height="718" alt="Andre SSO sign in" src="https://github.com/user-attachments/assets/1de752f3-9ec4-486e-9ec1-5406689511e9" />
<img width="912" height="502" alt="successful entra log in log" src="https://github.com/user-attachments/assets/618fceb5-150c-4ce9-9555-aeb352cd088f" />
<img width="921" height="743" alt="Andre iggy logs" src="https://github.com/user-attachments/assets/fa7e92b5-050a-40b9-8acb-8f50f66b94fa" />

---

## 💼 Key Technical Skills Demonstrated
* **Identity Providers & Protocols:** SAML 2.0, SCIM, Microsoft Entra ID (Azure AD), Okta, Salesforce.
* **Claim Engineering:** Custom SAML attribute claims, Okta Profile Editor schema expansion.
* **Identity Troubleshooting:** Analysis of Okta System Logs, SAML response assertions, and account matching policies.
* **Role-Based Access Control (RBAC):** Group-based application provisioning and entitlement governance.
