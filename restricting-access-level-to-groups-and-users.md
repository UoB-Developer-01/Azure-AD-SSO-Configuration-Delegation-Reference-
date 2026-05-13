```bash
# Prompt

This test user seems to be able to create groups and view user information : SSO App Creator Name: SSO App Creator Description: Can create SSO apps Template ID: 06523d44-31a4-47a0-88122-3c104a0eb9c1 

Role permissions 

- microsoft.directory/applications/standard/read Read standard properties of applications 
- microsoft.directory/servicePrincipals/create Create service principals 
- microsoft.directory/servicePrincipals/standard/read Read basic properties of service principals
```

---



Yes — what you are seeing is actually expected behavior in Microsoft Entra ID unless you additionally restrict the tenant’s **default member user permissions**.

Your custom role itself is **not** granting user/group administration permissions.

The issue is that the account is still a normal **member user**, and member users in Entra have fairly broad default read capabilities unless explicitly hardened. Microsoft documents this directly. ([Microsoft Learn][1])

Your role only grants:

```text id="7e5e1j"
microsoft.directory/applications/standard/read
microsoft.directory/servicePrincipals/create
microsoft.directory/servicePrincipals/standard/read
```

Those permissions alone do NOT include:

* `microsoft.directory/users/create`
* `microsoft.directory/groups/create`
* `microsoft.directory/users/allProperties/read`
* tenant-wide app admin permissions

So the behavior is coming from baseline Entra defaults, not from your custom role itself. ([Microsoft Learn][2])

---

# Why the User Can Create Groups

Microsoft explicitly states that by default:

> “Member users can create security groups” and “Create Microsoft 365 groups.” ([Microsoft Learn][1])

So unless you disable those tenant settings, *all normal member users* can often create groups.

That includes your test user.

---

# Why the User Can View User Information

Microsoft also explicitly documents:

> “Member users … can read all directory information (with a few exceptions).” ([Microsoft Learn][1])

This includes things like:

* Display names
* Email addresses
* Departments
* Group names
* User enumeration

This is default Entra behavior for member users.

Again:
that access is NOT coming from your custom app role.

---

# How to Properly Lock This Down

You need additional tenant hardening beyond the custom role.

---

# 1. Disable Group Creation

Go to:

```text id="mvic1l"
Identity
→ Groups
→ General
```

Set:

```text id="3i0yqv"
Users can create security groups = No
Users can create Microsoft 365 groups = No
```

Microsoft documents these settings under default user permissions. ([Microsoft Learn][3])

---

# 2. Restrict Reading Other Users (Advanced / Potentially Breaking)

Microsoft now documents a newer tenant flag:

```text id="mk9r26"
AllowedToReadOtherUsers = false
```

This can prevent non-admins from reading directory user information. ([Microsoft Learn][3])

However Microsoft warns:

> “This flag might prevent reading user information in other Microsoft services like Microsoft Teams.” ([Microsoft Learn][3])

So this is considered an advanced/special-case hardening setting.

---

# 3. Restrict Admin Center Access

Go to:

```text id="c0wl4k"
Identity
→ Users
→ User settings
→ Restrict access to Microsoft Entra administration portal
```

Set:

```text id="z8cx4o"
Yes
```

BUT Microsoft explicitly warns:

> “It is not a security measure.” ([Microsoft Learn][3])

It mainly reduces casual browsing.

---

# 4. Consider Using Guest-Type Accounts for Highly Restricted Developers

Microsoft documents that guest users have much more limited directory visibility compared to member users. ([Microsoft Learn][1])

Guests typically cannot:

* enumerate all users
* enumerate all groups
* broadly browse the directory

This is one reason some enterprises use B2B guest developers for external vendors/contractors.

---

# 5. Your Current Custom Role Is Actually Missing Application Creation

Your current role:

```text id="ngjlwm"
microsoft.directory/applications/standard/read
microsoft.directory/servicePrincipals/create
microsoft.directory/servicePrincipals/standard/read
```

does NOT appear to include:

```text id="2ht3sq"
microsoft.directory/applications/create
```

Without that permission, the user technically should not be able to create app registrations directly unless another permission path exists.

Your original recommended role was more correct:

```text id="jlwm7r"
microsoft.directory/applications/create
microsoft.directory/applications/owners/update
microsoft.directory/servicePrincipals/create
```

---

# Important Enterprise Reality

In Entra ID, there is a major distinction between:

