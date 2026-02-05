# Technical Implementation Guide: Hybrid Identity Lab

## 📐 Architecture Overview
```
┌─────────────────────────────────────────────────────────────────┐
│                        Microsoft Entra ID                        │
│                     (porschecloud.com tenant)                    │
│                                                                   │
│  ┌───────────────┐  ┌──────────────┐  ┌──────────────────┐    │
│  │  Synced Users │  │   Groups     │  │  Authentication  │    │
│  │  • Alice HR   │  │  • GG-HR     │  │    Methods       │    │
│  │  • Bob Finance│  │  • GG-Finance│  │  • PTA (Primary) │    │
│  │  • IAM Admin  │  │  • GG-SSPR   │  │  • PHS (Backup)  │    │
│  └───────────────┘  └──────────────┘  └──────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ Entra Connect Sync
                              │ + PTA Agent
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              On-Premises Infrastructure (VMnet10)                │
│                     Network: 10.10.10.0/24                       │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐ │
│  │   DC01           │  │   ADFS01         │  │ AADCONNECT01  │ │
│  │  10.10.10.10     │  │  10.10.10.20     │  │ 10.10.10.30   │ │
│  │                  │  │                  │  │               │ │
│  │  • AD DS         │  │  • AD FS         │  │ • Sync Engine │ │
│  │  • DNS           │  │  • IDP Sign-on   │  │ • PTA Agent   │ │
│  │  • AD CS (CA)    │  │  • SSL Cert      │  │ • PWD Writer  │ │
│  │  • Certificate   │  │  • gMSA Service  │  │               │ │
│  │    Templates     │  │                  │  │               │ │
│  └──────────────────┘  └──────────────────┘  └───────────────┘ │
│           │                     │                     │          │
│           └─────────────────────┴─────────────────────┘          │
│                    Domain: porsche.local                         │
│                                                                   │
│              ┌──────────────────────────┐                        │
│              │   WIN11-CLIENT           │                        │
│              │   10.10.10.100           │                        │
│              │   • Domain Joined        │                        │
│              │   • Test Workstation     │                        │
│              └──────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Authentication Flow Diagram

### Pass-through Authentication (PTA) Flow
```
┌─────────────┐
│   User      │
│ AliceHR@    │
│ porschecloud│
│   .com      │
└──────┬──────┘
       │ 1. Sign-in attempt
       ▼
┌─────────────────────────────────┐
│   Microsoft Entra ID             │
│   (Cloud)                        │
│                                  │
│  "Is this a PTA user?"          │
│  → Yes, forward to PTA agent     │
└────────────┬────────────────────┘
             │ 2. Auth request via
             │    Service Bus
             ▼
      ┌──────────────────┐
      │  AADCONNECT01    │
      │  PTA Agent       │
      │                  │
      │  3. Decrypt &    │
      │     validate     │
      └────────┬─────────┘
               │ 4. LDAP bind with
               │    credentials
               ▼
        ┌─────────────┐
        │    DC01     │
        │   (AD DS)   │
        │             │
        │ 5. Validate │
        │    password │
        └──────┬──────┘
               │ 6. Success/Fail
               ▼
      ┌──────────────────┐
      │  AADCONNECT01    │
      │  PTA Agent       │
      │                  │
      │  7. Return result│
      └────────┬─────────┘
               │ 8. Auth response
               ▼
┌─────────────────────────────────┐
│   Microsoft Entra ID             │
│                                  │
│  9. Issue tokens                 │
└────────────┬────────────────────┘
             │ 10. Access granted
             ▼
      ┌─────────────┐
      │   User      │
      │  Access to  │
      │   M365      │
      └─────────────┘
```

---

## 📊 Active Directory Structure
```
porsche.local
│
├── Builtin
│   └── [Default built-in accounts and groups]
│
├── Computers
│   └── [Default computer container]
│
├── Domain Controllers
│   └── DC01
│
├── Users [DEFAULT - Not synced]
│   └── [Built-in accounts]
│
├── Groups [✓ SYNCED TO ENTRA ID]
│   ├── GG-HR
│   ├── GG-Finance
│   ├── GG-IAM-Admins
│   ├── GG-SSPR-Pilot
│   ├── GG-App-OIDC-Users
│   └── GG-App-SaaS-SAML-Users
│
├── Users [✓ SYNCED TO ENTRA ID]
│   ├── AliceHR
│   │   ├── UPN: AliceHR@porschecloud.com
│   │   └── Member of: GG-HR, GG-SSPR-Pilot
│   │
│   ├── BobFinance
│   │   ├── UPN: bobfinance@porschecloud.com
│   │   └── Member of: GG-Finance
│   │
│   └── IAMAdmin
│       ├── UPN: IAMAdmin@porschecloud.com
│       └── Member of: GG-IAM-Admins
│
├── Servers
│   ├── ADFS01
│   └── AADCONNECT01
│
├── Workstations
│   └── WIN11-CLIENT
│
├── Service Accounts
│   ├── svc_domainjoin
│   └── gmsa_adfs$
│
└── Managed Service Accounts
    └── gmsa_adfs$
```

---

## 🔐 Certificate Chain & Trust Flow
```
┌────────────────────────────────────────────────┐
│        Trusted Root Certification              │
│              Authorities Store                 │
│                                                │
│  ┌──────────────────────────────────────┐    │
│  │  porsche-DC01-CA                     │    │
│  │  (Enterprise Root CA)                │    │
│  │                                       │    │
│  │  Issued to: porsche-DC01-CA          │    │
│  │  Issued by: porsche-DC01-CA          │    │
│  │  Valid: 1/29/2031                    │    │
│  │                                       │    │
│  │  [Automatically trusted by all       │    │
│  │   domain-joined machines via GPO]    │    │
│  └──────────────┬───────────────────────┘    │
└─────────────────┼────────────────────────────┘
                  │
                  │ Issues certificates using
                  │ published templates
                  ▼
         ┌────────────────────┐
         │  Certificate       │
         │  Templates         │
         │                    │
         │  • Web Server      │
         │  • Computer        │
         │  • User            │
         │  • Domain Controller│
         └─────────┬──────────┘
                   │
                   │ Issues certificate to
                   ▼
         ┌──────────────────────────┐
         │  Server Certificate      │
         │  (fs.porsche.local)      │
         │                          │
         │  Subject: fs.porsche.local│
         │  SAN: fs.porsche.local   │
         │  Issued by: porsche-DC01-CA│
         │  Valid: 1/30/2028        │
         │                          │
         │  Installed on: ADFS01    │
         │  Binding: HTTPS (443)    │
         └──────────────────────────┘
                   │
                   │ Used for SSL/TLS
                   ▼
         ┌──────────────────────┐
         │  AD FS Federation    │
         │  Service             │
         │                      │
         │  https://fs.porsche  │
         │        .local        │
         └──────────────────────┘
```

---

## 📋 Phase-by-Phase Implementation

---

## Phase 1: Infrastructure Foundation

### Network Setup

**VMware Virtual Network Editor Configuration:**
```
Network: VMnet10
Type: Host-only
Subnet IP: 10.10.10.0
Subnet Mask: 255.255.255.0
DHCP: Enabled (10.10.10.128 - 10.10.10.254)

[✓] Connect a host virtual adapter to this network
Host Adapter: VMware Network Adapter VMnet10
```

**Static IP Assignment Plan:**

| Hostname | IP Address | Subnet Mask | Default Gateway | DNS Primary | DNS Secondary |
|----------|-----------|-------------|-----------------|-------------|---------------|
| DC01 | 10.10.10.10 | 255.255.255.0 | - | 10.10.10.10 | - |
| ADFS01 | 10.10.10.20 | 255.255.255.0 | - | 10.10.10.10 | - |
| AADCONNECT01 | 10.10.10.30 | 255.255.255.0 | - | 10.10.10.10 | - |
| WIN11-CLIENT | 10.10.10.100 | 255.255.255.0 | - | 10.10.10.10 | - |

### Virtual Machine Creation

**DC01 Virtual Machine:**
```
VM Name: DC01
Guest OS: Windows Server 2016 Standard (64-bit)
Processors: 2
Memory: 4 GB (4096 MB)
Hard Disk: 60 GB (Thin provisioned)
Network Adapter: Custom (VMnet10)
ISO: Windows Server 2016 Standard Evaluation
```

**Installation Steps:**
1. Mount Windows Server 2016 ISO
2. Boot VM and select "Windows Server 2016 Standard Evaluation (Desktop Experience)"
3. Custom installation to disk
4. Set Administrator password
5. Complete initial setup

**Post-Installation Configuration:**
```powershell
# Rename computer to DC01
Rename-Computer -NewName "DC01" -Restart

