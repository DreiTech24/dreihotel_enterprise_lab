# DreiHotel IT Lab: Active Directory, Entra ID & Wazuh SIEM

▶️ **[Watch the Full Project Demonstration on YouTube](https://youtu.be/708ixprGRDA)**

## 🎯 Project Objective
The goal of this homelab is to build, secure, and monitor a hybrid IT network from scratch. This project demonstrates Active Directory domain setup, automated user provisioning with PowerShell, cloud identity synchronization via Microsoft Entra Connect (with MFA), and proactive threat detection using Wazuh SIEM against a simulated Kali Linux brute-force attack.

## 🧰 Technologies & Tools Used
* **Infrastructure:** Oracle VirtualBox, pfSense Firewall
* **Operating Systems:** Windows Server 2022, Windows 11, Kali Linux
* **Identity & Access Management:** Active Directory (AD DS), Microsoft Entra ID, Entra Connect
* **Security & Monitoring:** Wazuh SIEM, CrackMapExec (Red Team tool)
* **Scripting:** PowerShell

---

## 📝 Project Phases & Execution

### Phase 1: Local Infrastructure & Active Directory Setup
* Configured a Windows Server 2022 Domain Controller with a static IP and deployed the `dreihotel.local` domain.
* Automated the creation of departments (OUs) and employee accounts using a custom **PowerShell script**.
* Enforced a strict `-ChangePasswordAtLogon` policy to secure first-time user logins.
* Created an HR Shared Folder and secured it using **Role-Based Access Control (RBAC)** to restrict unauthorized access.

### Phase 2: Employee Workstation & Security Testing
* Connected a Windows 11 client machine to the local domain network.
* Verified the PowerShell security policy by forcing a new employee to change their default password upon their very first login.
* Automated the mapping of the shared network drive to all employees using **Group Policy (GPO)**.
* Tested the RBAC security by proving unauthorized users received an "Access Denied" error when trying to open the restricted HR folder.

### Phase 3: Cloud Identity & Multi-Factor Authentication (MFA)
* Installed and configured **Microsoft Entra Connect** on the Domain Controller.
* Successfully synchronized the local Active Directory users into the Microsoft Entra ID cloud environment.
* Enforced Microsoft "Security Defaults" to require **Multi-Factor Authentication (MFA)** via the Microsoft Authenticator app for all cloud logins.

### Phase 4: Blue Team Defense & SIEM Deployment
* Set up a **Wazuh SIEM** server to act as the central security monitoring dashboard.
* Silently deployed Wazuh security agents to the Windows Server and Windows 11 machines using PowerShell.
* Verified that the agents were actively running and successfully forwarding security event logs to the dashboard.

### Phase 5: Red Team Attack & Threat Hunting
* Switched to a **Kali Linux** attacker machine to perform network reconnaissance using `nmap`, successfully discovering an open SMB port (445).
* Launched a Brute-Force attack against the main Administrator account using the `crackmapexec` tool.
* Switched back to the Blue Team side and used the Wazuh "Threat Hunting" module to detect the attack in real-time.
* Successfully analyzed the security logs to identify the attacker's source IP address and the exact **Windows Event ID 4625 (Logon Failure)**.

---

## 🏗️ Network Diagram

<img width="6462" height="3324" alt="ADLAB drawio" src="https://github.com/user-attachments/assets/98a80418-9fd3-4f11-a94f-9e6cdeaa366e" />




## 💻 Active Directory Automation Script (PowerShell)

To efficiently provision employee accounts, I wrote and executed the following PowerShell script. It automatically creates Organizational Units (OUs), generates standard usernames, assigns default passwords, and enforces a strict password reset upon the user's first login.

```powershell
Import-Module ActiveDirectory

$Password = ConvertTo-SecureString "AdminP@ssw0rd!" -AsPlainText -Force
$Departments = @("FrontOffice", "FoodAndBeverage", "IT", "HR")

Write-Host "Creating Departments (OUs)..." -ForegroundColor Yellow

foreach ($Dept in $Departments) {
    if (-not (Get-ADOrganizationalUnit -Filter "Name -eq '$Dept'")) {
        New-ADOrganizationalUnit -Name $Dept -Path "DC=dreihotel,DC=local"
        Write-Host " [+] Created OU: $Dept" -ForegroundColor Green
    }
}

$Staff = @(
    [PSCustomObject]@{ First="Chelseah"; Last="Padilla"; Title="Front Office Manager"; Dept="FrontOffice" }
    [PSCustomObject]@{ First="Kristel"; Last="Clay"; Title="Receptionist"; Dept="FrontOffice" }
    [PSCustomObject]@{ First="Jake"; Last="Pratt"; Title="F&B Manager"; Dept="FoodAndBeverage" }
    [PSCustomObject]@{ First="Anya"; Last="Rocha"; Title="Head Chef"; Dept="FoodAndBeverage" }
    [PSCustomObject]@{ First="Drei"; Last="Deleon"; Title="IT Director"; Dept="IT" }
    [PSCustomObject]@{ First="Zaki"; Last="Marcus"; Title="SOC Analyst"; Dept="IT" }
    [PSCustomObject]@{ First="Miles"; Last="Lamb"; Title="HR Director"; Dept="HR" }
)

Write-Host "`nCreating User Accounts..." -ForegroundColor Yellow

foreach ($User in $Staff) {
    $Username = "$($User.First.Substring(0,1).ToLower())$($User.Last.ToLower())"
    $UserPrincipalName = "$Username@dreihotel.local"
    $Path = "OU=$($User.Dept),DC=dreihotel,DC=local"

    if (-not (Get-ADUser -Filter "SamAccountName -eq '$Username'")) {
        New-ADUser -Name "$($User.First) $($User.Last)" -GivenName $User.First -Surname $User.Last -SamAccountName $Username -UserPrincipalName $UserPrincipalName -Title $User.Title -Path $Path -AccountPassword $Password -Enabled $true -ChangePasswordAtLogon $true
        
        Write-Host " [+] Created User: $($User.First) $($User.Last) -> Login: $Username" -ForegroundColor Cyan
    }
}

Write-Host "`nScript Completed!" -ForegroundColor Green
