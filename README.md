# 🔄 Hybrid Identity Management — Entra ID & Active Directory
### PowerShell & Microsoft Graph Runbook for Sync Mode Switching and ConsistencyGuid Management

> Switch users between **Hybrid** and **Cloud-only** sync modes, and precisely control the `mS-DS-ConsistencyGuid` attribute — all via PowerShell and Microsoft Graph API.

---

## 📋 Overview

In hybrid Microsoft 365 environments, identity lifecycle events sometimes require manually toggling a user's sync mode or correcting their **ImmutableId / ConsistencyGuid** — for example during offboarding, tenant migrations, mailbox conversions, or break-glass recovery scenarios.

This runbook covers four core operations:

| # | Operation | Tool Used |
|---|-----------|-----------|
| 1 | Convert **Hybrid → Cloud-only** | Microsoft Graph API (PATCH) |
| 2 | Convert **Cloud-only → Hybrid** | Microsoft Graph PowerShell SDK |
| 3 | **Get** AD User `ConsistencyGuid` | Active Directory PowerShell |
| 4 | **Set** AD User `ConsistencyGuid` | Active Directory PowerShell |

---

## ⚙️ Prerequisites

| Requirement | Details |
|---|---|
| PowerShell | 5.1+ or PowerShell 7+ |
| Modules | `ActiveDirectory`, `Microsoft.Graph.Users` |
| Permissions | `User.ReadWrite.All` (Microsoft Graph) |
| AD Rights | Write access to user objects in AD |
| Graph Access | `Connect-MgGraph` authenticated session |

Install the Microsoft Graph module if not already present:

```powershell
Install-Module Microsoft.Graph.Users -Scope CurrentUser
```

---

## 📖 Operations

---

### 1️⃣ Change Hybrid → Cloud-only

Removes the on-premises sync relationship for a user, making them a **cloud-managed account**. Use this when a user has been offboarded from AD sync or moved to a cloud-only lifecycle.

> ⚠️ **Warning:** This action removes the `ImmutableId` binding. The user will no longer sync from on-premises AD. Ensure this is intentional before proceeding.

**Endpoint:**
```
PATCH https://graph.microsoft.com/v1.0/users/{userId}/onPremisesSyncBehavior
```

**Request Body:**
```json
{
  "onPremisesSyncEnabled": false
}
```

**PowerShell (via Invoke-MgGraphRequest):**
```powershell
# ── Target User ─────────────────────────────────────────────────
$userId = "e8f9a4d4-72ba-4808-86e2-0ffdfc434848"   # Object ID or UPN
# ────────────────────────────────────────────────────────────────

Connect-MgGraph -Scopes "User.ReadWrite.All"

Invoke-MgGraphRequest -Method PATCH `
    -Uri "https://graph.microsoft.com/v1.0/users/$userId/onPremisesSyncBehavior" `
    -Body @{ onPremisesSyncEnabled = $false } `
    -ContentType "application/json"

Write-Host "✅ User converted to Cloud-only successfully." -ForegroundColor Green
```

---

### 2️⃣ Change Cloud-only → Hybrid

Re-links a cloud user back to their on-premises AD object by stamping the **ImmutableId** (Base64-encoded ObjectGUID). Use this during re-hydration, tenant migration rollback, or when re-enabling AD sync for a previously cloud-only account.

> ℹ️ The `ImmutableId` must match the AD user's `ObjectGUID` (Base64 encoded) for the sync engine to correctly correlate identities.

```powershell
# ── Step 1: Retrieve ObjectGUID from AD and encode as ImmutableId ─
$samId       = "<SAMID>"                            # Replace with the user's SAM Account Name
$guid        = (Get-ADUser -Identity $samId).ObjectGUID
$immutableID = [System.Convert]::ToBase64String($guid.ToByteArray())

Write-Host "ImmutableId: $immutableID"

# ── Step 2: Authenticate to Microsoft Graph ───────────────────────
Connect-MgGraph -Scopes "User.ReadWrite.All"

# ── Step 3: Stamp the ImmutableId on the Entra ID user object ─────
$upn = "timtim@contoso.com"                         # Replace with target UPN

Update-MgUser -UserId $upn -OnPremisesImmutableId $immutableID