| Capability                          | Meaning                                |
| ----------------------------------- | -------------------------------------- |
| Administrative permissions          | Create/manage/delete users/groups/apps |
| Default member directory visibility | Read basic directory metadata          |

Your custom role controls the first category.

The second category comes from Entra’s baseline member-user model. ([Microsoft Learn][1])

That is why your user can still:

* browse users
* see groups
* potentially create groups

despite having a narrowly scoped app-management role.

---

# Most Accurate Enterprise Summary

A technically precise version would be:

```text id="d4wl1r"
Custom application roles can restrict developers to managing only owned applications and prevent tenant-wide administration.

However, Microsoft Entra member users still inherit default directory capabilities unless separately restricted, including:
- reading basic directory information,
- enumerating users/groups,
- and potentially creating groups.

Additional tenant hardening is required to restrict those baseline member-user capabilities.
```

[1]: https://learn.microsoft.com/en-us/entra/fundamentals/users-default-permissions?context=graph%2Fcontext&utm_source=chatgpt.com "Default user permissions - Microsoft Entra | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference?utm_source=chatgpt.com "Microsoft Entra built-in roles - Microsoft Entra ID | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/entra/fundamentals/users-default-permissions?utm_source=chatgpt.com "Default user permissions - Microsoft Entra | Microsoft Learn"

# Prompt #2: 

```bash
# prompt

can they still at least assign groups to applications given these settings
```

Yes — most likely they still can assign groups to applications **that they own**, and this is expected behavior.

Microsoft explicitly documents that:

> “Owner of the service principal” can assign users and groups to an enterprise application. ([Microsoft Learn][1])

Specifically, the prerequisites section for assigning users/groups to an enterprise app includes:

* Cloud Application Administrator
* Application Administrator
* User Administrator
* **Owner of the service principal** ([Microsoft Learn][1])

So if your developer:

* created the application
* became owner of the app/service principal

then they can usually:

* open:

  ```text
  Enterprise Applications
  → [Their App]
  → Users and groups
  ```
* assign users
* assign groups
* remove assignments

for that owned application.

---

# Important Distinction

This does **NOT** mean they can:

* manage the group itself
* change group membership
* create groups (if you disabled it)
* administer all enterprise apps

It only means:

```text id="aj8f5w"
They can use existing groups
as access assignments
for applications they own.
```

That is a major architectural distinction in Entra.

---

# What They Actually Need for Group Assignment

There are effectively two separate capabilities involved:

| Capability               | Separate Permission Domain      |
| ------------------------ | ------------------------------- |
| Assign a group to an app | App/service principal ownership |
| Modify the group itself  | Group permissions               |

Microsoft treats these separately. ([Microsoft Learn][1])

So a developer may be able to:

```text id="rjlwm3"
App X
  ↓
Assign Group A to App X
```

while still being unable to:

```text id="a9tt2u"
Add/remove users from Group A
Delete Group A
Create new groups
```

---

# What Happens If You Disable Group Creation

If you configure:

```text id="3z1gbf"
Users can create security groups = No
Users can create Microsoft 365 groups = No
```

then developers generally:

* can still assign EXISTING groups to owned apps
* cannot create NEW groups themselves

These are independent controls. ([Microsoft Learn][1])

---

# Important Security Implication

This means a developer could potentially:

```text id="qvjlwm"
Assign a broad existing group
(e.g. All Employees)
to their application
```

IF they can see/select that group.

So if you want tighter governance, you may additionally want:

* approval processes
* app assignment reviews
* restricted group visibility
* administrative review of enterprise apps
* access packages / entitlement management
* naming conventions and governance

---

# Very Important Licensing Nuance

Microsoft also documents:

> “Group-based assignment requires Microsoft Entra ID P1 or P2 edition.” ([Microsoft Learn][1])

Without P1/P2:

* user assignment still works
* group assignment may be unavailable

---

# Most Accurate Enterprise Summary

The most technically accurate statement is:

```text id="nuvt7w"
Application owners can generally assign existing users and groups to applications they own.

However, this does not inherently grant permission to:
- create groups,
- modify group membership,
- manage groups tenant-wide,
- or manage applications they do not own.
```

That wording aligns closely with Microsoft’s documented RBAC separation model.

[1]: https://learn.microsoft.com/fi-fi/entra/identity/enterprise-apps/assign-user-or-group-access-portal?utm_source=chatgpt.com "Manage users and groups assignment to an application - Microsoft Entra ID | Microsoft Learn"
