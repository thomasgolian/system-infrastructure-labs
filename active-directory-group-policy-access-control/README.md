# Active Directory Lab: Group Policy & Access Control Implementation

<img src="images/project-main-photo.jpg" width="900">

# Overview

Deployed and configured an Active Directory domain environment including OU structure, Group Policy management, DNS integration, and domain-joined client systems in a virtualized lab

This lab environment uses VMware networking to simulate an isolated enterprise LAN. Domain-connected systems are placed on a Host-Only network (VMnet1), forming a private subnet (192.168.56.0/24) where the domain controller provides DNS and identity services. 

NAT networking (VMnet8) is used separately for internet access when needed, enabling a dual-network design similar to real-world internal and external segmentation.

Environment:
<br>Windows Server 2025 (Domain Controller – DC01)
<br>2x Windows 11 client machines
<br>VMware Workstation
<br>Domain: `corp.local`

# Objectives

Separation of concerns: User policies vs Computer policies

OUs + GPO linkage

Build an enterprise-style Active Directory lab implementing centralized authentication, Group Policy management, network security controls, and role-based access using security groups

Configure Group Policy Objects (GPOs)

Implement SMB File Share

<br>

# Scenarios

1) Shared Drive + Department Access

2) Local Admin Restriction

3) User Lockdown (Security Hardening)

<br>

# Installation

With VMware -- We're provisioning our Windows Server VM with 8GB RAM, 4 CPU cores, 

Network adapter = host only (other domain host VMs will also be set to 'host only')

<img src="images/windows-server-set-up.jpg" width="600">

<br>

# Active Directory Configuration

On my physical host machine, we can see VMware's internal virtual switch is using `192.168.56.0/24` -- so we'll put the AD domain controller on that network

<img src="images/vmware-internal-switch.jpg" width="900">

<br>

IP addresses for our 3 Windows virtual machines:

```
Domain name: corp.local

Destination Server: DC01.corp.local

Domain Controller (DC01 server):
IP = 192.168.56.10
MASK = 255.255.255.0
DNS = 127.0.0.1

Client 1: 
IP = 192.168.56.20
MASK = 255.255.255.0
DNS = 192.168.56.10

Client 2: 
IP = 192.168.56.21
MASK = 255.255.255.0
DNS = 192.168.56.10

```

<br>

<img src="images/client-ip-config-ping.jpg" width="1100">

<br>

The server will run DNS + Active Directory

Install Active Directory Domain Services (AD DS) 

Add Roles and Features

<br>

<img src="images/ad-domainservices-configure.jpg" width="1100">

<br>

<img src="images/domain-add-services.jpg" width="700">

<br>

<img src="images/installing.jpg" width="600">

<br>

Promote AD DS to the domain controller.

We choose > Add a new forest > root domain name = corp.local

After prerequisits check, we install domain services. 

<br>

<img src="images/ds-install.jpg" width="900">

<br>

Because we created a new 'forest' -- our first domain (corp.local) becomes the forest itself (forest root domain)

Create Organization Unit (OU) `Workstations` for corp.local domain. The Group Policy Objects (GPOs) are applied to the OU.

<br>

<img src="images/create-ou.jpg" width="700">

<br>

Both client devices (Windows 11 Enterprise) are enrolled in corp.local domain.

<br>

<img src="images/enroll-domain.jpg" width="1000">

<br>

Using Administrator account to authenticate joining the domain.
<br>

<img src="images/authenticate-enroll.jpg" width="700">

<br>

Both client VMs are now connected to the domain network.

<br>

<img src="images/clients-joined.jpg" width="900">

<br>

Creating a new organizational unit (OU) > UserAccounts

Created both user accounts:

Alex Smith
<br>asmith@corp.local
<br>HR dept

Maria Garcia
<br>mgarcia@corp.local
<br>IT dept

<br>

Moved both clients to the 'Workstations' OU

In Group Policy Management > creating 'Allow Ping' object - Group Policy Object (GPO) inside Workstations OU

Group Policy Editor > Computer Configuration > Windows > Security > Firewall > Inbound Rules > Predefined Rules > File and Printer Sharing

<br>

<img src="images/allow-ping-gpo.jpg" width="900">

<br>

On both clients, command `gpupdate /force` to immediately update the clients' Group Policy. Our clients in the Workstation OU can now send / receive ICMP packets.

<br>

<img src="images/client-to-client-connectivity.jpg" width="700">

<br>

Making another Group Policy Object for the 'UserAccounts' OU where the 2 new user profiles are stored.

Changing password length:

Navigating to > Group Policy Management > corp.local > Default Domain Policy > edit > Computer config > Policies > Windows > Security > Account Policies > Password Policy

Change minimum password length to 10 characters

<br>

<img src="images/default-policy-password-legnth.jpg" width="900">

<br>

Inside Account Policies, changing the 'account lockout threshold' to 5 invalid login attempts to prevent brute force attempts, etc

In Active Directory Users and Computers, we can view more settings by enabling 'Advanced Features' in the View menu.

<br>

<img src="images/advanced-features.jpg" width="700">

<br>

Making sure checked box 'Protect object from accidental deletion' is enabled for the Workstation and UserAccounts OUs

<br>

<img src="images/protect-object.jpg" width="700">

<br>

Because this is a lab - create a GPO for the Workstations OU -- for client devices not to go to sleep.(Since client access to Power is blocked already)