# Configure static IP
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 10.10.10.10 `
    -PrefixLength 24 -DefaultGateway 10.10.10.1

Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" `
    -ServerAddresses 10.10.10.10

# Verify configuration
Get-NetIPAddress -InterfaceAlias "Ethernet0"
Get-DnsClientServerAddress -InterfaceAlias "Ethernet0"
```

**Verification Screenshot Locations:**
- Network adapter showing static IP configuration
- PowerShell output of `ipconfig /all`
- System Properties showing computer name "DC01"

---

## Phase 2: Active Directory Domain Services

### AD DS Installation

**Install AD DS Role:**
```powershell
# Install AD DS and management tools
Install-WindowsFeature -Name AD-Domain-Services `
    -IncludeManagementTools

# Import AD DS Deployment module
Import-Module ADDSDeployment
```

**Screenshot: Server Manager showing AD DS role installation progress**

### Domain Controller Promotion

**Promote to Domain Controller:**
```powershell
Install-ADDSForest `
    -DomainName "porsche.local" `
    -DomainNetbiosName "PORSCHE" `
    -ForestMode "WinThreshold" `
    -DomainMode "WinThreshold" `
    -InstallDns:$true `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) `
    -Force:$true
```

**Configuration Details:**
- **Forest Functional Level:** Windows Server 2016
- **Domain Functional Level:** Windows Server 2016
- **DNS:** Installed and configured automatically
- **NetBIOS Name:** PORSCHE
- **DSRM Password:** (Secure password set)

**Server automatically reboots after promotion**

### Post-Promotion Verification
```powershell
# Verify domain controller services
Get-Service ADWS,DNS,NTDS | Format-Table Name,Status,DisplayName

# Expected output:
# Name  Status  DisplayName
# ----  ------  -----------
# ADWS  Running Active Directory Web Services
# DNS   Running DNS Server
# NTDS  Running Active Directory Domain Services

# Verify domain and forest
Get-ADDomain | Select-Object Name,Forest,DomainMode
Get-ADForest | Select-Object Name,ForestMode

# Check DNS zones
Get-DnsServerZone | Select-Object ZoneName,ZoneType
```

**Expected DNS Zones:**
- `porsche.local` (Primary, AD-integrated)
- `_msdcs.porsche.local` (Primary, AD-integrated)
- `10.10.10.in-addr.arpa` (Reverse lookup zone)

**Screenshot: DNS Manager showing forward and reverse lookup zones**

### Organizational Unit (OU) Structure

**Create OU Hierarchy:**
```powershell
# Create top-level OUs
New-ADOrganizationalUnit -Name "Groups" -Path "DC=porsche,DC=local"
New-ADOrganizationalUnit -Name "Users" -Path "DC=porsche,DC=local"
New-ADOrganizationalUnit -Name "Servers" -Path "DC=porsche,DC=local"
New-ADOrganizationalUnit -Name "Workstations" -Path "DC=porsche,DC=local"
New-ADOrganizationalUnit -Name "Service Accounts" -Path "DC=porsche,DC=local"

# Verify OU creation
Get-ADOrganizationalUnit -Filter * | Select-Object Name,DistinguishedName
```

**Screenshot: Active Directory Users and Computers showing OU structure**

### Security Groups

**Create Security Groups:**
```powershell
# Create groups in Groups OU
$groupsOU = "OU=Groups,DC=porsche,DC=local"

# HR Group
New-ADGroup -Name "GG-HR" -GroupScope Global -GroupCategory Security `
    -Path $groupsOU -Description "HR Department Access"

# Finance Group
New-ADGroup -Name "GG-Finance" -GroupScope Global -GroupCategory Security `
    -Path $groupsOU -Description "Finance Department Access"

# IAM Admins Group
New-ADGroup -Name "GG-IAM-Admins" -GroupScope Global -GroupCategory Security `
    -Path $groupsOU -Description "Identity and Access Management Administrators"

# SSPR Pilot Group
New-ADGroup -Name "GG-SSPR-Pilot" -GroupScope Global -GroupCategory Security `
    -Path $groupsOU -Description "Self-Service Password Reset Pilot Users"

# OIDC App Users
New-ADGroup -Name "GG-App-OIDC-Users" -GroupScope Global -GroupCategory Security `
    -Path $groupsOU -Description "OIDC Application Access"

# SAML App Users
New-ADGroup -Name "GG-App-SaaS-SAML-Users" -GroupScope Global -GroupCategory Security `
    -Path $groupsOU -Description "SAML Federation Application Access"

# Verify groups
Get-ADGroup -Filter * -SearchBase $groupsOU | Select-Object Name,GroupScope,GroupCategory
```

**Naming Convention:**
- `GG-` prefix = Global Group
- Descriptive name indicates purpose
- Security groups (not distribution)

### User Account Creation

**Create Test Users:**
```powershell
$usersOU = "OU=Users,DC=porsche,DC=local"
$defaultPassword = ConvertTo-SecureString "Welcome2024!" -AsPlainText -Force

# Alice HR
New-ADUser -Name "Alice HR" -GivenName "Alice" -Surname "HR" `
    -SamAccountName "AliceHR" `
    -UserPrincipalName "AliceHR@porsche.local" `
    -Path $usersOU `
    -AccountPassword $defaultPassword `
    -Enabled $true `
    -PasswordNeverExpires $false `
    -ChangePasswordAtLogon $false

# Add Alice to groups
Add-ADGroupMember -Identity "GG-HR" -Members "AliceHR"
Add-ADGroupMember -Identity "GG-SSPR-Pilot" -Members "AliceHR"

# Bob Finance
New-ADUser -Name "Bob Finance" -GivenName "Bob" -Surname "Finance" `
    -SamAccountName "BobFinance" `
    -UserPrincipalName "bobfinance@porsche.local" `
    -Path $usersOU `
    -AccountPassword $defaultPassword `
    -Enabled $true `
    -PasswordNeverExpires $false `
    -ChangePasswordAtLogon $false

Add-ADGroupMember -Identity "GG-Finance" -Members "BobFinance"

# IAM Admin
New-ADUser -Name "IAM Admin" -GivenName "IAM" -Surname "Admin" `
    -SamAccountName "IAMAdmin" `
    -UserPrincipalName "IAMAdmin@porsche.local" `
    -Path $usersOU `
    -AccountPassword $defaultPassword `
    -Enabled $true `
    -PasswordNeverExpires $false `
    -ChangePasswordAtLogon $false

Add-ADGroupMember -Identity "GG-IAM-Admins" -Members "IAMAdmin"

# Verify users
Get-ADUser -Filter * -SearchBase $usersOU -Properties MemberOf | 
    Select-Object Name,SamAccountName,UserPrincipalName,@{Name="Groups";Expression={$_.MemberOf -join ", "}}
```

**Screenshot: AD Users and Computers showing user accounts and group memberships**

---

## Phase 3: Domain Join Operations

### Create Domain Join Service Account

**Service Account with Delegated Permissions:**
```powershell
$serviceAccountsOU = "OU=Service Accounts,DC=porsche,DC=local"
$svcPassword = ConvertTo-SecureString "ServiceP@ss2024!" -AsPlainText -Force

# Create service account
New-ADUser -Name "Domain Join Service Account" `
    -SamAccountName "svc_domainjoin" `
    -UserPrincipalName "svc_domainjoin@porsche.local" `
    -Path $serviceAccountsOU `
    -AccountPassword $svcPassword `
    -Enabled $true `
    -PasswordNeverExpires $true `
    -CannotChangePassword $true

# Get OU distinguished names for delegation
$serversOU = "OU=Servers,DC=porsche,DC=local"
$workstationsOU = "OU=Workstations,DC=porsche,DC=local"
```

**Delegate Control for Domain Join:**

1. Open Active Directory Users and Computers
2. Right-click **Servers** OU → **Delegate Control**
3. Add user: `svc_domainjoin`
4. Permissions to delegate:
   - ✅ Create Computer objects
   - ✅ Delete Computer objects
5. Repeat for **Workstations** OU

**PowerShell Alternative for Delegation:**
```powershell
# Import AD module
Import-Module ActiveDirectory

# Get service account
$svcAccount = Get-ADUser svc_domainjoin

# Set ACL for Servers OU
$serversACL = Get-Acl "AD:$serversOU"

$identity = $svcAccount.SID
$adRights = [System.DirectoryServices.ActiveDirectoryRights]::CreateChild -bor `
            [System.DirectoryServices.ActiveDirectoryRights]::DeleteChild
$type = [System.Security.AccessControl.AccessControlType]::Allow
$objectType = [GUID]"bf967a86-0de6-11d0-a285-00aa003049e2" # Computer object GUID

$ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule($identity,$adRights,$type,$objectType)
$serversACL.AddAccessRule($ace)
Set-Acl "AD:$serversOU" $serversACL

# Repeat for Workstations OU
$workstationsACL = Get-Acl "AD:$workstationsOU"
$workstationsACL.AddAccessRule($ace)
Set-Acl "AD:$workstationsOU" $workstationsACL
```

