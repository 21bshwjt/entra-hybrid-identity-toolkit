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