GPO 'No Sleep' > Sleep Settings > Specify the unattended sleep timeout (plugged in) > Enable > `0` value as null

Easy way to view GPO details > select GPO > 'settings'

<br>

<img src="images/no-sleep-confirm.jpg" width="900">

**************************************************************************************************************

# Scenario 1 — Shared Drive + Department Access

- Create shared folders (on DC)

- Set permissions (AD groups)

- Map it as a drive via GPO

Creating folder 'Shares' on Windows Server. Location = C:\

With two subfolders inside - 'HR' and 'IT'
<br>C:\Shares\IT
<br>C:\Shares\HR

<br>

Create security groups (a new object):

<br>

<img src="images/new-object-itgroup.jpg" width="700">

<br>

Adding users to corresponding Groups via 'Member Of' in the User's properties menu.

<br>

<img src="images/users-to-groups.jpg" width="900">

<br>

Set NTFS permissions on folders. In security tab, we add corresponding groups for ability to view and 'modify' the share.

<br>

<img src="images/permissions-folder-shares.jpg" width="1100">

<br>

Because these are department-specific folders, removing standard 'Users' access is recommended so they folder shares aren't reachable from all users.

Blocking inheritance may be required > by navigating to security > advanced > 'disable inheritance'

<br>

<img src="images/disable-inheritance.jpg" width="900">

<br>

We give both IT_Group and HR_Group 'read and execute' priviledges to the parenting 'Shares' folder - which will be mapped as a shared drive afterwards.

*Users require read access to the parent share for traversal, while write permissions are controlled at the subfolder level.*

<br>

<img src="images/authenticated-users.jpg" width="900">

<br>

Network Path: `\\DC01\Shares`

<br>

Time to map the 'Shares' folder as a shared drive for client devices

In Group Policy Management Editor > create a new drive
<br>Action: create
<br>Location: `\\DC01\Shares`
<br>Drive Letter: `S`

<br>

<img src="images/map-drive.jpg" width="900">

<br>

The mapped drive (S:) is now available to both users. On left, Alex Smith (HR) attempts to open IT Dept folder but receives a permission error.

<br>

<img src="images/mapped-drive-ready.jpg" width="1100">

**************************************************************************************************************

# Scenario 2 — Local Admin Restriction

Goal:
<br>Users are NOT local admins
<br>Only IT group can be admins

Create new Workstation Group

Added 'Maria Garcia' user to the 'Workstation Admins' Group as well. (She is IT Dept) 

This Group Policy Object (GPO) will be applied to the 'Workstations' OU because we're limiting 'local' meaning the device itself. 

By using 'Restricted Groups' within Group Policy Manager, we can overwrite local administrative access. Result = now the only users to log in locally to a end user device WITH Administrative access will be those users that are members of 'Workstation Admins' group

<br>

<img src="images/mgarcia-workstation.jpg" width="900">

<br>

Before: 
<br>Admin rights were: local + uncontrolled

Now:
<br>local Admin rights are: centrally controlled by AD

Verify with `net localgroup administrators`

<br>

<img src="images/client-device.jpg" width="700">

<br>

**************************************************************************************************************

# Scenario 3 - User Lockdown (Security Hardening)

Apply to UserAccounts OU (so these changes are tied to which user is logged into the device)

<br>

GPOs to create:

- Run Restrict - removes Run command

- Control Panel Restrict - blocks permission to use Task Manager

- Task Manager Restrict - disables Control Panel access
<br>

<img src="images/all3-restrict.jpg" width="900">

<br>

Group Policy Editor again > User Configuration > Windows > Administrative Templates > Control Panel

<img src="images/restrict-controlpanel-settings.jpg" width="900">

<br>

Commands `gpresult /r` and `gpresult /r /scope computer` shows the current group policy on the client machine and user account:

<br>

<img src="images/gp-result.jpg" width="900">

<br>

If a normal user now tries to access control panel > user will receive an restriction error message:

<br>

<img src="images/restricted.jpg" width="900">

<br>

Restrict the Task Manager:

<img src="images/lock-task-manager.jpg" width="900">

<br>

Very Group Policy is working again after all 3 restrictions are in place. From client-1:

<br>

<img src="images/verify-gpo.jpg" width="600">

<br>

**************************************************************************************************************

# Final Takeaways 

Implemented role-based access control (RBAC) using Active Directory security groups and NTFS permissions

Applied Group Policy Objects (GPOs) to enforce consistent user and system configurations across domain-joined machines

Demonstrated understanding of authentication vs authorization within a domain environment

Managed organizational units (OUs) to control scope and policy application

Configured and verified secure file sharing using SMB (Server Message Block), share permissions, and NTFS permissions

Enforced least privilege principles by restricting local administrator access through centralized policy

Validated configurations using tools such as `gpresult` ensuring accurate policy 

Group Policy applies separately to user and computer objects based on OU placement, allowing granular and centralized control

Implemented multiple Group Policy Objects (GPOs) to enforce user restrictions including Control Panel, Task Manager, and Run command access

`Restricted Groups` enforces centralized control over local administrator membership, ensuring only approved Active Directory groups have elevated privileges on domain-joined machines

Implemented role-based access control by combining Active Directory group membership with NTFS permissions and Group Policy drive mapping


***************************************************************************************************************

![Main](images/project-main-photo.jpg)