Write-Host "✅ User linked back to Hybrid sync successfully." -ForegroundColor Green
```

---

### 3️⃣ Get AD User ConsistencyGuid

Reads the `mS-DS-ConsistencyGuid` attribute from an on-premises AD user. This attribute, when populated, is used as the **source anchor** by Azure AD Connect — taking precedence over `ObjectGUID`.

```powershell
# ── Target User ──────────────────────────────────────────────────
$samAccountName = "vihaan.gupta"                    # Replace with SAM Account Name
# ────────────────────────────────────────────────────────────────

$user  = Get-ADUser -Identity $samAccountName `
                    -Properties "mS-DS-ConsistencyGuid", "ObjectGUID"

$bytes = $user."mS-DS-ConsistencyGuid"

Write-Host "`n── User: $($user.SamAccountName) ──────────────────────"
Write-Host "ObjectGUID            : $($user.ObjectGUID)"

if ($bytes) {
    $consistencyGuid = [Guid]::new($bytes)
    Write-Host "mS-DS-ConsistencyGuid : $consistencyGuid" -ForegroundColor Cyan
} else {
    Write-Host "mS-DS-ConsistencyGuid : (not set — ObjectGUID is used as source anchor)" `
               -ForegroundColor Yellow
}
```

**Sample Output:**
```
── User: vihaan.gupta ──────────────────────
ObjectGUID            : a1b2c3d4-0011-2233-aabb-001122334455
mS-DS-ConsistencyGuid : f5630cde-f3bf-46d2-b8ed-a70f2855e7fb
```

---

### 4️⃣ Set AD User ConsistencyGuid

Writes a specific GUID value into `mS-DS-ConsistencyGuid` on an AD user. Use this to **pin the source anchor** before enabling Azure AD Connect, or to correct a mismatched ImmutableId after a migration.

> ⚠️ **Warning:** Setting an incorrect `ConsistencyGuid` can break identity matching in Azure AD Connect. Always note the existing value first (see Operation 3) before overwriting.

```powershell
# ── Configuration ────────────────────────────────────────────────
$identity = "vihaan.gupta"                                         # samAccountName, DN, or UPN
$guid     = [Guid]"f5630cde-f3bf-46d2-b8ed-a70f2855e7fb"          # Target GUID value
# ────────────────────────────────────────────────────────────────

# ── Step 1: Convert GUID to byte array and write to AD ───────────
$bytes = $guid.ToByteArray()

Set-ADUser -Identity $identity -Replace @{ "mS-DS-ConsistencyGuid" = $bytes }

Write-Host "✅ mS-DS-ConsistencyGuid written successfully." -ForegroundColor Green

# ── Step 2: Read back and verify ─────────────────────────────────
$u          = Get-ADUser -Identity $identity -Properties "mS-DS-ConsistencyGuid"
$storedGuid = [Guid]::new($u."mS-DS-ConsistencyGuid")

Write-Host "🔍 Verification — Stored GUID: $storedGuid"

if ($storedGuid -eq $guid) {
    Write-Host "✅ GUID matches expected value." -ForegroundColor Green
} else {
    Write-Host "❌ GUID mismatch — please review!" -ForegroundColor Red
}
```

---

## 🔁 Decision Flow

```
         Is the user currently Hybrid-synced?
                    │
          ┌─────────┴──────────┐
         YES                   NO
          │                    │
    Convert to              Convert to
    Cloud-only               Hybrid
  (Operation 1)           (Operation 2)
                               │
                    Does mS-DS-ConsistencyGuid
                         need to be set?
                    ┌──────────┴──────────┐
                   YES                    NO
                    │                     │
              Set ConsistencyGuid    ObjectGUID used
               (Operation 4)        as source anchor
```

---

## 🔐 Permissions Reference

| Scope / Right | Required For |
|---|---|
| `User.ReadWrite.All` (Graph) | Operations 1 & 2 |
| AD Write on `mS-DS-ConsistencyGuid` | Operation 4 |
| AD Read on user objects | Operations 3 & 4 |

---

## 📚 Further Reading

- 📖 [Azure AD Connect: Design Concepts & Source Anchor](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/plan-connect-design-concepts#sourceanchor) — Microsoft Docs
- 📖 [Microsoft Graph — Update User](https://learn.microsoft.com/en-us/graph/api/user-update) — Microsoft Docs
- 📖 [mS-DS-ConsistencyGuid as sourceAnchor](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/plan-connect-design-concepts#using-ms-ds-consistencyguid-as-sourceanchor) — Microsoft Docs

---

## 🤝 Contributing

Pull requests and issue reports are welcome. Please open an issue first for significant changes.

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).
