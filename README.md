# Hybrid Identity Lab: Active Directory + Microsoft Entra ID Integration

![Active Directory](https://img.shields.io/badge/Active_Directory-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![Microsoft Entra ID](https://img.shields.io/badge/Microsoft_Entra_ID-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![AD FS](https://img.shields.io/badge/AD_FS-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![VMware](https://img.shields.io/badge/VMware-607078?style=for-the-badge&logo=vmware&logoColor=white)
![PowerShell](https://img.shields.io/badge/PowerShell-5391FE?style=for-the-badge&logo=powershell&logoColor=white)

## 📋 Project Overview

Built a complete hybrid identity environment integrating on-premises Active Directory with Microsoft Entra ID (formerly Azure AD). This lab demonstrates enterprise-grade identity and access management (IAM) capabilities including directory synchronization, federated authentication, certificate services, and hybrid authentication methods.

### 🎯 Business Objectives
- Enable secure single sign-on (SSO) between on-premises and cloud resources
- Implement hybrid identity for seamless user experience across environments
- Establish certificate-based trust for federated authentication
- Demonstrate real-world IAM architecture and troubleshooting skills

### 🏗️ Architecture Components
- **Domain Controller (DC01)**: Windows Server 2016 running AD DS, DNS, and AD CS
- **Federation Server (ADFS01)**: AD FS for federated authentication
- **Sync Server (AADCONNECT01)**: Entra Connect Sync with Pass-through Authentication
- **Client Workstation (WIN11-CLIENT)**: Domain-joined Windows 11 for testing
- **Cloud Identity**: Microsoft Entra ID tenant with hybrid sync

---

## 🚀 Implementation Phases

### Phase 1: Infrastructure Foundation

**Network Configuration**
- Created isolated VMnet10 host-only network (10.10.10.0/24)
- Implemented static IP addressing scheme for all servers
- Configured DNS resolution pointing to domain controller

**IP Address Plan:**
| Server | IP Address | Role |
|--------|-----------|------|
| DC01 | 10.10.10.10 | Domain Controller, DNS, CA |
| ADFS01 | 10.10.10.20 | AD FS Federation Server |
| AADCONNECT01 | 10.10.10.30 | Entra Connect Sync Server |
| WIN11-CLIENT | 10.10.10.100 | Test Client (DHCP/Static) |

**Virtual Machine Specifications**
- DC01: 2 vCPU, 4-6 GB RAM, 60 GB disk
- ADFS01: 2 vCPU, 4-6 GB RAM, 60 GB disk
- AADCONNECT01: 2 vCPU, 6-8 GB RAM, 80 GB disk
- WIN11-CLIENT: 2 vCPU, 6-8 GB RAM, 80 GB disk

---

### Phase 2: On-Premises Active Directory

**Domain Controller Deployment**
- Installed Windows Server 2016 Standard with Desktop Experience
- Deployed AD DS and DNS roles on DC01
- Created new forest: `porsche.local`
- Configured static IP (10.10.10.10) with DNS pointing to itself

**Organizational Structure**
Created logical OU hierarchy for scoped synchronization:
- `Groups` - Security groups for RBAC
- `Users` - User accounts with group assignments
- `Servers` - Infrastructure servers
- `Workstations` - Client computers
- `Service Accounts` - Managed service accounts

**Security Groups (Global)**
- `GG-HR` - HR department access
- `GG-Finance` - Finance department access
- `GG-IAM-Admins` - Identity administrators
- `GG-SSPR-Pilot` - Self-service password reset pilot
- `GG-App-OIDC-Users` - OIDC application access
- `GG-App-SaaS-SAML-Users` - SAML federation access

**Test User Accounts**
- Alice HR (GG-HR, GG-SSPR-Pilot)
- Bob Finance (GG-Finance)
- IAM Admin (GG-IAM-Admins)

**Domain Join Operations**
- Joined ADFS01, AADCONNECT01, and WIN11-CLIENT to domain
- Created delegated service account (svc_domainjoin) with computer object permissions
- Moved computer objects to appropriate OUs for policy application

---

### Phase 3: Certificate Infrastructure (PKI)

**Enterprise Root CA Deployment**
- Installed AD Certificate Services on DC01
- Configured Enterprise Root CA for domain-integrated PKI
- Published Web Server certificate template for AD FS SSL

**Certificate Issuance Process**
1. Created DNS A record: `fs.porsche.local` → 10.10.10.20
2. Configured Web Server template with enrollment permissions for ADFS01
3. Issued SSL certificate for federation service

**Troubleshooting: Certificate Request Failure**

*Problem:* Certificate enrollment wizard failed silently when requesting Web Server certificate through MMC on ADFS01.

*Root Cause:* Enterprise CA requires explicit certificate template reference. MMC wizard did not include template information in the request, causing CA to reject issuance.

*Solution:* 
- Created manual certificate request using `certreq` with template specification
- Generated request.inf file explicitly referencing Web Server template
- Successfully issued certificate with proper Subject and SAN attributes
```powershell
# Example certreq workflow
certreq -new request.inf request.req
certreq -submit request.req
certreq -accept certnew.cer
```

**Certificate Validation**
- Verified certificate chain trust on domain-joined systems
- Confirmed Root CA certificate in Trusted Root store
- Validated SSL certificate properties (Subject: fs.porsche.local)

---

### Phase 4: Active Directory Federation Services (AD FS)

**AD FS Role Installation**
- Installed AD FS role on ADFS01
- Configured federation service properties:
  - Federation Service Name: `fs.porsche.local`
  - Display Name: `Porsche`
  - SSL Certificate: Issued by porsche-DC01-CA

**Service Account Configuration**
- Created Group Managed Service Account (gMSA): `gmsa_adfs$`
- Granted gMSA permissions to AD FS service

**Federation Database**
- Deployed Windows Internal Database (WID) for AD FS farm
- Single-server farm configuration for lab environment

**IdP-Initiated Sign-On**

*Initial Issue:* IdP sign-on page returned "resource not available" error when accessing `https://fs.porsche.local/adfs/ls/idpinitiatedsignon.aspx`

*Resolution:* Enabled IdP-initiated sign-on via PowerShell (disabled by default for security):
```powershell
Set-AdfsProperties -EnableIdPInitiatedSignonPage $true
Restart-Service adfssrv
```

**Validation**
- Confirmed AD FS services running (adfssrv)
- Verified SSL certificate binding on port 443
- Tested IdP-initiated sign-on page accessibility

---

### Phase 5: Hybrid Identity with Entra Connect Sync

**Entra Connect Sync Deployment**
- Installed Microsoft Entra Connect Sync on AADCONNECT01
- Configured custom deployment (not Express Settings)

**Authentication Method**
- Selected: **Pass-through Authentication (PTA)**
- Enabled: **Password Hash Synchronization** (backup method)
- Enabled: **Password Writeback** (for SSPR)

> **Note:** Initially planned Federation with AD FS, but pivoted to PTA due to unverified custom domain. In production, AD FS would enable federated SSO with on-prem authentication.

**UPN Alignment**

*Challenge:* On-prem users had `@porsche.local` UPN suffix, which doesn't match routable domain.

*Solution:*
1. Added alternative UPN suffix in AD Domains and Trusts: `porschecloud.com`
2. Updated user UPN suffix to `@porschecloud.com`
3. Added custom domain to Entra ID tenant
4. Configured UPN matching for hybrid authentication

**Scoped Synchronization**
- Enabled OU filtering to sync only:
  - `Users` OU
  - `Groups` OU
- Excluded infrastructure OUs (Servers, Workstations, Service Accounts)

**Optional Features Enabled**
- ✅ Password Hash Synchronization (backup authentication)
- ✅ Password Writeback (SSPR integration)
- ✅ Group Writeback (for Microsoft 365 Groups)

**Synchronization Validation**
- Verified initial sync cycle completed successfully
- Confirmed synced users appear in Entra ID portal
- Validated `On-premises sync enabled == Yes` attribute
- Tested cloud sign-in using on-prem credentials via PTA

---

## 🔧 Key Technical Challenges & Solutions

### 1. Certificate Template Enrollment Failure
**Problem:** Web Server certificate requests failed silently through MMC
**Solution:** Used `certreq` with explicit template reference to successfully issue certificate

### 2. IdP Sign-On Page Unavailable
**Problem:** AD FS sign-on page returned resource error
**Solution:** Enabled IdP-initiated sign-on via PowerShell (security-hardened by default)

### 3. UPN Suffix Mismatch
**Problem:** `.local` domain suffix incompatible with cloud authentication
**Solution:** Added routable UPN suffix and updated user accounts

### 4. Unverified Custom Domain
**Problem:** Cannot configure AD FS federation without verified domain
**Solution:** Pivoted to Pass-through Authentication as hybrid authentication method

---

## 💡 Key Learnings

### Identity & Access Management
- **Hybrid Authentication Methods**: Understanding tradeoffs between Federation (AD FS), Pass-through Authentication (PTA), and Password Hash Sync (PHS)
- **Certificate Trust Models**: Enterprise CA integration with Active Directory for automated certificate enrollment
- **Scoped Synchronization**: Using OU filtering to control which objects sync to cloud
- **UPN Alignment**: Importance of routable UPN suffixes for hybrid identity

### Infrastructure & Architecture
- **Service Account Best Practices**: Using Group Managed Service Accounts (gMSA) for AD FS
- **Network Isolation**: Host-only networking for secure lab environments
- **DNS Dependencies**: Critical role of DNS in AD, federation, and hybrid identity

### Troubleshooting & Operations
- **Certificate Request Workflows**: Manual certificate enrollment using `certreq` tool
- **PowerShell Administration**: Configuring AD FS properties and services
- **Synchronization Health Monitoring**: Validating sync cycles and authentication flows

---

## 🛠️ Skills Demonstrated

- Active Directory Domain Services (AD DS)
- Microsoft Entra ID (Azure AD)
- Active Directory Federation Services (AD FS)
- Active Directory Certificate Services (AD CS / PKI)
- Entra Connect Sync (Azure AD Connect)
- Pass-through Authentication (PTA)
- Identity Lifecycle Management
- Hybrid Authentication Architecture
- VMware Workstation / Virtual Infrastructure
- PowerShell Scripting & Administration
- DNS Configuration & Troubleshooting
- SSL/TLS Certificate Management
- Windows Server Administration
- Identity and Access Management (IAM)

---

## 📚 Technologies Used

**On-Premises:**
- Windows Server 2016 Standard
- Active Directory Domain Services
- Active Directory Federation Services
- Active Directory Certificate Services
- DNS Server
- Group Policy

**Cloud:**
- Microsoft Entra ID (Azure AD)
- Entra Connect Sync
- Pass-through Authentication Agents

**Infrastructure:**
- VMware Workstation
- Windows 11 (Client)
- PowerShell 5.1

---

## 🔮 Future Enhancements

- [ ] Implement Conditional Access policies in Entra ID
- [ ] Configure Multi-Factor Authentication (MFA) for hybrid users
- [ ] Deploy Self-Service Password Reset (SSPR) with password writeback
- [ ] Add Web Application Proxy for external AD FS access
- [ ] Implement AD FS claims rules for application authorization
- [ ] Configure device registration for Hybrid Azure AD Join
- [ ] Deploy Privileged Identity Management (PIM) for admin accounts

---

## 📄 License

This project is a personal learning lab for portfolio demonstration purposes.

---

## 📧 Contact

**[Your Name]**  
[Your Email] | [LinkedIn Profile] | [GitHub Profile]

---

*This lab environment demonstrates enterprise IAM capabilities in a controlled, isolated setting. All configurations follow Microsoft best practices for hybrid identity deployments.*