**Verification:**
```powershell
# Verify delegated permissions
(Get-Acl "AD:$serversOU").Access | 
    Where-Object {$_.IdentityReference -like "*svc_domainjoin*"} |
    Select-Object IdentityReference,ActiveDirectoryRights,AccessControlType
```

### Join Servers to Domain

**Configure DNS on ADFS01:**

1. Log into ADFS01 VM
2. Open Network and Sharing Center
3. Change adapter settings → Ethernet0 → Properties
4. Select "Internet Protocol Version 4 (TCP/IPv4)" → Properties
5. Configure:
```
   IP address: 10.10.10.20
   Subnet mask: 255.255.255.0
   Preferred DNS server: 10.10.10.10
```

**PowerShell Method:**
```powershell
# On ADFS01
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 10.10.10.20 `
    -PrefixLength 24

Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" `
    -ServerAddresses 10.10.10.10

# Verify DNS resolution
Resolve-DnsName porsche.local
Resolve-DnsName dc01.porsche.local
```

**Join ADFS01 to Domain:**
```powershell
# Join domain using service account
$credential = Get-Credential -UserName "PORSCHE\svc_domainjoin" `
    -Message "Enter domain join credentials"

Add-Computer -DomainName "porsche.local" `
    -Credential $credential `
    -OUPath "OU=Servers,DC=porsche,DC=local" `
    -Restart
```

**GUI Method:**
1. System Properties → Computer Name → Change
2. Domain: `porsche.local`
3. Credentials: `PORSCHE\svc_domainjoin`
4. Computer will restart

**Repeat for AADCONNECT01:**
- IP: 10.10.10.30
- DNS: 10.10.10.10
- Join to domain, place in Servers OU

### Join Windows 11 Client

**Configure DNS and Join:**
```powershell
# On WIN11-CLIENT
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 10.10.10.100 `
    -PrefixLength 24

Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" `
    -ServerAddresses 10.10.10.10

# Join domain
$credential = Get-Credential -UserName "PORSCHE\svc_domainjoin"
Add-Computer -DomainName "porsche.local" `
    -Credential $credential `
    -OUPath "OU=Workstations,DC=porsche,DC=local" `
    -Restart
```

### Move Computer Objects to Correct OUs

**On DC01:**
```powershell
# Get computer objects
$adfs01 = Get-ADComputer -Identity "ADFS01"
$aadconnect01 = Get-ADComputer -Identity "AADCONNECT01"
$win11client = Get-ADComputer -Identity "WIN11-CLIENT"

# Move to correct OUs
Move-ADObject -Identity $adfs01.DistinguishedName `
    -TargetPath "OU=Servers,DC=porsche,DC=local"

Move-ADObject -Identity $aadconnect01.DistinguishedName `
    -TargetPath "OU=Servers,DC=porsche,DC=local"

Move-ADObject -Identity $win11client.DistinguishedName `
    -TargetPath "OU=Workstations,DC=porsche,DC=local"

# Verify placement
Get-ADComputer -Filter * -SearchBase "OU=Servers,DC=porsche,DC=local" | 
    Select-Object Name,DistinguishedName

Get-ADComputer -Filter * -SearchBase "OU=Workstations,DC=porsche,DC=local" | 
    Select-Object Name,DistinguishedName
```

**Screenshot: Computer objects in correct OUs**

### Test Domain Logon

**On WIN11-CLIENT:**

1. Restart WIN11-CLIENT
2. At logon screen, click "Other user"
3. Sign in with domain credentials:
```
   Username: AliceHR
   Password: Welcome2024!
   Domain: PORSCHE
```

**Verify Logon:**
```powershell
# Check logged-on user
whoami
# Output: porsche\alicehr

# Verify domain membership
Get-ComputerInfo | Select-Object CsDomain,CsDomainRole
# CsDomain: porsche.local
# CsDomainRole: MemberWorkstation

# Check group memberships
whoami /groups
```

**Screenshot: Windows 11 showing domain user logged in**

---

## Phase 4: Certificate Services (PKI)

### Install AD CS on DC01

**Install Certificate Services Role:**
```powershell
# Install AD CS role
Install-WindowsFeature -Name AD-Certificate `
    -IncludeManagementTools

# Install CA role service
Install-AdcsCertificationAuthority -CAType EnterpriseRootCA `
    -CryptoProviderName "RSA#Microsoft Software Key Storage Provider" `
    -KeyLength 2048 `
    -HashAlgorithmName SHA256 `
    -ValidityPeriod Years `
    -ValidityPeriodUnits 5 `
    -CACommonName "porsche-DC01-CA" `
    -CADistinguishedNameSuffix "DC=porsche,DC=local" `
    -Force
```

**GUI Method:**
1. Server Manager → Add Roles and Features
2. Select **Active Directory Certificate Services**
3. Add features, click Next
4. Select role service: **Certification Authority**
5. Click Install

**Configure Certification Authority:**

1. Server Manager → Notifications → **Configure Active Directory Certificate Services**
2. Credentials: Use current domain administrator account
3. Role Services: ✅ Certification Authority
4. Setup Type: ● Enterprise CA
5. CA Type: ● Root CA
6. Private Key: ● Create a new private key
7. Cryptography:
```
   Provider: RSA#Microsoft Software Key Storage Provider
   Key length: 2048
   Hash algorithm: SHA256
```
8. CA Name:
```
   Common name: porsche-DC01-CA
   Distinguished name suffix: DC=porsche,DC=local
```
9. Validity Period: 5 years
10. Certificate Database: Default locations
11. Confirm and click Configure

**Verification:**
```powershell
# Check CA service
Get-Service CertSvc | Select-Object Name,Status,DisplayName

# View CA configuration
certutil -CAInfo

# List certificate templates
certutil -CATemplates

# Check if CA is Enterprise Root
certutil -CAInfo type
# Should output: Enterprise Root CA
```

**Screenshot: AD CS configuration complete**

### Publish Web Server Certificate Template

**Make Web Server Template Available:**
```powershell
# Open Certification Authority console
certsrv.msc

# Navigate to Certificate Templates
```

**GUI Steps:**
1. Open **Certification Authority** console
2. Expand **porsche-DC01-CA**
3. Right-click **Certificate Templates** → **Manage**
4. Find **Web Server** template
5. Right-click → **Duplicate Template**
6. Template Properties:
```
   Template name: Web Server
   Template display name: Web Server
   Validity period: 2 years
   Renewal period: 6 weeks
```
7. **Security** tab:
   - Add **ADFS01$** computer account
   - Permissions for ADFS01$:
     - ✅ Read
     - ✅ Enroll

**Publish Template:**

1. Return to Certification Authority console
2. Right-click **Certificate Templates** → **New** → **Certificate Template to Issue**
3. Select **Web Server**
4. Click OK

**Verification:**
```powershell
# List issued templates
certutil -CATemplates

# Should see "Web Server" in list
```

**Screenshot: Certificate Templates showing Web Server template published**

### Create DNS Record for AD FS

**On DC01:**
```powershell
# Create A record for federation service
Add-DnsServerResourceRecordA -Name "fs" `
    -ZoneName "porsche.local" `
    -IPv4Address "10.10.10.20"

# Verify DNS record
Resolve-DnsName fs.porsche.local
```

**Screenshot: DNS Manager showing fs.porsche.local A record**

### Request Certificate on ADFS01

#### Method 1: MMC Console (Initial Attempt - Failed)

**On ADFS01:**
```powershell
# Open MMC console
mmc.exe
```

**Add Certificates Snap-in:**
1. File → Add/Remove Snap-in
2. Select **Certificates** → Add
3. Select **Computer account**
4. Local computer → Finish

**Request Certificate:**
1. Expand **Certificates (Local Computer)** → **Personal**
2. Right-click **Certificates** → **All Tasks** → **Request New Certificate**
3. Click Next through welcome
4. Select **Active Directory Enrollment Policy** → Next
5. Select **Web Server** template
6. Click **More information is required to enroll for this certificate**
7. Certificate Properties:
```
   Subject:
   - Common name: fs.porsche.local
   
   Alternative name:
   - DNS: fs.porsche.local
```
8. Click OK → Enroll

**❌ Result: Certificate did not appear in Personal store**

#### Troubleshooting: Certificate Request Failure

**Problem Analysis:**

1. Checked Certificate Templates on CA:
```powershell
   # On DC01
   certutil -CATemplates
   # Web Server template present
```

2. Verified ADFS01 permissions on template:
   - Computer Templates console → Web Server → Security
   - ADFS01$ has Read and Enroll ✅

