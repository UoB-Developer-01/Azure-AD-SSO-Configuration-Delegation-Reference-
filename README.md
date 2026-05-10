# Azure-AD-SSO-Configuration-Delegation-Reference-

Yes — in Microsoft Entra ID (formerly Azure AD), you can delegate application management so that developers can create and manage **only the applications they own/create**, including SSO configuration for those applications. ([Microsoft Learn][1])

The typical model is:

* Allow selected developers to register applications
* They automatically become the **owner** of the app registration and enterprise application
* Owners can manage:

  * SSO configuration
  * User assignments
  * Provisioning
  * Certificates/secrets
  * Other owners
* But only for applications they own, not all tenant applications ([Microsoft Learn][1])

A common secure setup is:

| Role / Setting                                                            | Purpose                                       |
| ------------------------------------------------------------------------- | --------------------------------------------- |
| “Users can register applications” = No                                    | Prevent everyone from creating apps           |
| Assign selected users the Application Developer capability or custom role | Allow only approved developers to create apps |
| Use App Ownership                                                         | Scope management to owned apps only           |
| Avoid Application Administrator unless necessary                          | That role manages all apps tenant-wide        |

Key distinction:

* **Application Owner** → scoped to specific owned apps
* **Application Administrator / Cloud Application Administrator** → can manage all apps in the tenant ([Microsoft Learn][2])

For SSO specifically:

* App owners can configure SAML/OIDC SSO for their owned enterprise applications. ([Microsoft Learn][1])

There are a few important caveats:

1. App owners can potentially elevate privileges indirectly if the app itself has powerful permissions. Microsoft explicitly warns about this. ([Microsoft Learn][3])

2. Admin consent is tenant-wide, not owner-scoped.
   If an app receives admin consent for powerful APIs, the effect can extend beyond just that developer’s users unless assignment restrictions are enforced. ([Reddit][4])

3. Best practice is to:

   * Require at least 2 owners per app
   * Maintain governance/inventory
   * Review app permissions regularly
   * Restrict who can grant admin consent ([Microsoft Learn][1])

A very common enterprise pattern is:

```text
Developers
  ↓
Can create app registrations
  ↓
Automatically become app owners
  ↓
Can manage SSO and secrets ONLY for their apps
  ↓
Security/Identity team retains admin consent authority
```

Relevant Microsoft documentation:

* [Microsoft Entra application ownership overview](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/overview-assign-app-owners?utm_source=chatgpt.com)
* [Delegate app registration permissions in Entra ID](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/delegate-app-roles?utm_source=chatgpt.com)
* [Default user permissions in Entra ID](https://learn.microsoft.com/en-us/entra/fundamentals/users-default-permissions?utm_source=chatgpt.com)

[1]: https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/overview-assign-app-owners?utm_source=chatgpt.com "Overview of Enterprise Application Ownership - Microsoft Entra ID | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/delegate-app-roles?utm_source=chatgpt.com "Delegate application management administrator permissions - Microsoft Entra ID | Microsoft Learn"
[3]: https://learn.microsoft.com/bs-latn-ba/entra/identity/enterprise-apps/overview-assign-app-owners?utm_source=chatgpt.com "Overview of Enterprise Application Ownership - Microsoft Entra ID | Microsoft Learn"
[4]: https://www.reddit.com/r/entra/comments/1rjxsnq/entra_id_why_admin_consent_is_not_userscoped_and/?utm_source=chatgpt.com "Entra ID – Why Admin Consent Is Not User-Scoped (and how to restrict access properly)"

# Walkthrough

## Goal

Allow developers in Microsoft Entra ID to:

* Create their own applications
* Configure SSO for only those applications
* Manage secrets/certificates for their applications
* NOT manage applications owned by others
* Keep tenant-wide admin control centralized

---

# Recommended Secure Setup

## Architecture

```text id="entrasetup"
Global Admin / Identity Team
    │
    ├── Controls admin consent
    ├── Controls tenant-wide settings
    └── Reviews permissions

Developers
    │
    ├── Can create app registrations
    ├── Become owners of apps they create
    └── Can manage ONLY owned apps
```

---

# Step 1 — Disable General App Registration for Everyone

By default, many tenants allow all users to register apps.

You usually want to disable this first.

## Steps

1. Open:

[Microsoft Entra Admin Center](https://entra.microsoft.com?utm_source=chatgpt.com)

2. Go to:

```text
Identity
→ Users
→ User settings
```

3. Find:

```text
Users can register applications
```

4. Set to:

```text
No
```

5. Save

---

# Step 2 — Create a Developers Group

## Steps

1. Go to:

```text
Identity
→ Groups
→ New group
```

2. Create:

| Setting    | Value          |
| ---------- | -------------- |
| Group Type | Security       |
| Name       | App Developers |
| Membership | Assigned       |

3. Add your approved developers

---

# Step 3 — Allow ONLY That Group to Create Applications

There are two good approaches.

---

# Option A (Recommended) — Use Custom Role

This is the cleanest enterprise approach.

## Create Custom Role

Go to:

```text
Identity
→ Roles & admins
→ New custom role
```

Create role:

| Field       | Value                                     |
| ----------- | ----------------------------------------- |
| Name        | App Registration Creator                  |
| Description | Can create/manage owned applications only |

---

## Add Permissions

Add these permissions:

```text
microsoft.directory/applications/create
microsoft.directory/applications/standard/read
microsoft.directory/applications/owners/update
```

Optionally:

```text
microsoft.directory/servicePrincipals/create
```

Save role.

---

## Assign Role to Developers Group

1. Open the role
2. Click:

```text
Assignments
→ Add assignments
```

3. Assign to:

```text
App Developers
```

---

# Option B — Use Built-In Role

Simpler but broader.

Assign developers to:

* Application Developer

This allows:

* App registrations
* Ownership of created apps

But still does NOT give full tenant-wide app administration.

---

# Step 4 — Verify Ownership Behavior

When a developer creates:

* App Registration
* Enterprise Application (Service Principal)

They automatically become owner.

To verify:

1. Go to:

```text
Identity
→ Applications
→ App registrations
```

2. Open the app

3. Go to:

```text
Owners
```

You should see the creator listed.

---

# Step 5 — Allow SSO Management

App owners can manage SSO on owned applications.

## They Can Configure

* SAML
* OpenID Connect
* OAuth2
* Certificates
* Redirect URIs
* Secrets
* Claims mapping
* User assignments

Path:

```text
Enterprise Applications
→ [Their App]
→ Single sign-on
```

No extra permissions are usually required if they are owners.

---

# Step 6 — Restrict Admin Consent

This is VERY important.

Without this:
developers may request powerful APIs.

---

## Configure Admin Consent Workflow

Go to:

```text
Identity
→ Applications
→ Enterprise applications
→ Consent and permissions
→ User consent settings
```

Recommended:

| Setting                | Value        |
| ---------------------- | ------------ |
| User consent           | Do not allow |
| Admin consent workflow | Enabled      |

---

## Why

This allows:

* Developers to build apps
* But security/admin team approves dangerous permissions

Examples:

* Graph API Mail.Read
* Directory.ReadWrite.All
* User.Read.All

---

# Step 7 — Restrict Visibility (Optional but Recommended)

You can limit who sees applications.

Go to:

```text
Enterprise Applications
→ User settings
```

Configure:

| Setting                          | Recommended |
| -------------------------------- | ----------- |
| Users can consent to apps        | No          |
| Users can see only assigned apps | Yes         |

---

# Step 8 — Require Assignment for Enterprise Apps

For sensitive applications:

1. Open Enterprise App
2. Go to:

```text
Properties
```

3. Set:

```text
Assignment required = Yes
```

This prevents broad unintended access.

---

# Step 9 — Add Governance

Recommended controls:

| Control           | Recommendation   |
| ----------------- | ---------------- |
| Minimum owners    | 2 owners per app |
| Secret expiration | ≤ 12 months      |
| App reviews       | Quarterly        |
| Unused apps       | Disable/remove   |
| Naming standard   | Required         |

Example naming:

```text
UOB-HR-Portal-DEV
UOB-Admissions-API-PROD
```

---

# Step 10 — Monitor Activity

Use:

```text
Identity
→ Monitoring & health
→ Audit logs
```

Track:

* App creation
* Consent grants
* Secret creation
* SSO changes

---

# What Developers Will Be Able to Do

| Action                          | Allowed |
| ------------------------------- | ------- |
| Create apps                     | Yes     |
| Configure SSO                   | Yes     |
| Create secrets                  | Yes     |
| Add redirect URIs               | Yes     |
| Manage owned apps               | Yes     |
| Manage other developers' apps   | No      |
| Grant tenant-wide admin consent | No      |
| Become Global Admin             | No      |

---

# What NOT to Assign

Avoid giving developers:

* Global Administrator
* Cloud Application Administrator
* Application Administrator

These roles can manage ALL tenant apps.

---

# Microsoft Documentation

* [Entra app ownership documentation](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/overview-assign-app-owners?utm_source=chatgpt.com)
* [Delegate app registration permissions](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/delegate-app-roles?utm_source=chatgpt.com)
* [Admin consent configuration](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-admin-consent-workflow?utm_source=chatgpt.com)
* [Default user permissions in Entra ID](https://learn.microsoft.com/en-us/entra/fundamentals/users-default-permissions?utm_source=chatgpt.com)


