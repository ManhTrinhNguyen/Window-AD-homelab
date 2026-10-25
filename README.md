## AD homelab architecture overview

<img width="888" height="946" alt="AD-homelab-img" src="https://github.com/user-attachments/assets/bb0121e1-2182-41b2-8834-314c1b169d87" />

## Build Active Directory Online use AWS 

### Create a Windows Server VM

Window 11 iso Enterprise 

Window Server 2022

### Create Window server 2022 

Configure my EC2 instance: 

```
Setting	My recommendation
Name	AD-Lab-DC01
AMI	Windows Server 2022 Base
Architecture	x86-64
Instance type	t3.medium
Key pair	Create an RSA key pair
Network	Default VPC initially
Security group	No public RDP access; use a secure connection
Storage	50 GiB gp3
```

### Connect to the Windows desktop using Remote Desktop (RDP)

Step 1 — Find your EC2 public IP address
1. Sign in to the AWS EC2 Console.
2. Click Instances.
3. Select your Windows Server 2022 instance.
4. In the Details tab, find Public IPv4 address.
5. Copy that address.
For example: 54.123.45.67 (example only).

If Public IPv4 address is blank, your instance doesn't have a public IP. You'll need a private connectivity method, such as Systems Manager port forwarding, instead.

Step 2 — Get your Windows Administrator password

1. Select your EC2 instance.
2. Click Connect.
3. Select the RDP client tab.
4. Click Get password.
5. Upload the private key file (.pem) you downloaded when creating the instance.
6. Click Decrypt password.
7. Copy the decrypted password.
Your username is usually: Administrator

Windows password generation may take several minutes after the instance first launches. Never share your private key or Administrator password.

Step 3 — Check your Security Group

In AWS:
1. Select your EC2 instance.
2. Open the Security tab.
3. Click the linked Security Group.
4. Open Inbound rules → Edit inbound rules.
5. Confirm this rule exists:
Type	Protocol	Port	Source
RDP	TCP	3389	My IP (/32)

Choose My IP, not Anywhere (0.0.0.0/0).

This method also requires your instance to have a public IP, a route through an internet gateway, and applicable network ACL rules.

Step 4 — Connect using Windows App on your Mac

1. Open Windows App on your Mac.
2. Select Devices.
3. Click the + button and select Add PC.
4. Enter the following:
Field	Enter
PC Name	Your EC2 Public IPv4 address
User Account	Administrator
Password	The decrypted EC2 password
Friendly Name	AWS-AD-Lab

5. Click Add.
6. Double-click your newly added PC.
7. Accept the certificate prompt only after verifying that you're connecting to your intended EC2 server. AWS documents how to compare the RDP certificate fingerprint.

### Promote Window Server 2022 to a Domain Controller 

Check internet connection : `ipconfig`

Install Domain Controller: Go to Dashboard -> Server Manager -> Add roles and features -> 3 times -> Active Directory Domain Services  -> Next 2times -> install -> After that see the message : Promote this Server to Domain Controller -> Add new forest 

Go to Active Directory and User : This is where to manage all the computer 

Create new User : Right click -> new -> User

Make user to the Admin : Right click -> propeties -> Member of -> add -> type Domain admin

### Join Window 11 to Domain 

Force DNS to talk to Domain controller : Open Search -> type dns -> Ethernet Setting -> DNS server assignment -> Manual -> IPv4 -> type the address of the Domain controller server

That mean I am telling this machine to use DNS of the Domain Controller Server

Rename the machine: Go to system -> rename 

Change the machine to Manage AD: System -> About -> Domain or Workgroup -> Change -> Domain -> Domain controller name (jsslab.local)

If not working : 
- Typing not correctly 
- Check network 
- restart both machines 
- turnoff some firewall 

### Create Organization Units 

Connect to Domain Controller 

Open Server Manager -> Tools -> Active Directory User and Computer

To Create main Company OU: Right Click transip.local -> select new -> new Organization Unit -> Keep protect container from accident deletion checked -> OK 

To create deparment OU: Right lick to new OU just created name TransIP -> new -> Organization Unit -> Create IT, HR, Finance, Workstation

To create test Employee: Right to IT unit -> new -> user -> enter employee name and login name -> next -> create password -> Select User must change password at next logon -> finish

### Reset a User's Password in Active Directory

Open Server Manager -> Tools -> Active Directories and Computers 

Expand transip.local -> Navigate to TransIP -> IT -> AnhVo

To reset password -> Right click Anh Vo -> Reset Password 

### Lock and unlock an Active Directory account

Check the domain's lockout policy

On Power shell:
```
Get-ADDefaultDomainPasswordPolicy |
Select-Object LockoutThreshold, LockoutDuration, LockoutObservationWindow
```

The important of Lockout Threshold: 
- 0 mean automatic account lockout is disabled 
- A positvie number mean account can lock after that many failed attempt, subjects to the policy 

Check anhvo's current status
```
Get-ADUser -Identity "anhvo" -Properties LockedOut |
Select-Object Name, Enabled, LockedOut
```
 
Simulate a locked account: If your lockout threshold is already enabled, you can deliberately enter an incorrect password on the Windows 11 client until the account locks.

Unclock Account : Active Directory and computers -> anhvo -> right click properties -> Account lab -> If account locked, check option unclock -> finish 

### Open group policy managment 

Open Server Manager -> Tools -> Group policy Management -> Expand Forest: transip.local -> Domains -> transips.local -> Right click Default Domain Policy and select Edit 

To configure account lockout : In Group Policy Management Editor -> Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Account Lockout Policy


### Active Directory Security Groups and Folder Permissions

To create Security Group: Server Manager → Tools → Active Directory Users and Computers -> Expand transip.local -> Open your TransIP → Finance OU -> Right-click the Finance OU → New → Group . 

- Group name : Finance-Share-RW
- Group Scope : Global 
- Group type : Security

Add Anhvo to the SG -> Double-click Finance-Share-RW  -> Select the Members tab -> Click Add -> Enter anhvo -> Click Check Names -> Click OK → Apply

To create a shared Finance folder: Open File Explorer on Windows Server -> Navigate to C:\. -> Create a folder named FinanceShared -> Right-click the folder → Properties → Sharing -> Click Advanced Sharing -> Check Share this folder -> Set the share name to FinanceShared -> Click Permissions

Configure share permissions: Remove the Everyone entry if present -> Click Add -> Enter Finance-Share-RW -> Click Check Names → OK -> Select Finance-Share-RW -> Allow Change and Read -> Click Apply → OK.

Note: Share permession and NTFS permissions work together 

Configure NTFS permissions: Right-click C:\FinanceShared → Properties -> Open the Security tab -> Click Advanced -> Click Disable inheritance, then choose Convert inherited permissions into explicit permissions -> Add Finance-Share-RW with Modify permission for this folder, subfolders, and files -> Keep SYSTEM and Administrators with Full Control.



Remove broad access entries such as Users or Authenticated Users that would otherwise grant ordinary users access. Do not remove administrative or system permissions.



Click Apply → OK.