3. Checked CA logs:
```
   Event Viewer → Applications and Services Logs 
   → Microsoft → Windows → CertificationAuthority
```

**Root Cause Identified:**

Enterprise CA requires explicit certificate template information in the request. The MMC wizard did not include template details, causing the CA to silently reject the request.

#### Method 2: Manual Certificate Request (Success)

**Create Certificate Request File:**

On ADFS01, create `C:\Temp\request.inf`:
```ini
[Version]
Signature="$Windows NT$"

[NewRequest]
Subject = "CN=fs.porsche.local"
KeySpec = 1
KeyLength = 2048
Exportable = TRUE
MachineKeySet = TRUE
SMIME = FALSE
PrivateKeyArchive = FALSE
UserProtected = FALSE
UseExistingKeySet = FALSE
ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
ProviderType = 12
RequestType = PKCS10
KeyUsage = 0xa0
HashAlgorithm = SHA256

[EnhancedKeyUsageExtension]
OID=1.3.6.1.5.5.7.3.1 ; Server Authentication

[Extensions]
2.5.29.17 = "{text}"
_continue_ = "dns=fs.porsche.local&"

[RequestAttributes]
CertificateTemplate = "WebServer"
```

**Generate and Submit Request:**
```powershell
# Create Temp directory
New-Item -Path "C:\Temp" -ItemType Directory -Force

# Generate certificate request
certreq -new C:\Temp\request.inf C:\Temp\request.req

# Submit request to CA
certreq -submit -config "DC01.porsche.local\porsche-DC01-CA" `
    C:\Temp\request.req C:\Temp\cert.cer

# The CA will process and approve the request

# Install certificate
certreq -accept C:\Temp\cert.cer
```

**✅ Certificate Successfully Issued**

**Verification:**
```powershell
# View certificate in personal store
Get-ChildItem Cert:\LocalMachine\My | 
    Where-Object {$_.Subject -like "*fs.porsche.local*"} |
    Format-List Subject,Issuer,Thumbprint,NotAfter,EnhancedKeyUsageList

# Export for backup (without private key for documentation)
$cert = Get-ChildItem Cert:\LocalMachine\My | 
    Where-Object {$_.Subject -like "*fs.porsche.local*"}

Export-Certificate -Cert $cert -FilePath "C:\Temp\fs-porsche-cert.cer"
```

**Certificate Details:**
```
Subject: CN=fs.porsche.local
Issuer: CN=porsche-DC01-CA, DC=porsche, DC=local
Valid From: [Date]
Valid To: [Date + 2 years]
Thumbprint: [Hash]
Enhanced Key Usage: Server Authentication (1.3.6.1.5.5.7.3.1)
Subject Alternative Name: DNS Name=fs.porsche.local
```

**Screenshot: Certificate in Personal store with full details**

### Verify Root CA Trust

**On ADFS01:**
```powershell
# Check Trusted Root Certification Authorities
Get-ChildItem Cert:\LocalMachine\Root | 
    Where-Object {$_.Subject -like "*porsche-DC01-CA*"}

# Verify GPO-deployed root certificate
gpresult /h C:\Temp\gpresult.html
# Open and check Certificate Services policy
```

**Why Root CA is Trusted:**

1. Enterprise Root CA publishes certificate to AD
2. Group Policy automatically deploys to domain computers
3. Certificate appears in Trusted Root Certification Authorities store
4. All certificates issued by this CA are automatically trusted

**Certificate Chain:**
```
Trusted Root Certification Authorities
└── porsche-DC01-CA (Root CA)
    └── fs.porsche.local (Issued Certificate)
```

**Screenshot: Trusted Root Certification Authorities showing porsche-DC01-CA**

---

## Phase 5: Active Directory Federation Services

### Install AD FS Role on ADFS01

**Install AD FS:**
```powershell
# Install AD FS role
Install-WindowsFeature -Name ADFS-Federation `
    -IncludeManagementTools

# Verify installation
Get-WindowsFeature | Where-Object {$_.Name -like "*ADFS*"}
```

**Screenshot: AD FS role installation completed**

### Create Group Managed Service Account (gMSA)

**On DC01:**
```powershell
# Check if KDS Root Key exists
Get-KdsRootKey

# If not, create KDS Root Key
# NOTE: In production, wait 10 hours for AD replication
# For lab, use -EffectiveImmediately
Add-KdsRootKey -EffectiveImmediately

# Create gMSA for AD FS
New-ADServiceAccount -Name "gmsa_adfs" `
    -DNSHostName "gmsa_adfs.porsche.local" `
    -ServicePrincipalNames "http/fs.porsche.local" `
    -PrincipalsAllowedToRetrieveManagedPassword "ADFS01$"

# Verify gMSA creation
Get-ADServiceAccount gmsa_adfs -Properties *
```

**On ADFS01:**
```powershell
# Install AD module for gMSA support
Install-WindowsFeature RSAT-AD-PowerShell

# Test gMSA
Test-ADServiceAccount gmsa_adfs
# Should return: True

# Install gMSA
Install-ADServiceAccount gmsa_adfs
```

### Configure AD FS Farm

**Launch AD FS Configuration Wizard:**
```powershell
# Start configuration
# Server Manager → Notifications → "Configure the federation service on this server"
```

**Or via PowerShell:**

1. Welcome → Next
2. **Connect to AD DS:**
   - Credentials: `PORSCHE\Administrator`
   - Click Next

3. **Specify Service Properties:**
   - SSL Certificate: Select **fs.porsche.local** (issued by porsche-DC01-CA)
   - Federation Service Name: `fs.porsche.local`
   - Federation Service Display Name: `Porsche`

4. **Specify Service Account:**
   - ● Create a Group Managed Service Account
   - Account Name: `PORSCHE\gmsa_adfs$`

5. **Specify Database:**
   - ● Create a database on this server using Windows Internal Database
   - Database Location: Default

6. **Review Options:**
   - Verify all settings
   - Click Next

7. **Pre-requisite Checks:**
   - All checks should pass
   - Click **Configure**

8. **Configuration Progress:**
   - Wait for configuration to complete
   - **Important:** Server will restart

**Post-Configuration Verification:**
```powershell
# Check AD FS service
Get-Service adfssrv | Select-Object Name,Status,StartType

# Expected:
# Name     Status  StartType
# adfssrv  Running Automatic

# Check AD FS properties
Get-AdfsProperties | Select-Object HostName,DisplayName,Identifier

# Expected:
# HostName       : fs.porsche.local
# DisplayName    : Porsche
# Identifier     : http://fs.porsche.local/adfs/services/trust

# Check certificate bindings
Get-AdfsCertificate

# Check SSL binding
Get-AdfsSslCertificate
```

**Screenshot: AD FS Management console showing farm configuration**

### Enable IdP-Initiated Sign-On

**Issue: Sign-On Page Not Accessible**

Initial attempt to access IdP sign-on page failed:
```
URL: https://fs.porsche.local/adfs/ls/idpinitiatedsignon.aspx
Error: "The resource you are trying to access is not available"
```

**Root Cause:**

IdP-initiated sign-on is disabled by default in AD FS for security reasons. This prevents unsolicited authentication requests.

**Solution:**
```powershell
# Enable IdP-initiated sign-on
Set-AdfsProperties -EnableIdPInitiatedSignonPage $true

# Restart AD FS service
Restart-Service adfssrv

# Verify setting
Get-AdfsProperties | Select-Object EnableIdPInitiatedSignonPage
```

**Test Sign-On Page:**

1. On WIN11-CLIENT, open browser
2. Navigate to: `https://fs.porsche.local/adfs/ls/idpinitiatedsignon.aspx`
3. Page should load showing:
```
   Porsche
   
   You are not signed in. Sign in to this site.
   [Sign in] button
```

**Test Authentication:**

1. Click **Sign in**
2. Enter credentials:
```
   Username: AliceHR
   Password: Welcome2024!
```
3. Should see: "You are signed in"

**Screenshot: IdP sign-on page successfully loading**

### Configure AD FS for Entra ID Integration

**Prepare for Federation Trust:**
```powershell
# Get AD FS federation metadata URL
$federationServiceIdentifier = (Get-AdfsProperties).Identifier
Write-Host "Federation Metadata URL: $federationServiceIdentifier/FederationMetadata/2007-06/FederationMetadata.xml"

# Expected output:
# http://fs.porsche.local/adfs/services/trust/FederationMetadata/2007-06/FederationMetadata.xml

# Verify metadata is accessible
Invoke-WebRequest -Uri "$federationServiceIdentifier/FederationMetadata/2007-06/FederationMetadata.xml" `
    -UseBasicParsing | Select-Object StatusCode,Content
