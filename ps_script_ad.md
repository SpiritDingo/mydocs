Вот пример PowerShell-скрипта, который использует группы из указанного OU. Предполагается, что группы созданы по шаблону именования:

```powershell
# PowerShell Script for Linux Server Access Management
# Groups are located in OU=Linux,OU=Groups,OU=SPECIAL,DC=domain,DC=ru

# Import Active Directory module
Import-Module ActiveDirectory

# Define server name (change accordingly)
$ServerName = "server1"

# Define group names based on naming convention
$ACLAccessGroup = "acl_access_$ServerName"
$ADMLocalAdminsGroup = "ADM_localadmins_$ServerName"
$ACLPowerUsersGroup = "ACL_PowerUsers_$ServerName"

# Verify groups exist in target OU
try {
    $GroupCheck = Get-ADGroup -Identity $ACLAccessGroup -Properties DistinguishedName
    if ($GroupCheck.DistinguishedName -notmatch "OU=Linux,OU=Groups,OU=SPECIAL,DC=domain,DC=ru") {
        Write-Warning "Group $ACLAccessGroup found in wrong OU"
        exit 1
    }
    
    $GroupCheck = Get-ADGroup -Identity $ADMLocalAdminsGroup -Properties DistinguishedName
    if ($GroupCheck.DistinguishedName -notmatch "OU=Linux,OU=Groups,OU=SPECIAL,DC=domain,DC=ru") {
        Write-Warning "Group $ADMLocalAdminsGroup found in wrong OU"
        exit 1
    }
    
    $GroupCheck = Get-ADGroup -Identity $ACLPowerUsersGroup -Properties DistinguishedName
    if ($GroupCheck.DistinguishedName -notmatch "OU=Linux,OU=Groups,OU=SPECIAL,DC=domain,DC=ru") {
        Write-Warning "Group $ACLPowerUsersGroup found in wrong OU"
        exit 1
    }
}
catch {
    Write-Error "One or more groups not found: $($_.Exception.Message)"
    exit 1
}

# Main script logic (пример использования групп)
Write-Host "Applying access rules for $ServerName" -ForegroundColor Green
Write-Host "Access Group: $ACLAccessGroup" -ForegroundColor Cyan
Write-Host "Local Admins Group: $ADMLocalAdminsGroup" -ForegroundColor Cyan
Write-Host "Power Users Group: $ACLPowerUsersGroup" -ForegroundColor Cyan

# Пример интеграции с sudoers (требуется дополнительная доработка)
# $SudoersConfig = @"
# %$($ADMLocalAdminsGroup) ALL=(ALL) ALL
# %$($ACLPowerUsersGroup) ALL=(ALL) /usr/bin/systemctl restart httpd
# "@

# Set-Content -Path "/etc/sudoers.d/stada_access" -Value $SudoersConfig

Write-Host "Configuration completed successfully" -ForegroundColor Green
```

Важные замечания:

1. Скрипт требует модуль ActiveDirectory
2. Автоматически генерирует имена групп по шаблону:
   · acl_access_<имя_сервера>
   · ADM_localadmins_<имя_сервера>
   · ACL_PowerUsers_<имя_сервера>
3. Проверяет расположение групп в целевом OU
4. Содержит пример интеграции с sudoers (закомментирован)

Для использования:

1. Укажите правильное имя сервера в переменной $ServerName
2. Убедитесь, что группы созданы в правильном OU
3. Раскомментируйте нужные секции в зависимости от задач

Пример создания групп заранее (требуется права AD администратора):

```powershell
New-ADGroup -Name "acl_access_server1" -GroupScope DomainLocal -Path "OU=Linux,OU=Groups,OU=SPECIAL,DC=domain,DC=ru"
New-ADGroup -Name "ADM_localadmins_server1" -GroupScope DomainLocal -Path "OU=Linux,OU=Groups,OU=SPECIAL,DC=domain,DC=ru"
New-ADGroup -Name "ACL_PowerUsers_server1" -GroupScope DomainLocal -Path "OU=Linux,OU=Groups,OU=SPECIAL,DC=domain,DC=ru"
```