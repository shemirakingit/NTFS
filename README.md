# NTFS
Provisions a 3-VM Active Directory + NTFS file server lab on Azure using Terraform and PowerShell — fully automated, Infrastructure as Code


A fully automated Azure lab that deploys a Windows Active Directory environment 
with role-based NTFS file permissions using Terraform and PowerShell.

### What this deploys
- DC01 — Windows Server 2022 Domain Controller (Active Directory, DNS, Group Policy)
- FS01 — File Server with 4 SMB shares (Finance, HR, Sales, IT)
- CLIENT01 — Windows 11 workstation for end-to-end permission testing

### What it covers
- Active Directory (OUs, security groups, test users)
- NTFS permissions enforced per security group
- SMB file share configuration
- Azure Key Vault for secret management
- Remote State with Azure Blob Storage
- Automated verification scripts (PASS/FAIL output)

### Stack
Terraform · PowerShell · Azure (EC2, VNet, NSG, Key Vault)

### Part of a series
This is Lab 1 of 3. Lab 2 (Azure RBAC) builds directly on this infrastructure.

### Deploy time
~25–35 minutes total | ~$0.15–$0.25/hr while running