```

**AD FS Configuration Summary:**

| Setting | Value |
|---------|-------|
| Federation Service Name | fs.porsche.local |
| Display Name | Porsche |
| Service Account | PORSCHE\gmsa_adfs$ |
| SSL Certificate | fs.porsche.local (issued by porsche-DC01-CA) |
| Database | Windows Internal Database (WID) |
| Farm Behavior Level | Windows Server 2016 |
| IdP Sign-On | Enabled |

**Screenshot: AD FS Properties showing all configuration**

---

## Phase 6: Hybrid Identity with Entra Connect Sync

### Add Custom Domain to Entra ID

**On DC01 - Add UPN Suffix:**

1. Open **Active Directory Domains and Trusts**
2. Right-click **Active Directory Domains and Trusts** → **Properties**
3. Under **Alternative UPN suffixes**, add: `porschecloud.com`
4. Click **Add** → **OK**

**PowerShell Method:**
```powershell
# Add UPN suffix to forest
Set-ADForest -UPNSuffixes @{Add="porschecloud.com"}

# Verify UPN suffix
Get-ADForest | Select-Object -ExpandProperty UPNSuffixes
```

**Update User UPN Suffixes:**
```powershell
# Update Alice HR
Set-ADUser -Identity AliceHR -UserPrincipalName "AliceHR@porschecloud.com"

# Update Bob Finance
Set-ADUser -Identity BobFinance -UserPrincipalName "bobfinance@porschecloud.com"

# Update IAM Admin
Set-ADUser -Identity IAMAdmin -UserPrincipalName "IAMAdmin@porschecloud.com"

# Verify updates
Get-ADUser -Filter * -SearchBase "OU=Users,DC=porsche,DC=local" -Properties UserPrincipalName |
    Select-Object Name,UserPrincipalName
```

**Screenshot: User properties showing updated UPN suffix**

**In Entra ID Portal:**

1. Navigate to Microsoft Entra admin center
2. Go to **Settings** → **Custom domain names**
3. Click **+ Add custom domain**
4. Enter domain name: `porschecloud.com`
5. Click **Add domain**

**Domain Status:**
```
Domain: porschecloud.com
Status: Unverified
```

**Note:** For this lab, domain verification is not completed. In production:
1. Add TXT or MX record to DNS
2. Click "Verify" in portal
3. Wait for DNS propagation
4. Domain status changes to "Verified"

**Screenshot: Custom domain added to Entra ID (unverified)**

### Install Entra Connect Sync on AADCONNECT01

**Download Entra Connect:**

1. On AADCONNECT01, open browser
2. Navigate to: https://www.microsoft.com/en-us/download/details.aspx?id=47594
3. Download **Microsoft Entra Connect Sync**
4. Run installer: `AzureADConnect.msi`

**Launch Configuration Wizard:**
```
Welcome to Microsoft Entra Connect Sync

This tool will help you configure synchronization between
your on-premises directory and Microsoft Entra ID.
```

1. **Welcome:**
   - ✅ I agree to the license terms and privacy notice
   - Click **Continue**

2. **Express Settings:**
   - Click **Customize** (not Express Settings)
   - *We need to configure scoped sync and specific features*

### Configure Entra Connect Sync

**Install Required Components:**

1. **Install required components:**
   - ☐ Specify a custom installation location
   - ☐ Use an existing SQL Server
   - ☐ Use an existing service account
   - ☐ Specify custom sync groups
   - ☐ Import synchronization settings
   - Click **Install**

**Synchronization Services Installation:**
```
Installing synchronization engine...
Installing synchronization service...
Configuring service accounts...
Installation complete.
```

**Screenshot: Required components installation progress**

**User Sign-In:**

1. **Select Sign On method:**
   - ○ Password Hash Synchronization
   - ● **Pass-through authentication** ← Selected
   - ○ Federation with AD FS
   - ○ Federation with PingFederate
   - ○ Do not configure

2. **Pass-through authentication benefits:**
   - ✅ Real-time authentication against on-prem AD
   - ✅ No passwords stored in cloud
   - ✅ Reduced attack surface
   - ✅ Supports password complexity policies

3. **Enable single sign-on:**
   - ☐ Enable single sign-on
   - *Not configured for this lab*

**Why Pass-through Authentication:**

Original plan was Federation with AD FS, but:
- Custom domain `porschecloud.com` is unverified
- Federation requires verified domain
- PTA provides similar security without domain verification requirement

**Screenshot: User sign-in method selection showing PTA**

**Connect to Microsoft Entra ID:**

1. Enter Entra ID credentials:
```
   Username: admin@[tenant].onmicrosoft.com
   Password: [admin password]
```

2. Click **Next**

**Verification:**
```
Connecting to Microsoft Entra ID...
Validating credentials...
Connected successfully.
```

**Screenshot: Successfully connected to Entra ID tenant**

**Connect Directories:**

1. **Connect your directories:**
   - Directory Type: **Active Directory**
   - Click **Add Directory**

2. **AD forest account:**
   - ● Create new AD account
   - Enterprise Admin username: `PORSCHE\Administrator`
   - Password: [admin password]
   - Click **OK**

3. **Configured Directories:**
```
   ✓ porsche.local (Active Directory)
```

**Screenshot: Connected to on-prem AD forest**

**Microsoft Entra Sign-in Configuration:**

1. **UPN Suffixes for on-premises environment:**

   | Active Directory UPN Suffix | Microsoft Entra ID Domain | Status |
   |-----------------------------|---------------------------|---------|
   | porsche.local | Not Added | - |
   | porschecloud.com | porschecloud.com | ⚠ Not Verified |

2. **Select the on-premises attribute to use as Microsoft Entra ID username:**
   - ● userPrincipalName

3. **Continue without matching all UPN suffixes to verified domains:**
   - ✅ **Continue without matching all UPN suffixes to verified domains**

**Warning:**
```
Users will not be able to sign in to Microsoft Entra ID with on-premises 
credentials if the UPN suffix does not match a verified domain.
```

*For production: Domain must be verified for full functionality*

**Screenshot: UPN configuration showing unverified domain**

**Domain and OU Filtering:**

1. **Sync selected domains and OUs:**
   - ● **Sync selected domains and OUs**
   - ☐ Sync all domains and OUs

2. **Select OUs to synchronize:**
```
   ☑ porsche.local
       ☐ Builtin
       ☐ Computers
       ☐ Domain Controllers
       ☐ ForeignSecurityPrincipals
       ☑ Groups                    ← Selected
       ☐ Infrastructure
       ☐ LostAndFound
       ☐ Managed Service Accounts
       ☐ Program Data
       ☐ Servers
       ☐ Service Accounts
       ☐ System
       ☑ Users                     ← Selected
       ☐ Workstations
```

**Why Scoped Sync:**
- Only sync user accounts and groups needed in cloud
- Exclude infrastructure objects (computers, service accounts)
- Reduces sync complexity and improves performance
- Follows least-privilege principle

**Screenshot: OU filtering configuration**

**Identifying Users:**

1. **How should users be identified across directories:**
   - ● **Users are represented only once across all directories**
   
2. **How should users be identified with Microsoft Entra ID:**
   - ● **Let Azure manage source anchor** (recommended)
   - Source Anchor: `ms-DS-ConsistencyGuid`

**Matching Logic:**
- Users matched by ObjectGUID/ConsistencyGuid
- Ensures each on-prem user maps to exactly one cloud user
- Prevents duplicate accounts

**Screenshot: User identification configuration**

**Filtering:**

1. **Synchronize:**
   - ● **Synchronize all users and devices**
   - ○ Synchronize selected

*For production, use group-based filtering for phased rollout*

**Optional Features:**

1. **Select enhanced functionality:**
   - ☐ Exchange hybrid deployment
   - ☐ Exchange Mail Public Folders
   - ☐ Microsoft Entra app and attribute filtering
   - ☑ **Password hash synchronization** ← Backup authentication
   - ☑ **Password writeback** ← Enable SSPR
   - ☐ Group writeback
   - ☐ Device writeback
   - ☐ Directory extension attribute sync

**Key Features Explained:**

**Password Hash Synchronization (PHS):**
- Enabled as **backup** authentication method
- If PTA agents fail, authentication falls back to PHS
- Improves resilience and disaster recovery
- Microsoft best practice recommendation

**Password Writeback:**
- Enables Self-Service Password Reset (SSPR) in Entra ID
- Password changes in cloud written back to on-prem AD
- Users can reset password via https://passwordreset.microsoftonline.com
- Critical for hybrid user experience

**Screenshot: Optional features with PHS and writeback enabled**

### Configure Pass-through Authentication Agent

**Install PTA Agent:**

During Entra Connect installation, PTA agent automatically installed on AADCONNECT01.

**Verify PTA Agent:**
```powershell
# Check PTA service
Get-Service -Name "AzureADConnectAuthenticationAgent*"

