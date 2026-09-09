# Federated Identity Credentials in Microsoft Entra ID

**Anchor example: Microsoft Fabric workspace identity**

What a federated identity credential (FIC) is, where it lives, who creates it, and how Fabric's workspace identity uses one internally.

---

## Table of Contents

- [1. The Problem FICs Solve](#1-the-problem-fics-solve)
- [2. What a Federated Identity Credential Is](#2-what-a-federated-identity-credential-is)
- [3. The Token Exchange](#3-the-token-exchange)
- [4. Where a FIC Lives](#4-where-a-fic-lives)
- [5. Terminology](#5-terminology)
- [6. Anchor Example: Fabric Workspace Identity](#6-anchor-example-fabric-workspace-identity)
- [7. The Fabric ↔ Entra Relationship](#7-the-fabric--entra-relationship)
- [8. Common Misconceptions](#8-common-misconceptions)
- [9. Security Considerations](#9-security-considerations)
- [10. Documented vs. Inferred](#10-documented-vs-inferred)
- [11. References](#11-references)

---

## 1. The Problem FICs Solve

A non-human workload needs to authenticate to Entra ID. Traditionally the app registration holds a client secret or certificate, and the workload holds a copy. That model breaks in predictable ways:

| Problem | Consequence |
| --- | --- |
| Secrets expire | Pipelines break when nobody remembers who owns the app |
| Secrets are copyable | Anyone with read access to the credential store holds the identity |
| Secrets are portable | The credential works from any IP, indefinitely |
| Rotation is toil | Every rotation is a change window and a chance to break production |

Federated identity credentials remove the credential entirely. Nothing is stored, so nothing can leak, expire, or need rotating.

---

## 2. What a Federated Identity Credential Is

A FIC is a **standing rule** on an Entra app registration or user-assigned managed identity:

> Any OIDC token signed by issuer `X`, carrying subject `Y`, for audience `Z`, may be exchanged for an access token for this identity.

All three credential types occupy the same collection on the application object, under **Certificates & secrets**:

| Credential type | What it proves | Stored on the workload? |
| --- | --- | --- |
| Client secret | Something you **have** | Yes |
| Certificate | Something you **have** | Yes |
| Federated identity credential | Something you **qualify for** | No |

A secret is proof by possession. A FIC is proof by matching a rule.

**On the name.** *Federated* describes the mechanism: an identity vouched for by one trust domain, accepted in another via token exchange. Same sense as federated user authentication with AD FS, different plumbing. *Credential* is literal — it sits alongside secrets and certificates and serves the identical purpose. The name says nothing about *who* the issuer is.

---

## 3. The Token Exchange

```
Workload starts                    no credential stored anywhere
        │
        ▼
Its own IdP signs a JWT            short-lived OIDC token
        │                          presented as client_assertion
        ▼
Entra validates                    fetches issuer JWKS, verifies signature,
        │                          matches issuer + subject + audience
        ▼
Entra issues access token          short-lived, scoped to the SP
        │
        ▼
Target resource                    ADLS, Key Vault, etc.
```

Mechanically this is the OAuth 2.0 client credentials flow with `client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer`. The difference from certificate auth is that the assertion is signed by the **external** issuer, not by a key the workload holds.

**Anatomy of a FIC:**

| Field | Purpose | Example (GitHub Actions) |
| --- | --- | --- |
| `issuer` | Whose signature Entra accepts | `https://token.actions.githubusercontent.com` |
| `subject` | Which specific workload context | `repo:my-org/infra:environment:production` |
| `audience` | Who the token is intended for | `api://AzureADTokenExchange` |
| `name` | Label only, no security meaning | `gh-prod-deploy` |

Constraints: maximum **20 FICs** per app or UAMI; `issuer` + `subject` must be unique on the app; **wildcards are unsupported**; a subject typo saves successfully and then fails the exchange **without a useful error**.

---

## 4. Where a FIC Lives

**Always on the identity being assumed. Never on the workload doing the assuming.**

| Scenario | FIC lives on | Workload holds |
| --- | --- | --- |
| GitHub Actions → Azure | The Entra app registration | Nothing persistent |
| AKS pod → Azure | The user-assigned managed identity | A projected service account token |
| Fabric workspace identity | The Entra app registration | Nothing persistent |

This is why a workspace identity's FIC is visible in the **Entra app registration blade** and nowhere in Fabric's workspace settings. Fabric created it; Entra stores and enforces it. Creation and residence are different things.

Example: Workspace admins in Microsoft Fabric who want their Workspaces to interact with Azure resources in Entra such as Storage accounts needs an identity. 
In this scenario, Workspace admins can create a workspace identity tied to the Workspace so that it can authentiate to Entra ID and recieve an access token.
<img width="1296" height="693" alt="image" src="https://github.com/user-attachments/assets/de50d03d-0c47-4aa7-b4eb-22a2894e2440" />


---

## 5. Terminology

| Term | What it is | Licensing |
| --- | --- | --- |
| Workload identity | The object: service principal or managed identity | Core Entra |
| Workload identity federation | The FIC-based mechanism described here | Core Entra, free |
| Microsoft Entra Workload ID | SKU: CA for workload identities, ID Protection for SPs, SP access reviews | Premium add-on |
| Entra Workload ID for AKS | AKS feature using federation for pod-level tokens | Core |

Rows 2 and 3 get conflated constantly. **Federation does not require the premium SKU.**

---

## 6. Anchor Example: Fabric Workspace Identity

### What Fabric provisions

Creating a workspace identity in **Workspace settings → Workspace identity** causes Fabric to create a **service principal**, an accompanying **app registration**, and the credential on it.

Microsoft's documentation describes this only as "Fabric automatically manages the credentials associated with workspace identities" and does not specify the mechanism. In practice a **FIC is present on the app registration**, observable in the Entra portal.

Other documented behaviour: the identity gets **no workspace role by default**; trusted workspace access to firewall-enabled ADLS Gen2 requires an **F SKU**; renaming the workspace renames the identity but leaves the Entra objects unchanged; deleting the workspace deletes the identity, and it **cannot be restored**.

### Who creates the FIC

**Fabric does**, acting through a Microsoft first-party service principal named **Fabric Identity Management**, which the documentation names as the *configuration owner* of the application in Enterprise Applications.

Example:

<img width="1279" height="588" alt="image" src="https://github.com/user-attachments/assets/a4b12a5d-2cc4-45f1-b4d9-fd754a2d5b07" />


**Entra ID does not create it.** Entra never authors credentials on its own initiative. Every FIC exists because something called Microsoft Graph. For a CI/CD pipeline that caller is you; for a workspace identity it is Fabric Identity Management.

### Who the "external" IdP is

Microsoft itself. Entra's federation machinery is issuer-agnostic — it validates a signed OIDC token against an issuer/subject/audience triple and does not care whether the signer is GitHub, an EKS cluster, or a Microsoft first-party service.

**"Federated" does not imply "third-party."** It implies token exchange across a trust boundary, and Microsoft draws internal trust boundaries too.

### Division of responsibility

| Party | Role |
| --- | --- |
| Fabric Identity Management | Creates the app registration, SP, and FIC; owns their lifecycle |
| Microsoft internal token issuer | Signs the OIDC assertion at runtime |
| Entra ID | Stores the FIC, validates it, issues the access token |
| You | Grant the identity permissions on target resources. Nothing else |

---

## 7. The Fabric ↔ Entra Relationship

### Two distinct trusts

Collapsing these is the most common source of confusion.

| Layer | Does Entra trust Fabric? | Why |
| --- | --- | --- |
| **Control plane** | Yes, automatically | Fabric Identity Management is a first-party app with directory permissions to create applications and write credentials. This is what lets it provision without asking you. Nothing to do with federation |
| **Runtime** | No | Entra trusts one issuer/subject/audience triple on one app object and re-validates the signature on every exchange. Fabric gets no standing beyond what the FIC states |

The automatic trust is in Fabric's ability to **create** the credential, not in Entra's willingness to **honor** it.

There is **no tenant-wide trusted-IdP list** for workload federation. That is a real architectural difference from federated *user* authentication, where a WS-Fed or SAML trust applies to an entire verified domain.

### Workspace identity is not a managed identity

Microsoft's documentation is explicit that lifecycle, administration, and governance differ.

| | Managed identity | Fabric workspace identity |
| --- | --- | --- |
| Underlying object | MSI resource | App registration + SP |
| Credential mechanism | Platform-held cert via IMDS | FIC on the app object |
| Lifecycle owner | Azure resource provider | Fabric |
| Visible in App registrations | No | Yes |

---

## 8. Common Misconceptions

| Misconception | Reality |
| --- | --- |
| "The FIC lives in Fabric" | It lives on the Entra application object. Fabric only created it |
| "Entra provides the federated credential" | Entra stores and validates. Something must call Graph to create one |
| "Federated means a third-party IdP" | It means token exchange across a trust boundary. The issuer can be Microsoft |
| "Entra trusts GitHub / Fabric as an IdP" | Entra trusts one issuer + subject pair on one app. Nothing broader |
| "Workspace identity is a managed identity" | Similar behaviour, different object model and governance |
| "Federation needs Workload ID Premium" | Federation is free and part of core Entra |
| "A wrong subject throws an error" | It saves successfully and fails silently at exchange time |

---

## 9. Security Considerations

### Subject scoping is the security boundary

For FICs you configure yourself, the `subject` claim is the entire access control decision. Over-broad subjects are the dominant misconfiguration:

- A branch wildcard or `pull_request` context means **anyone who can open a PR** obtains a token with whatever roles the app holds.
- Because wildcards are unsupported, scaling means **one identity per environment**, not one per branch.
- Repo rename or transfer can leave a FIC pointing at a name someone else can claim.

### Rogue FIC as a privilege escalation path

A workspace identity's app registration is an ordinary app registration. Microsoft's documentation confirms **Application Administrators or higher can view, modify, and delete** it.

A holder of that role can **add a second FIC** pointing at an issuer and subject they control, producing an externally assumable identity that may hold trusted workspace access to a firewall-enabled storage account. Fabric's UI surfaces none of this.

Microsoft warns that unauthorized modifications "may be reverted." **Treat that as a reconciliation loop, not a security control.** There may be a usable window before reversion, and silent reversion can destroy the evidence.

### Detections worth building

| Detection | Signal |
| --- | --- |
| FIC added to a workspace-identity app | Entra audit: `Update application – Certificates and secrets management`. Fire on the **add event**, not on persisted state |
| FIC count drift | Baseline expected count per workspace identity; investigate anything above it |
| Subject wildcards / PR contexts | Periodic Graph enumeration of all FICs; flag branch-level and `pull_request` subjects |
| Unexpected Application Administrator grants | Standing role review — this role is the escalation prerequisite |
| Workspace identity lifecycle | Purview audit: `Created` / `Deleted Fabric Identity for Workspace` |
| Token issuance baseline | Purview audit: `Retrieved Fabric Identity Token for Workspace` |

---

## 10. Documented vs. Inferred

Microsoft does not document the internal credential mechanism, so epistemic status matters here.

| Claim | Status |
| --- | --- |
| Fabric creates an SP and app registration | Documented |
| Fabric automatically manages the credentials | Documented (mechanism unspecified) |
| Fabric Identity Management is configuration owner | Documented |
| App Admins can modify the app; changes may be reverted | Documented |
| A FIC is present on the app registration | **Observed in tenant**, not documented |
| The specific issuer and subject values used | **Inferred** |

Microsoft's silence on the mechanism is expected for a first-party internal. It is not evidence against the FIC being there.

---

## 11. References

- [Workspace identity — Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/security/workspace-identity)
- [Authenticate with workspace identity](https://learn.microsoft.com/en-us/fabric/security/workspace-identity-authenticate)
- [Trusted workspace access](https://learn.microsoft.com/en-us/fabric/security/security-trusted-workspace-access)
- [Manage Fabric identities](https://learn.microsoft.com/en-us/fabric/admin/fabric-identities-manage)
- [Workload identity federation considerations](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-considerations)
- [Add a credential to your application](https://learn.microsoft.com/en-us/entra/identity-platform/how-to-add-credentials)
- [Application Administrator role reference](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#application-administrator)
