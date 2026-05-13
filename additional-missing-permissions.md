```bash
# Prompt

are the current permissions sufficient? what does each do and might be missing?

microsoft.directory/applications/standard/read Read standard properties of applications microsoft.directory/servicePrincipals/create Create service principals microsoft.directory/servicePrincipals/standard/read Read basic properties of service principals
```

---

No — those permissions alone are probably **not sufficient** for the full workflow you described earlier (creating app registrations, configuring SSO, managing secrets, redirect URIs, etc.).

Your current role is actually very minimal.

# Your Current Permissions

```text id="3l8psq"
microsoft.directory/applications/standard/read
microsoft.directory/servicePrincipals/create
microsoft.directory/servicePrincipals/standard/read
```

Here is what each one really does.

| Permission                                            | What It Allows                                    | What It Does NOT Allow                    |
| ----------------------------------------------------- | ------------------------------------------------- | ----------------------------------------- |
| `microsoft.directory/applications/standard/read`      | Read basic app registration properties            | Cannot create or modify applications      |
| `microsoft.directory/servicePrincipals/create`        | Create enterprise applications/service principals | Does not automatically make creator owner |
| `microsoft.directory/servicePrincipals/standard/read` | Read basic enterprise app properties              | Cannot modify enterprise apps             |

Microsoft documents these permission definitions in the Entra RBAC permission reference. ([Microsoft Learn][1])

---

# Major Missing Permission

You are missing the most important permission:

```text id="79a8dx"
microsoft.directory/applications/createAsOwner
```

OR alternatively:

```text id="r0zj9z"
microsoft.directory/applications/create
```

Without one of those:

* users cannot properly create app registrations
* they may only be creating service principals
* ownership behavior may not work correctly

Microsoft explicitly documents the difference:

| Permission                   | Behavior                            |
| ---------------------------- | ----------------------------------- |
| `applications/createAsOwner` | Creator automatically becomes owner |
| `applications/create`        | Creator is NOT automatically owner  |

([Microsoft Learn][2])

For your use case, `createAsOwner` is usually the correct choice.

---

# Why `createAsOwner` Is Better

Microsoft states:

> “Assigning this permission results in the creator being added as the first owner.” ([Microsoft Learn][2])

That is exactly the secure delegation model you want:

```text id="v0y7uv"
Developer
  ↓
Creates app
  ↓
Automatically becomes owner
  ↓
Can manage ONLY owned app
```

---

# Another Important Missing Permission

You are also likely missing:

```text id="kgc4j9"
microsoft.directory/applications/owners/update
```

Without it:

* ownership management may be limited
* adding secondary owners may fail
* some ownership operations in portal/API may not work

Microsoft documents this permission as:

> “Update owners of applications.” ([Microsoft Learn][1])

---

# What You Probably Need for Full SSO Management

If your goal is:

* create app registrations
* configure SAML/OIDC
* manage secrets/certificates
* configure redirect URIs
* manage enterprise app settings
* assign users/groups
* manage owned apps only

then your role likely needs something closer to:

```text id="x4n7ph"
microsoft.directory/applications/createAsOwner
microsoft.directory/applications/standard/read
microsoft.directory/applications/basic/update
microsoft.directory/applications/authentication/update
microsoft.directory/applications/credentials/update
microsoft.directory/applications/owners/update

microsoft.directory/servicePrincipals/create
microsoft.directory/servicePrincipals/standard/read
microsoft.directory/servicePrincipals/basic/update
microsoft.directory/servicePrincipals/credentials/update
microsoft.directory/servicePrincipals/owners/update
microsoft.directory/servicePrincipals/appRoleAssignedTo/update
```

---

# What These Additional Permissions Do

| Permission                                   | Purpose                            |
| -------------------------------------------- | ---------------------------------- |
| `applications/basic/update`                  | Rename/update app properties       |
| `applications/authentication/update`         | Configure redirect URIs, SAML/OIDC |
| `applications/credentials/update`            | Create secrets/certificates        |
| `servicePrincipals/basic/update`             | Update enterprise app settings     |
| `servicePrincipals/credentials/update`       | Manage enterprise app credentials  |
| `servicePrincipals/appRoleAssignedTo/update` | Assign users/groups to app         |

All of these are documented in Microsoft’s permission reference. ([Microsoft Learn][1])

---

# Important Security Warning

Be careful with this permission:

```text id="9ny3qh"
microsoft.directory/applications/permissions/update
```

This allows modifying API permissions requested by applications. ([Microsoft Learn][2])

That means developers could:

* request Graph permissions
* request sensitive delegated scopes
* request application permissions

Even though admin consent may still block activation.

For many enterprises:

```text id="s2y2n5"
applications/permissions/update
```

is considered higher risk and may require governance review.

---

# Recommended Minimal Secure Role

For your scenario, a good balance is probably:

```text id="b0vfd8"
microsoft.directory/applications/createAsOwner
microsoft.directory/applications/standard/read
microsoft.directory/applications/basic/update
microsoft.directory/applications/authentication/update
microsoft.directory/applications/credentials/update
microsoft.directory/applications/owners/update

microsoft.directory/servicePrincipals/create
microsoft.directory/servicePrincipals/standard/read
microsoft.directory/servicePrincipals/basic/update
microsoft.directory/servicePrincipals/credentials/update
microsoft.directory/servicePrincipals/appRoleAssignedTo/update
```

while intentionally NOT granting:

```text id="mjlwm7"
microsoft.directory/users/create
microsoft.directory/groups/create
microsoft.directory/applications/delete
microsoft.directory/servicePrincipals/delete
microsoft.directory/applications/allProperties/read
microsoft.directory/servicePrincipals/allProperties/read
```

---

# Important Enterprise Recommendation

If possible, Microsoft’s built-in:

* Application Developer

is often safer than trying to manually reconstruct all required permissions unless you have a very specific governance requirement. ([Microsoft Learn][1])

Because custom app-management roles can become tricky:

* some operations silently fail
* ownership behavior changes
* portal experiences partially work
* undocumented dependencies appear

This is a recurring theme in Entra admin community discussions. ([reddit.com][3])

[1]: https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference?utm_source=chatgpt.com "Microsoft Entra built-in roles - Microsoft Entra ID | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/%20entra/identity/role-based-access-control/custom-available-permissions?utm_source=chatgpt.com "Custom role permissions for app registration - Microsoft Entra ID | Microsoft Learn"
[3]: https://www.reddit.com/r/entra/comments/1t49p73/create_custom_roles_to_split_api_permissions_of/?utm_source=chatgpt.com "Create custom roles to split API Permissions of Entra ID Application administrator builtin role"