# Expected:
# Name: AzureADConnectAuthenticationAgentUpdater
# Status: Running
# StartType: Automatic
```

**PTA Agent Registration:**
```powershell
# View PTA configuration
Get-Service PassthroughAuthPSModule

# Check PTA agent logs
Get-EventLog -LogName "Application" -Source "Azure AD Connect Authentication Agent" -Newest 10
```

**In Entra ID Portal:**

1. Navigate to **Microsoft Entra ID** → **Microsoft Entra Connect**
2. Go to **Connect Sync** → **Pass-through authentication**
3. Verify agent status:
```
   Server: AADCONNECT01.porsche.local
   Status: ✓ Active
   Version: [version]
```

**Screenshot: PTA agent showing as active in Entra admin center**

**How PTA Works:**
```
1. User enters credentials in cloud
2. Entra ID sends auth request to PTA agent (via Service Bus)
3. PTA agent decrypts request
4. Agent performs LDAP bind against DC01
5. DC01 validates credentials
6. Result returned to PTA agent
7. Agent sends encrypted response to Entra ID
8. Entra ID issues token if successful
```

### Configure Synchronization

**Ready to Configure:**

1. Review configuration summary:
```
   Synchronization:
   - Directory: porsche.local
   - OUs: Groups, Users
   - User Sign-In: Pass-through authentication
   - Optional Features:
     • Password hash synchronization
     • Password writeback
```

2. **Start the synchronization process when configuration completes:**
   - ✅ Checked

3. Click **Install**

**Configuration Progress:**
```
Configuring Microsoft Entra Connect Sync...

✓ Installing synchronization engine
✓ Configuring synchronization rules
✓ Installing Pass-through authentication agent
✓ Configuring password writeback
✓ Establishing connection to Microsoft Entra ID
✓ Importing Active Directory schema
✓ Starting initial synchronization

Configuration complete.
```

**Screenshot: Configuration complete screen**

### Verify Synchronization

**On AADCONNECT01:**
```powershell
# Import sync module
Import-Module ADSync

# Check sync scheduler
Get-ADSyncScheduler

# Expected output:
# AllowedSyncCycleInterval: 00:30:00
# CurrentlyEffectiveSyncCycleInterval: 00:30:00
# NextSyncCyclePolicyType: Delta
# NextSyncCycleStartTimeInUTC: [datetime]
# SyncCycleEnabled: True

# View sync connector status
Get-ADSyncConnectorRunStatus

# Check last sync cycle
Get-ADSyncCSObject -DistinguishedName "CN=AliceHR,OU=Users,DC=porsche,DC=local" `
    -ConnectorName "porsche.local" | Select-Object -ExpandProperty SynchronizationPolicyStatus
```

**Manual Sync (if needed):**
```powershell
# Force full sync
Start-ADSyncSyncCycle -PolicyType Initial

# Force delta sync
Start-ADSyncSyncCycle -PolicyType Delta
```

**In Entra ID Portal:**

1. Navigate to **Microsoft Entra ID** → **Users**
2. Add filter: **On-premises sync enabled == Yes**
3. Verify synced users appear:
```
   Display Name    | User Principal Name              | On-premises sync
   ----------------|----------------------------------|------------------
   Alice HR        | AliceHR@porschecloud.com        | Yes
   Bob Finance     | bobfinance@porschecloud.com     | Yes
   IAM Admin       | IAMAdmin@porschecloud.com       | Yes
```

**Screenshot: Entra ID Users showing synced accounts**

**Check User Properties:**

1. Click on **Alice HR**
2. View properties:
```
   Display name: Alice HR
   User principal name: AliceHR@porschecloud.com
   Object ID: [GUID]
   On-premises immutable ID: [Base64 encoded GUID]
   On-premises sync enabled: Yes
   Source: Windows Server AD
```

**Screenshot: User profile showing sync attributes**

### Test Hybrid Authentication

**Test Pass-through Authentication:**

1. On WIN11-CLIENT, open browser (InPrivate/Incognito)
2. Navigate to: https://portal.office.com
3. Enter credentials:
```
   Email: AliceHR@porschecloud.com
   Password: Welcome2024!
```

**Expected Flow:**
```
1. User enters credentials
2. Redirected to login.microsoftonline.com
3. Entra ID recognizes PTA user
4. Auth request sent to PTA agent on AADCONNECT01
5. PTA agent validates against DC01
6. Success - Token issued
7. User signs in to Microsoft 365 portal
```

**Verification in Entra ID:**

1. Navigate to **Microsoft Entra ID** → **Sign-in logs**
2. Filter for AliceHR
3. View sign-in details:
```
   User: AliceHR@porschecloud.com
   Application: Microsoft 365
   Status: Success
   Authentication method: Pass-through authentication
   Location: [Your location]
   IP address: [Your IP]
```

**Screenshot: Successful sign-in log showing PTA authentication**

**Test Password Writeback (SSPR):**

1. Open browser to: https://passwordreset.microsoftonline.com
2. Enter: `AliceHR@porschecloud.com`
3. Complete verification challenge
4. Set new password
5. Wait 1-2 minutes for writeback
6. On WIN11-CLIENT, sign out and sign in with new password
   - Should work immediately (password changed on-prem)

**Screenshot: SSPR portal showing successful password reset**

### Monitor Synchronization Health

**Entra Connect Health:**

1. **Microsoft Entra ID** → **Microsoft Entra Connect**
2. **Connect Sync** status:
```
   Sync Status: ✓ Enabled
   Last sync: [timestamp]
   Pass-through authentication: ✓ Healthy
   Password hash sync: ✓ Healthy
   Password writeback: ✓ Healthy
```

**Synchronization Errors:**
```powershell
# On AADCONNECT01
# Check for sync errors
Get-ADSyncScheduler | Select-Object LastSyncResult
Get-ADSyncCSObject | Where-Object {$_.ConnectorName -eq "porsche.local"} | 
    Where-Object {$_.ImportError -ne $null}
```

**Screenshot: Connect Sync health dashboard showing green status**

---

## 🎯 Final Architecture Validation

### End-to-End Testing Checklist

**✅ On-Premises Infrastructure:**
- [x] DC01 running AD DS, DNS, AD CS
- [x] ADFS01 running AD FS with valid certificate
- [x] AADCONNECT01 running Entra Connect Sync + PTA agent
- [x] WIN11-CLIENT domain-joined and accessible
- [x] All servers on same network (10.10.10.0/24)
- [x] DNS resolution working for all resources

**✅ Active Directory:**
- [x] Domain: porsche.local created
- [x] OUs: Groups, Users, Servers, Workstations, Service Accounts
- [x] Security groups created with proper scope
- [x] Test users created with correct UPN suffix
- [x] Computer objects in correct OUs
- [x] Delegation configured for domain join account

**✅ Certificate Services:**
- [x] Enterprise Root CA operational
- [x] Root CA certificate deployed to all domain computers
- [x] Web Server template published
- [x] SSL certificate issued for fs.porsche.local
- [x] Certificate installed on ADFS01
- [x] Certificate chain validates

**✅ Federation Services:**
- [x] AD FS installed and configured
- [x] gMSA created for AD FS service
- [x] Federation service name: fs.porsche.local
- [x] SSL certificate bound to HTTPS (443)
- [x] IdP-initiated sign-on enabled and working
- [x] Federation metadata accessible

**✅ Hybrid Identity:**
- [x] Custom domain added to Entra ID
- [x] UPN suffixes aligned (porschecloud.com)
- [x] Entra Connect Sync installed
- [x] Pass-through authentication configured
- [x] PTA agent healthy and responding
- [x] Password hash sync enabled (backup)
- [x] Password writeback enabled
- [x] Scoped sync configured (Users, Groups OUs only)
- [x] Initial synchronization completed
- [x] Users visible in Entra ID with "On-premises sync enabled"

**✅ Authentication Testing:**
- [x] Domain users can sign in to domain-joined computers
- [x] Cloud authentication works via PTA
- [x] Sign-in logs show successful PTA authentication
- [x] Password changes replicate bidirectionally
- [x] SSPR works and writes back to on-prem AD

### Architecture Diagram Legend
```
┌─────────────┐
│   Cloud     │  = Microsoft Entra ID (Azure AD)
└─────────────┘

┌─────────────┐
│  On-Prem    │  = On-premises virtual machine
│   Server    │
└─────────────┘

      ↕        = Bidirectional communication
      ↓        = Unidirectional flow
      
[✓]           = Validation checkpoint passed
[●]           = Configuration required
```

---

## 🔍 Troubleshooting Guide

### Common Issues and Resolutions

#### Issue 1: Certificate Request Fails Silently

**Symptoms:**
- Certificate enrollment wizard completes but no certificate appears
- Personal certificate store remains empty
- No errors displayed

