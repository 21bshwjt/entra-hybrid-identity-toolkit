### Change Hybrid to Cloud-only
```powershell
# PATCH
https://graph.microsoft.com/v1.0/users/e8f9a4d4-72ba-4808-86e2-0ffdfc434848/onPremisesSyncBehavior
```

### Change Cloud-only to Hybrid
```powershell
$guid = (Get-ADUser -Identity "<SAMID>").ObjectGUID
$immutableID = [System.Convert]::ToBase64String($guid.ToByteArray())
$immutableID

Connect-MgGraph -Scopes "User.ReadWrite.All"
Update-MgUser -UserId "timtim@contoso.com" -OnPremisesImmutableId $immutableID 
```
### Get AD User ConsistencyGuid
```powershell
$user = Get-ADUser -Identity "vihaan.gupta" -Properties "mS-DS-ConsistencyGuid","ObjectGUID"
$user."mS-DS-ConsistencyGuid"

$bytes = $user."mS-DS-ConsistencyGuid"
if ($bytes) {
    [Guid]::new($bytes)
} else {
    "mS-DS-ConsistencyGuid is empty"
}
```

### Set AD User ConsistencyGuid
```powershell
$identity = "vihaan.gupta"  # samAccountName or distinguishedName or UPN
$guid     = [Guid]"f5630cde-f3bf-46d2-b8ed-a70f2855e7fb"

# Convert GUID to 16-byte array and write it to mS-DS-ConsistencyGuid
$bytes = $guid.ToByteArray()
Set-ADUser -Identity $identity -Replace @{ "mS-DS-ConsistencyGuid" = $bytes }

# Verify
$u = Get-ADUser -Identity $identity -Properties "mS-DS-ConsistencyGuid"
$storedGuid = [Guid]::new($u."mS-DS-ConsistencyGuid")
"Stored GUID: $storedGuid"
```