**Diagnosis:**
```powershell
# Check CA logs
Get-EventLog -LogName Application -Source "Microsoft-Windows-CertificationAuthority" -Newest 20

# Check for denied requests
certutil -view -restrict "Disposition=30" -out "RequestID,RequesterName,CommonName"
```

**Root Cause:**
Enterprise CA requires explicit certificate template in request.

**Solution:**
Use `certreq` with INF file specifying template:
```powershell
certreq -new request.inf request.req
certreq -submit request.req
certreq -accept certnew.cer
```

---

#### Issue 2: IdP Sign-On Page Returns 404

**Symptoms:**
- Cannot access https://fs.porsche.local/adfs/ls/idpinitiatedsignon.aspx
- Error: "Resource you are trying to access is not available"

**Diagnosis:**
```powershell
# Check if sign-on page is enabled
Get-AdfsProperties | Select-Object EnableIdPInitiatedSignonPage
# Returns: False
```

**Root Cause:**
IdP-initiated sign-on disabled by default for security.

**Solution:**
```powershell
Set-AdfsProperties -EnableIdPInitiatedSignonPage $true
Restart-Service adfssrv
```

---

#### Issue 3: UPN Suffix Doesn't Match Entra ID Domain

**Symptoms:**
- Users cannot authenticate to Entra ID
- Warning during Entra Connect configuration
- Sign-in fails with "user not found"

**Diagnosis:**
```powershell
# Check user UPN
Get-ADUser AliceHR -Properties UserPrincipalName | Select-Object UserPrincipalName
# Shows: AliceHR@porsche.local

# Check available UPN suffixes
Get-ADForest | Select-Object -ExpandProperty UPNSuffixes
# Missing: porschecloud.com
```

**Root Cause:**
- .local domain not routable
- UPN must match verified domain in Entra ID

**Solution:**
```powershell
# Add UPN suffix to forest
Set-ADForest -UPNSuffixes @{Add="porschecloud.com"}

# Update user UPNs
Set-ADUser -Identity AliceHR -UserPrincipalName "AliceHR@porschecloud.com"

# Verify in Entra ID portal
# Add custom domain: porschecloud.com
```

---

#### Issue 4: Synchronization Not Starting

**Symptoms:**
- Users not appearing in Entra ID after 30+ minutes
- Sync scheduler shows cycle pending but never runs

**Diagnosis:**
```powershell
# Check scheduler status
Get-ADSyncScheduler

# Check connector status
Get-ADSyncConnectorRunStatus

# Review sync errors
Get-ADSyncCSObject | Where-Object {$_.ImportError -ne $null}
```

**Common Causes & Solutions:**

**Cause 1: Scheduler disabled**
```powershell
Set-ADSyncScheduler -SyncCycleEnabled $true
```

**Cause 2: Connector authentication failure**
```powershell
# Re-enter AD credentials
Get-ADSyncConnector -Name "porsche.local" | 
    Set-ADSyncConnector -ConnectivityParameters @{
        forestfqdn = "porsche.local"
        username = "PORSCHE\Administrator"
        password = Read-Host -AsSecureString
    }
```

**Cause 3: Filtering excludes all objects**
```powershell
# Verify OU filtering
Get-ADSyncConnector -Name "porsche.local" | 
    Select-Object -ExpandProperty Partitions |
    Select-Object -ExpandProperty ConnectorPartitionScope

# Adjust if needed via Entra Connect wizard
```

---

#### Issue 5: Pass-through Authentication Not Working

**Symptoms:**
- Cloud sign-in fails
- "Incorrect username or password" error
- Sign-in logs show authentication method failures

**Diagnosis:**
```powershell
# Check PTA agent service
Get-Service -Name "AzureADConnect*Auth*"

# Check PTA connector status in Entra ID portal:
# Entra ID → Microsoft Entra Connect → Pass-through authentication
```

**Common Causes & Solutions:**

**Cause 1: PTA agent not running**
```powershell
Start-Service -Name "AzureADConnectAuthenticationAgent"
Set-Service -Name "AzureADConnectAuthenticationAgent" -StartupType Automatic
```

**Cause 2: Firewall blocking outbound HTTPS**
```powershell
# Verify outbound 443 to Azure
Test-NetConnection -ComputerName "login.microsoftonline.com" -Port 443
```

**Cause 3: Agent not registered**
```powershell
# Re-register agent (from Entra Connect wizard)
# Or download standalone agent installer from portal
```

---

#### Issue 6: Password Writeback Not Working

**Symptoms:**
- Password reset in Entra ID succeeds but doesn't update AD
- SSPR works but user can't sign in with new password on-prem

**Diagnosis:**
```powershell
# Check writeback status
Get-ADSyncAADPasswordResetConfiguration

# Check for permission errors in event log
Get-EventLog -LogName Application -Source "ADSync" -Newest 50 | 
    Where-Object {$_.Message -like "*password*"}
```

**Common Causes & Solutions:**

**Cause 1: Writeback not enabled**
```powershell
# Enable via Entra Connect wizard
# Or PowerShell:
Set-ADSyncAADPasswordResetConfiguration -Enable $true
```

**Cause 2: Insufficient permissions**
```powershell
# Sync account needs "Reset password" permission on Users OU
# Grant via ADSI Edit or delegation wizard
```

---

### Performance Optimization

**Sync Cycle Tuning:**
```powershell
# Adjust sync interval (default 30 min)
Set-ADSyncScheduler -CustomizedSyncCycleInterval 00:30:00

# Force immediate sync after changes
Start-ADSyncSyncCycle -PolicyType Delta
```

**Filtering Best Practices:**

- Use OU-based filtering instead of attribute-based when possible
- Sync only required attributes (reduces payload size)
- Exclude large distribution groups not needed in cloud

---

### Logging and Monitoring

**Key Log Locations:**

**On DC01:**
```
Event Viewer → Applications and Services Logs
  → Directory Service (AD DS events)
  → DNS Server (DNS resolution)
  → Microsoft → Windows → CertificationAuthority (Certificate events)
```

**On ADFS01:**
```
Event Viewer → Applications and Services Logs
  → AD FS → Admin (Federation events)
C:\Windows\ADFS\Logs (Detailed trace logs if enabled)
```

**On AADCONNECT01:**
```
Event Viewer → Application and Services Logs
  → ADSync (Sync engine events)
C:\ProgramData\AADConnect\trace-{timestamp}.log
```

**Entra ID Portal:**
```
Microsoft Entra ID → Sign-in logs (Authentication events)
Microsoft Entra ID → Audit logs (Configuration changes)
Microsoft Entra Connect → Sync errors (Sync issues)
```

---

## 📚 Additional Resources

### Microsoft Documentation

- [Microsoft Entra Connect Sync](https://learn.microsoft.com/entra/identity/hybrid/connect/whatis-azure-ad-connect)
- [Pass-through Authentication](https://learn.microsoft.com/entra/identity/hybrid/connect/how-to-connect-pta)
- [AD FS Deployment Guide](https://learn.microsoft.com/windows-server/identity/ad-fs/deployment/windows-server-2012-r2-ad-fs-deployment-guide)
- [AD CS Best Practices](https://learn.microsoft.com/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/dn486880(v=ws.11))

### Concepts Explained

**Hybrid Identity:**
- Synchronizing on-prem identities to cloud
- Enables single identity across environments
- Supports various authentication methods

**Pass-through Authentication (PTA):**
- Authentication agent validates credentials against on-prem AD
- No passwords stored in cloud
- Real-time validation
- Requires agent connectivity to internet

**Password Hash Synchronization (PHS):**
- Syncs hash of password hash to cloud
- Enables cloud authentication without PTA agents
- Used as backup if PTA unavailable
- Enables leak detection features

**Federation (AD FS):**
- Claims-based authentication
- Full control over authentication process
- Supports advanced MFA scenarios
- Requires more infrastructure

**Group Managed Service Accounts (gMSA):**
- Automatic password management
- Service account with no manual password required
- Supported by AD FS and other Windows services
- Enhances security by rotating passwords automatically

---

## 🎓 Skills Demonstrated

This lab demonstrates proficiency in:

**Identity & Access Management:**
- Hybrid identity architecture design
- Multi-factor authentication concepts
- Single sign-on (SSO) implementation
- Identity lifecycle management
- Group-based access control (RBAC)

**Active Directory:**
- Domain controller deployment
- OU structure design
- Security group strategy
- Service account management
- Group Policy fundamentals

**Microsoft Entra ID (Azure AD):**
- Tenant configuration
- Custom domain management
- User synchronization
- Authentication methods
- Conditional Access readiness

**Certificate Services (PKI):**
- Enterprise CA deployment
- Certificate template configuration
- SSL/TLS certificate management
- Trust chain validation

**Federation Services:**
- AD FS deployment and configuration
- Claims-based authentication concepts
- Federation trust establishment
- IdP-initiated sign-on

**Directory Synchronization:**
- Entra Connect Sync deployment
- Scoped synchronization with OU filtering
- Password writeback configuration
- Sync troubleshooting

**Authentication Technologies:**
- Pass-through Authentication (PTA)
- Password Hash Synchronization (PHS)
- Federation protocols (WS-Fed, SAML)
- Authentication flow analysis

**Infrastructure & Virtualization:**
- VMware Workstation configuration
- Virtual networking design
- Static IP addressing schemes
- DNS infrastructure

**PowerShell Administration:**
- Active Directory module
- AD FS cmdlets
- Entra Connect sync automation
- Certificate request automation

**Troubleshooting & Operations:**
- Log analysis and diagnostics
- Event viewer interpretation
- Network connectivity testing
- Service health monitoring

---

## 🔐 Security Considerations

### Implemented Security Controls

**✅ Least Privilege:**
- Service accounts have minimal permissions required
- OU-based delegation limits administrative scope
- gMSA for automatic credential rotation

**✅ Defense in Depth:**
- Multiple authentication methods (PTA + PHS backup)
- Certificate-based trust (PKI)
- Isolated lab network (no external access)

**✅ Password Security:**
- Password writeback enables SSPR
- Password policies enforced on-prem and synced
- No plaintext passwords in cloud (PTA)

**✅ Credential Protection:**
- gMSA for AD FS (no stored passwords)
- Encrypted communication channels
- Service Bus relay for PTA (no inbound firewall rules)

### Production Hardening Recommendations

**For Production Deployment:**

1. **High Availability:**
   - Deploy multiple AD FS servers behind load balancer
   - Install additional PTA agents (3+ recommended)
   - Use SQL Server instead of WID for AD FS farm
   - Deploy multiple domain controllers

2. **Network Security:**
   - Place AD FS in DMZ with Web Application Proxy (WAP)
   - Implement firewall rules limiting network access
   - Use separate VLANs for each tier
   - Enable network segmentation

3. **Certificate Management:**
   - Use public CA or subordinate CA for AD FS certificate
   - Implement certificate lifecycle management
   - Set up automated renewal alerts
   - Consider using wildcard or SAN certificates

4. **Monitoring & Alerting:**
   - Enable Microsoft Entra Connect Health
   - Configure alerts for sync failures
   - Monitor PTA agent health
   - Set up SIEM integration for security logs

5. **Disaster Recovery:**
   - Document recovery procedures
   - Maintain offline root CA backup
   - Regular AD backup and tested restore
   - Export AD FS configuration regularly

6. **Authentication Security:**
   - Enable Conditional Access policies
   - Implement MFA for all admin accounts
   - Configure risk-based sign-in policies
   - Enable Identity Protection

7. **Compliance:**
   - Verify domain ownership (complete domain verification)
   - Enable audit logging for all identity operations
   - Implement privileged access management
   - Regular access reviews and attestation

---

## 🚀 Next Steps & Enhancements

### Phase 7 (Future): Advanced Identity Features

**Planned Enhancements:**

1. **Multi-Factor Authentication (MFA):**
   - Configure MFA for Entra ID users
   - Integrate with on-prem AD FS for MFA
   - Deploy NPS extension for RADIUS MFA

2. **Conditional Access:**
   - Create location-based policies
   - Device compliance requirements
   - App-specific access controls
   - Risk-based access policies

3. **Self-Service Password Reset (SSPR):**
   - Configure SSPR registration portal
   - Test password writeback flows
   - Deploy registration campaign for users

4. **Privileged Identity Management (PIM):**
   - Enable just-in-time admin access
   - Configure approval workflows
   - Implement access reviews

5. **Application Integration:**
   - Configure enterprise applications
   - Set up SAML-based SSO
   - Deploy OAuth/OIDC applications
   - Configure application proxy

6. **Device Management:**
   - Hybrid Azure AD Join for Windows 11
   - Intune enrollment
   - Compliance policies
   - Conditional Access based on device state

7. **Reporting & Analytics:**
   - Sign-in logs analysis
   - Usage reports
   - Security alerts dashboard
   - Custom PowerBI reports

8. **Automation:**
   - Automated user provisioning
   - Lifecycle workflows
   - PowerShell automation scripts
   - Microsoft Graph API integration

---

## 📋 Appendix

### PowerShell Script Collection

**Quick Sync Trigger:**
```powershell
# Force immediate sync cycle
Start-ADSyncSyncCycle -PolicyType Delta
```

**Sync Status Check:**
```powershell
# Comprehensive sync status
Get-ADSyncScheduler | Format-List
Get-ADSyncConnectorRunStatus | Format-Table
```

**User Sync Verification:**
```powershell
# Check if specific user is synced
$user = "AliceHR"
Get-ADSyncCSObject -DistinguishedName "CN=$user,OU=Users,DC=porsche,DC=local" `
    -ConnectorName "porsche.local"
```

**PTA Agent Health:**
```powershell
# Check PTA agent status
Get-Service -Name "AzureADConnect*Auth*" | Format-Table Name,Status,StartType
```

**Certificate Validation:**
```powershell
# Check certificate expiration
Get-ChildItem Cert:\LocalMachine\My | 
    Where-Object {$_.Subject -like "*fs.porsche.local*"} |
    Select-Object Subject,NotAfter,Thumbprint
```

### Configuration Files

**Sample Certificate Request INF:**

Save as `C:\Temp\request.inf`:
```ini
[Version]
Signature="$Windows NT$"

[NewRequest]
Subject = "CN=fs.porsche.local"
KeySpec = 1
KeyLength = 2048
Exportable = TRUE
MachineKeySet = TRUE
SMIME = FALSE
PrivateKeyArchive = FALSE
UserProtected = FALSE
UseExistingKeySet = FALSE
ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
ProviderType = 12
RequestType = PKCS10
KeyUsage = 0xa0
HashAlgorithm = SHA256

[EnhancedKeyUsageExtension]
OID=1.3.6.1.5.5.7.3.1

[Extensions]
2.5.29.17 = "{text}"
_continue_ = "dns=fs.porsche.local&"

[RequestAttributes]
CertificateTemplate = "WebServer"
```

### Network Configuration Reference

**Complete Network Configuration:**

| Server | Hostname | IP Address | Subnet Mask | DNS | Role |
|--------|----------|-----------|-------------|-----|------|
| DC01 | DC01.porsche.local | 10.10.10.10 | 255.255.255.0 | 127.0.0.1 | Domain Controller, DNS, CA |
| ADFS01 | ADFS01.porsche.local | 10.10.10.20 | 255.255.255.0 | 10.10.10.10 | Federation Server |
| AADCONNECT01 | AADCONNECT01.porsche.local | 10.10.10.30 | 255.255.255.0 | 10.10.10.10 | Sync Server, PTA Agent |
| WIN11-CLIENT | WIN11-CLIENT.porsche.local | 10.10.10.100 | 255.255.255.0 | 10.10.10.10 | Test Workstation |

---

## ✅ Project Completion Checklist

**Infrastructure:**
- [x] VMware network configured (VMnet10)
- [x] 4 virtual machines created
- [x] Static IP addressing implemented
- [x] DNS resolution verified

**Active Directory:**
- [x] Domain forest deployed (porsche.local)
- [x] OU structure created
- [x] Security groups configured
- [x] Test users created with UPN alignment
- [x] Computer objects organized

**Certificates:**
- [x] Enterprise CA deployed on DC01
- [x] Certificate template published
- [x] SSL certificate issued for AD FS
- [x] Root CA trusted by all domain members

**Federation:**
- [x] AD FS installed on ADFS01
- [x] gMSA created for service account
- [x] Federation service configured
- [x] IdP-initiated sign-on enabled

**Hybrid Identity:**
- [x] Custom domain added to Entra ID
- [x] Entra Connect Sync installed
- [x] Pass-through authentication configured
- [x] Password writeback enabled
- [x] Scoped synchronization configured
- [x] Initial sync completed successfully

**Testing:**
- [x] Domain logon tested
- [x] Cloud authentication tested (PTA)
- [x] Password writeback tested (SSPR)
- [x] Sign-in logs verified
- [x] Sync health confirmed

**Documentation:**
- [x] Architecture diagrams created
- [x] Configuration documented
- [x] Troubleshooting guide written
- [x] PowerShell scripts collected
- [x] Security considerations noted

---

*This technical implementation guide provides comprehensive details for replicating the hybrid identity lab environment. All configurations follow Microsoft best practices and industry standards for enterprise identity management.*

**Lab Environment:** Isolated virtual infrastructure  
**Completion Date:** [Your completion date]  
**Total Implementation Time:** ~8-12 hours

---

**END OF TECHNICAL DETAILS**
