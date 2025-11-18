# Windows Server Security Hardening

## Security Hardening Implementation overview.

	Now that our Active Directory infrastructure is complete, we'll implement enterprise-grade security hardening to protect against common threats and attacks. 
	This phase focuses on systematically locking down our environment while maintaining functionality.

## Hardening Priority List:
		
		- Account Security - Enhance password policies and account protections
		- Network Hardening - Secure services, firewall rules, and protocols
		- System Hardening - Apply security templates and user rights
		- GPO Security - Harden Group Policy permissions and settings
	
# First step, account security hardening

	We will begin with account security because compromised credentials are the #1 attack vector. Strong accounts form the foundation that all other security layers depend on.
	
		First I'll strengthen password policies by :
			
			- Increasing the minimum password length from 12 to 14
			- Increasing the password history to prevent users from cycling their favourite passwords
		
		Then I'll implement lockout refinements such as:
			
			- Stricter lockout and password policies for admins
			- Track failed logons across entire domain
			- Increase lockout duration for repeated attacks
		
		I'll also harden the service accounts by:
			
			- Appllying "logon as service" restrictions
			- Configure managd service accounts where possible
			- Implement regular password rotation practices
			
		Then I'll finish by configuring session security with:
		
			- Session timeouts for idle accounts
			- Limiting concurrent logons for sensitive accounts
			- Smart card requirements for admins
			
### Increasing minimum password length for everyone, then verifying

	Set-ADDefaultDomainPasswordPolicy
	-Identity greg.local 
	-MinPasswordLength 14

	MinPasswordLength PasswordHistoryCount
	----------------- --------------------
				   14                   24
 
### Now to change the password history count, and verify
	
	Set-ADDefaultDomainPasswordPolicy 
	-Identity greg.local 
	-PasswordHistoryCount 48
	
	MinPasswordLength PasswordHistoryCount
	----------------- --------------------
				   14                   48
	
### Account lockout refinements will go ahead like so, first I'll increase the lockout duration from 30 to 60 minutes for repeated attacks

	Set-ADDefaultDomainPasswordPolicy
	-Identity greg.local 
	-LockoutDuration "01:00:00"

### Ill also extend the observation window from 30 to 60 minutes so attackers have to wait longer before repeated attempts.

	Set-ADDefaultDomainPasswordPolicy
	-Identity greg.local 
	-LockoutObservationWindow "01:00:00"

### And the test to check setting have changed

	LockoutThreshold LockoutDuration LockoutObservationWindow
	---------------- --------------- ------------------------
				   5 01:00:00        01:00:00


### Now that domain wide users have better security policies, Ill go on to enchance admin users security with a fine grained password policy, link to the relevant Admin OUs and check the policy is active 

	New-ADFineGrainedPasswordPolicy 
	-Name "AdminPasswordPolicy" 
	-Precedence 1 							# Takes highest precedence 
	-MinPasswordLength 16 					# Increase minimum password length
	-LockoutThreshold 3 					# 3 Attempts before lockout not 5!
	-LockoutDuration "02:00:00" 			# Locked out for 2 hours not 1
	-LockoutObservationWindow "01:00:00" 	
	-ComplexityEnabled $true 	
	-MaxPasswordAge "90.00:00:00"			# Admins must change password every 90 days	 
	-MinPasswordAge "1.00:00:00" 			# Have to wait 1 day before changin password once changed
	-PasswordHistoryCount 48				 
	-ReversibleEncryptionEnabled $false
	
	Add-ADFineGrainedPasswordPolicySubject 
	-Identity "AdminPasswordPolicy" 
	-Subjects "IT-Admins", "Domain Admins", "Enterprise Admins"

	AppliesTo                   : {CN=IT-Admins,OU=Administrative,OU=Security Groups,DC=greg,DC=local, CN=Domain Admins,CN=Users,DC=greg,DC=local, CN=Enterprise Admins,CN=Users,DC=greg,DC=local}
	ComplexityEnabled           : True
	DistinguishedName           : CN=AdminPasswordPolicy,CN=Password Settings Container,CN=System,DC=greg,DC=local
	LockoutDuration             : 02:00:00
	LockoutObservationWindow    : 01:00:00
	LockoutThreshold            : 3
	MaxPasswordAge              : 90.00:00:00
	MinPasswordAge              : 1.00:00:00
	MinPasswordLength           : 16
	Name                        : AdminPasswordPolicy
	ObjectClass                 : msDS-PasswordSettings
	ObjectGUID                  : 307a3e3f-1d14-4ba8-8011-f7b6b5caf549
	PasswordHistoryCount        : 48
	Precedence                  : 1
	ReversibleEncryptionEnabled : False

### Now ill move on to hardening service accounts. Service accounts are prime targets for bad actors as they often have broad permissions, theyre rarely monitored (compared to human accounts) and their passwords never change.

### Ill start by checking current service account settings
	
	Get-ADUser -Filter {Name -like "svc-*"} 
	| Select-Object 
	Name, 
	Enabled, 
	PasswordNeverExpires, 
	CannotChangePassword, 
	LogonWorkstations
	
		
	Name                 : svc-backup
	Enabled              : False
	PasswordNeverExpires :
	CannotChangePassword :
	LogonWorkstations    :

	Name                 : svc-monitoring
	Enabled              : True
	PasswordNeverExpires :
	CannotChangePassword :
	LogonWorkstations    :

### I can see here that both svc backup and monitoring files settings need to be configured. Ill start with svc-backup by entering 

	Set-ADUser 
	-Identity "svc-backup" 
	-Enabled $true 
	-PasswordNeverExpires $true 
	-CannotChangePassword $true

### I get an error message 
	
	Set-ADUser : The password does not meet the length, complexity, or history requirement of the domain.
	At line:1 char:1
	+ Set-ADUser -Identity "svc-backup" -Enabled $true -PasswordNeverExpire 
	+ ------------------------------------------------------------------------------------------------
		+ CategoryInfo          : InvalidData: (svc-backup:ADUser) [Set-ADUser], ADPasswordComplexityException
		+ FullyQualifiedErrorId : ActiveDirectoryServer:1325,Microsoft.ActiveDirectory.Management.Commands.SetADUser

### This tells me the service account itself has a weak password that doesnt meet our new domain requirements, and when we try to enable/modify the account, AD is checking the current password and rejecting the operation! 

### At least it shows our new policies are working! :) One possible fix for this would be to use group managed service accounts. (gMSA) This is automatic password management for service accounts. We could also coordinate with teams and organise scheduled maintenence 

### However for now ill just change the password manually.

	Set-ADAccountPassword 
	-Identity "svc-backup" 
	-NewPassword (ConvertTo-SecureString "xxxxxxxxxxxxxxxx" -AsPlainText -Force) 
	-Reset
	
### Now I can apply the service account settings hopefully

	Set-ADUser 
	-Identity "svc-backup" 
	-Enabled $true 
	-PasswordNeverExpires $true 
	-CannotChangePassword $true
	
### No error looking good, now time to do the same for svc-monitoring

	Set-ADAccountPassword 
	-Identity "svc-monitoring" 
	-NewPassword (ConvertTo-SecureString "xxxxxxxxxxxxxxxx" 
	-AsPlainText -Force) 
	-Reset
	
	Set-ADUser 
	-Identity "svc-monitoring" 
	-PasswordNeverExpires $true 
	-CannotChangePassword $true
	
	Name           Enabled PasswordNeverExpires CannotChangePassword
	----           ------- -------------------- --------------------
	svc-backup        True
	svc-monitoring    True

### I see we havent configured Password never expires and CannotChangePassword - These will expire based on domain policiy, critical fix here! Ill do both at the same time using the hashtable unction i learned about :)

	"svc-backup", "svc-monitoring" 
	| ForEach-Object {
		Set-ADUser
		-Identity $_ 
		-PasswordNeverExpires $true 
		-CannotChangePassword $true
	}

### Ill now check with

	 Get-ADUser 
	 -Filter {Name -like "svc-*"} 
	 | Select-Object 
	 Name, 
	 Enabled, 
	 PasswordNeverExpires, 
	 CannotChangePassword
	 
	
	Name           Enabled PasswordNeverExpires CannotChangePassword
	----           ------- -------------------- --------------------
	svc-backup        True
	svc-monitoring    True

### Oh no the changes i made seem to have not taken effect. Still no true or false values in the colums we changed configs for.


	Get-ADUser 
	-Filter {Name -like "svc-*"} 
	-Properties PasswordNeverExpires, CannotChangePassword 
	| Select-Object
	Name, 
	Enabled, 
	PasswordNeverExpires, 
	CannotChangePassword
	
### Doing it the non lazy way shows what I wanted to see

	Name           Enabled PasswordNeverExpires CannotChangePassword
	----           ------- -------------------- --------------------
	svc-backup        True                 True                 True
	svc-monitoring    True                 True                 True

## Note: Always use -Properties when I need to see specific attributes otherwise I might see misleading empty attributes!

### I've already secured our service accounts against interactive login while maintaining their background service functionality in AD-group-policies. The userAccountControl flags are properly set, preventing svc-backup and svc-monitoring from being used for RDP or console access while preserving their ability to run essential services.

## Session Security Implementation

### Next I'll configure concurrent login limits for administrative accounts to prevent credential sharing and limit exposure from compromised admin sessions. While we already have screen timeouts and forced logoff for expired accounts, concurrent session limits add another layer of protection against admin account misuse.

### I'll limit IT admin accounts to 2 concurrent sessions maximum - enough for legitimate administrative work while preventing widespread credential sharing or an attacker maintaining multiple persistent connections.

### The implementation will use Group Policy to enforce these limits across all domain-joined systems, ensuring consistent protection regardless of which computer administrative accounts access.

### First Ill create a new GPO and link relevant OU'same
	
	New-GPO -Name "Admin Session Limits"
	New-GPLink -Name "Admin Session Limits" -Target "OU=IT,OU=User Accounts,DC=greg,DC=local"

### Then ill set max concurrent RDP sessions to 2

	Set-GPRegistryValue 
	Name "Admin Session Limits" 
	-Key "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" 
	-ValueName "MaxInstanceCount" # This is to set the maximum RDP instance amount
	-Type DWord 
	-Value 2 # This is the number to assign to max instances
	
### Then ill verify with
	
	Get-GPRegistryValue 
	-Name "Admin Session Limits"
	-Key "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services"


	KeyPath     : SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services
	FullKeyPath : HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services
	Hive        : LocalMachine
	PolicyState : Set
	Value       : 2
	Type        : DWord
	ValueName   : MaxInstanceCount
	HasValue    : True
	
### And we can see the values have been set.

## Next Ill begin the network hardening. This will consist of:
	
	- Disabling unnecessary windows services
	- Securing essential services
	- Firewall rules configuration
	- Protocol Security
	- Restricting administrative access
	- Configuring network segmentation
	
### I'll start by identifying and disabling high-risk, unnecessary windows services to immediatly reduce the attack surface of my servers and workstations.

### First Ill get a list of running services that are set to start automatically 

	Get-Service 
	| Where-Object {$_.Status -eq 'Running' -and $_.StartType -eq 'Automatic'} 
	| Sort-Object 
	DisplayName
	

	Status   Name               DisplayName
	------   ----               -----------
	Running  NTDS               Active Directory Domain Services
	Running  ADWS               Active Directory Web Services
	Running  BFE                Base Filtering Engine
	Running  EventSystem        COM+ Event System
	Running  DiagTrack          Connected User Experiences and Tele...
	Running  CoreMessagingRe... CoreMessaging
	Running  CryptSvc           Cryptographic Services
	Running  DcomLaunch         DCOM Server Process Launcher
	Running  Dfs                DFS Namespace
	Running  DFSR               DFS Replication
	Running  Dhcp               DHCP Client
	Running  DPS                Diagnostic Policy Service
	Running  DispBrokerDeskt... Display Policy Service
	Running  MSDTC              Distributed Transaction Coordinator
	Running  Dnscache           DNS Client
	Running  DNS                DNS Server
	Running  EFS                Encrypting File System (EFS)
	Running  gpsvc              Group Policy Client
	Running  IsmServ            Intersite Messaging
	Running  iphlpsvc           IP Helper
	Running  Kdc                Kerberos Key Distribution Center
	Running  LSM                Local Session Manager
	Running  WinDefend          Microsoft Defender Antivirus Service
	Running  MDCoreSvc          Microsoft Defender Core Service
	Running  Netlogon           Netlogon
	Running  nsi                Network Store Interface Service
	Running  Power              Power
	Running  RpcSs              Remote Procedure Call (RPC)
	Running  RpcEptMapper       RPC Endpoint Mapper
	Running  SamSs              Security Accounts Manager
	Running  LanmanServer       Server
	Running  StateRepository    State Repository Service
	Running  SysMain            SysMain
	Running  SENS               System Event Notification Service
	Running  SystemEventsBroker System Events Broker
	Running  Schedule           Task Scheduler
	Running  UsoSvc             Update Orchestrator Service
	Running  UALSVC             User Access Logging Service
	Running  UserManager        User Manager
	Running  ProfSvc            User Profile Service
	Running  mpssvc             Windows Defender Firewall
	Running  EventLog           Windows Event Log
	Running  WLMS               Windows Licensing Monitoring Service
	Running  Winmgmt            Windows Management Instrumentation
	Running  WinRM              Windows Remote Management (WS-Manag...
	Running  W32Time            Windows Time
	Running  LanmanWorkstation  Workstation

### I'll disable three non-essential services to reduce our attack surface and improve server performance. DiagTrack collects telemetry data posing privacy risks, SysMain wastes memory on server workloads where it provides no benefit, and UALSVC duplicates logging functions we already handle through proper audit policies. Disabling these eliminates unnecessary resource consumption and potential vulnerabilities without impacting core domain operations.

	"DiagTrack", "SysMain", "UALSVC" 
	| ForEach-Object {
	Set-Service 
	-Name $_ 
	-StartupType Disabled # Changes service so it isnt automatic on startup
	-PassThru 			  # Passes the service object to the next Command
	| Stop-Service; 	  # Immediatley stops running the service	
	Write-Host "Disabled and stopped: $_" } # Shows confirmation message for each service

	Disabled and stopped: DiagTrack
	Disabled and stopped: SysMain
	Disabled and stopped: UALSVC # This shows the commmand worked properley! (We will still verify though)
	
	Get-CimInstance 
	Win32_Service 
	| Where-Object {
	$_.Name 
	-in "DiagTrack",
	"SysMain", 
	"UALSVC"} 
	| Select-Object
	Name, 
	State, 		# Current status (Running/Stopped)
	StartMode 	# Startup configuration (Auto/Manual/Disabled)
	
	
	Name      State   StartMode
	----      -----   ---------
	DiagTrack Stopped Disabled
	SysMain   Stopped Disabled
	UALSVC    Stopped Disabled

### This confirms my actions.

### Next ill configure the firewall settings by unblocking unnecessary ports. This will reduce the attack surface by closing network pathways that arent required for domain operations, preventing unauthoriasd accept attempts through services

### Ill start by checking the current firewall profile status 

	Get-NetFirewallProfile  # Gets all firewall rules
	| Select-Object 
	Name, 			
	Enabled	
	
### This just shows the firewall is on for all profiles but doesnt show whats allowed through. Lets build on the command to get a better firewall analysis

	Get-NetFirewallRule 
	| Where-Object {$_.Enabled -eq 'True'} # filter (Where-Object) this rules enabled property ($_.Enabled) is equal to true (-eq "True")
	| Select-Object 
	DisplayName, 
	Direction, 							   # Inbound or Outbound
	Action,								   # Allow or Block
	Profile 							   # Domain, private or public network
	| Sort-Object
	DisplayName	
	
### This shows us all enabled firewall rules on this computer 

	DisplayName                                                                  Direction Action                 Profile
	-----------                                                                  --------- ------                 -------
	Active Directory Domain Controller -  Echo Request (ICMPv4-In)                 Inbound  Allow                     Any
	Active Directory Domain Controller -  Echo Request (ICMPv4-Out)               Outbound  Allow                     Any
	Active Directory Domain Controller -  Echo Request (ICMPv6-In)                 Inbound  Allow                     Any
	Active Directory Domain Controller -  Echo Request (ICMPv6-Out)               Outbound  Allow                     Any
	Active Directory Domain Controller - LDAP (TCP-In)                             Inbound  Allow                     Any
	Active Directory Domain Controller - LDAP (UDP-In)                             Inbound  Allow                     Any
	Active Directory Domain Controller - LDAP for Global Catalog (TCP-In)          Inbound  Allow                     Any
	Active Directory Domain Controller - NetBIOS name resolution (UDP-In)          Inbound  Allow                     Any
	Active Directory Domain Controller - SAM/LSA (NP-TCP-In)                       Inbound  Allow                     Any
	Active Directory Domain Controller - SAM/LSA (NP-UDP-In)                       Inbound  Allow                     Any
	Active Directory Domain Controller - Secure LDAP (TCP-In)                      Inbound  Allow                     Any
	Active Directory Domain Controller - Secure LDAP for Global Catalog (TCP-In)   Inbound  Allow                     Any
	Active Directory Domain Controller - W32Time (NTP-UDP-In)                      Inbound  Allow                     Any
	Active Directory Domain Controller (RPC)                                       Inbound  Allow                     Any
	Active Directory Domain Controller (RPC-EPMAP)                                 Inbound  Allow                     Any
	Active Directory Domain Controller (TCP-Out)                                  Outbound  Allow                     Any
	Active Directory Domain Controller (UDP-Out)                                  Outbound  Allow                     Any
	Active Directory Web Services (TCP-In)                                         Inbound  Allow                     Any
	Active Directory Web Services (TCP-Out)                                       Outbound  Allow                     Any
	All Outgoing (TCP)                                                            Outbound  Allow                     Any
	All Outgoing (UDP)                                                            Outbound  Allow                     Any
	AllJoyn Router (TCP-In)                                                        Inbound  Allow         Domain, Private
	AllJoyn Router (TCP-Out)                                                      Outbound  Allow         Domain, Private
	AllJoyn Router (UDP-In)                                                        Inbound  Allow         Domain, Private
	AllJoyn Router (UDP-Out)                                                      Outbound  Allow         Domain, Private
	Connected User Experiences and Telemetry                                      Outbound  Allow                     Any
	Core Networking - Destination Unreachable (ICMPv6-In)                          Inbound  Allow                     Any
	Core Networking - Destination Unreachable Fragmentation Needed (ICMPv4-In)     Inbound  Allow                     Any
	Core Networking - DNS (UDP-Out)                                               Outbound  Allow                     Any
	Core Networking - Dynamic Host Configuration Protocol (DHCP-In)                Inbound  Allow                     Any
	Core Networking - Dynamic Host Configuration Protocol (DHCP-Out)              Outbound  Allow                     Any
	Core Networking - Dynamic Host Configuration Protocol for IPv6(DHCPV6-In)      Inbound  Allow                     Any
	Core Networking - Dynamic Host Configuration Protocol for IPv6(DHCPV6-Out)    Outbound  Allow                     Any
	Core Networking - Group Policy (LSASS-Out)                                    Outbound  Allow                  Domain
	Core Networking - Group Policy (NP-Out)                                       Outbound  Allow                  Domain
	Core Networking - Group Policy (TCP-Out)                                      Outbound  Allow                  Domain
	Core Networking - Internet Group Management Protocol (IGMP-In)                 Inbound  Allow                     Any
	Core Networking - Internet Group Management Protocol (IGMP-Out)               Outbound  Allow                     Any
	Core Networking - IPHTTPS (TCP-In)                                             Inbound  Allow                     Any
	Core Networking - IPHTTPS (TCP-Out)                                           Outbound  Allow                     Any
	Core Networking - IPv6 (IPv6-In)                                               Inbound  Allow                     Any
	Core Networking - IPv6 (IPv6-Out)                                             Outbound  Allow                     Any
	Core Networking - Multicast Listener Done (ICMPv6-In)                          Inbound  Allow                     Any
	Core Networking - Multicast Listener Done (ICMPv6-Out)                        Outbound  Allow                     Any
	Core Networking - Multicast Listener Query (ICMPv6-In)                         Inbound  Allow                     Any
	Core Networking - Multicast Listener Query (ICMPv6-Out)                       Outbound  Allow                     Any
	Core Networking - Multicast Listener Report (ICMPv6-In)                        Inbound  Allow                     Any
	Core Networking - Multicast Listener Report (ICMPv6-Out)                      Outbound  Allow                     Any
	Core Networking - Multicast Listener Report v2 (ICMPv6-In)                     Inbound  Allow                     Any
	Core Networking - Multicast Listener Report v2 (ICMPv6-Out)                   Outbound  Allow                     Any
	Core Networking - Neighbor Discovery Advertisement (ICMPv6-In)                 Inbound  Allow                     Any
	Core Networking - Neighbor Discovery Advertisement (ICMPv6-Out)               Outbound  Allow                     Any
	Core Networking - Neighbor Discovery Solicitation (ICMPv6-In)                  Inbound  Allow                     Any
	Core Networking - Neighbor Discovery Solicitation (ICMPv6-Out)                Outbound  Allow                     Any
	Core Networking - Packet Too Big (ICMPv6-In)                                   Inbound  Allow                     Any
	Core Networking - Packet Too Big (ICMPv6-Out)                                 Outbound  Allow                     Any
	Core Networking - Parameter Problem (ICMPv6-In)                                Inbound  Allow                     Any
	Core Networking - Parameter Problem (ICMPv6-Out)                              Outbound  Allow                     Any
	Core Networking - Router Advertisement (ICMPv6-In)                             Inbound  Allow                     Any
	Core Networking - Router Advertisement (ICMPv6-Out)                           Outbound  Allow                     Any
	Core Networking - Router Solicitation (ICMPv6-In)                              Inbound  Allow                     Any
	Core Networking - Router Solicitation (ICMPv6-Out)                            Outbound  Allow                     Any
	Core Networking - Teredo (UDP-In)                                              Inbound  Allow                     Any
	Core Networking - Teredo (UDP-Out)                                            Outbound  Allow                     Any
	Core Networking - Time Exceeded (ICMPv6-In)                                    Inbound  Allow                     Any
	Core Networking - Time Exceeded (ICMPv6-Out)                                  Outbound  Allow                     Any
	Delivery Optimization (TCP-In)                                                 Inbound  Allow                     Any
	Delivery Optimization (UDP-In)                                                 Inbound  Allow                     Any
	DFS Management (DCOM-In)                                                       Inbound  Allow                     Any
	DFS Management (SMB-In)                                                        Inbound  Allow                     Any
	DFS Management (TCP-In)                                                        Inbound  Allow                     Any
	DFS Management (WMI-In)                                                        Inbound  Allow                     Any
	DFS Replication (RPC-EPMAP)                                                    Inbound  Allow                     Any
	DFS Replication (RPC-In)                                                       Inbound  Allow                     Any
	DNS (TCP, Incoming)                                                            Inbound  Allow                     Any
	DNS (UDP, Incoming)                                                            Inbound  Allow                     Any
	File and Printer Sharing (Echo Request - ICMPv4-In)                            Inbound  Allow Domain, Private, Public
	File and Printer Sharing (Echo Request - ICMPv4-Out)                          Outbound  Allow Domain, Private, Public
	File and Printer Sharing (Echo Request - ICMPv6-In)                            Inbound  Allow Domain, Private, Public
	File and Printer Sharing (Echo Request - ICMPv6-Out)                          Outbound  Allow Domain, Private, Public
	File and Printer Sharing (LLMNR-UDP-In)                                        Inbound  Allow Domain, Private, Public
	File and Printer Sharing (LLMNR-UDP-Out)                                      Outbound  Allow Domain, Private, Public
	File and Printer Sharing (NB-Datagram-In)                                      Inbound  Allow Domain, Private, Public
	File and Printer Sharing (NB-Datagram-Out)                                    Outbound  Allow Domain, Private, Public
	File and Printer Sharing (NB-Name-In)                                          Inbound  Allow Domain, Private, Public
	File and Printer Sharing (NB-Name-Out)                                        Outbound  Allow Domain, Private, Public
	File and Printer Sharing (NB-Session-In)                                       Inbound  Allow Domain, Private, Public
	File and Printer Sharing (NB-Session-Out)                                     Outbound  Allow Domain, Private, Public
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv4-In)              Inbound  Allow                  Public
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv4-Out)            Outbound  Allow                  Public                                                                                                                        File and Printer Sharing (Restrictive) (Echo Request - ICMPv6-In)              Inbound  Allow                  Public                                                                                                                        File and Printer Sharing (Restrictive) (Echo Request - ICMPv6-Out)            Outbound  Allow                  Public                                                                                                                        File and Printer Sharing (Restrictive) (LLMNR-UDP-In)                          Inbound  Allow                  Public                                                                                                                        File and Printer Sharing (Restrictive) (LLMNR-UDP-Out)                        Outbound  Allow                  Public                                                                                                                        File and Printer Sharing (Restrictive) (SMB-In)                                Inbound  Allow                  Public                                                                                                                        File and Printer Sharing (Restrictive) (SMB-Out)                              Outbound  Allow                  Public
	File and Printer Sharing (Restrictive) (Spooler Service - RPC)                 Inbound  Allow                  Public
	File and Printer Sharing (Restrictive) (Spooler Service - RPC-EPMAP)           Inbound  Allow                  Public
	File and Printer Sharing (Restrictive) (Spooler Service Worker - RPC)          Inbound  Allow                  Public
	File and Printer Sharing (SMB-In)                                              Inbound  Allow Domain, Private, Public
	File and Printer Sharing (SMB-Out)                                            Outbound  Allow Domain, Private, Public
	File and Printer Sharing (Spooler Service - RPC)                               Inbound  Allow Domain, Private, Public
	File and Printer Sharing (Spooler Service - RPC-EPMAP)                         Inbound  Allow Domain, Private, Public
	File and Printer Sharing (Spooler Service Worker - RPC)                        Inbound  Allow Domain, Private, Public
	File Replication (RPC)                                                         Inbound  Allow                     Any
	File Replication (RPC-EPMAP)                                                   Inbound  Allow                     Any
	File Server Remote Management (DCOM-In)                                        Inbound  Allow                     Any
	File Server Remote Management (SMB-In)                                         Inbound  Allow                     Any
	File Server Remote Management (WMI-In)                                         Inbound  Allow                     Any
	Kerberos Key Distribution Center - PCR (TCP-In)                                Inbound  Allow                     Any
	Kerberos Key Distribution Center - PCR (UDP-In)                                Inbound  Allow                     Any
	Kerberos Key Distribution Center (TCP-In)                                      Inbound  Allow                     Any
	Kerberos Key Distribution Center (UDP-In)                                      Inbound  Allow                     Any
	mDNS (UDP-In)                                                                  Inbound  Allow                  Public
	mDNS (UDP-In)                                                                  Inbound  Allow                  Domain
	mDNS (UDP-In)                                                                  Inbound  Allow                 Private
	mDNS (UDP-Out)                                                                Outbound  Allow                  Public
	mDNS (UDP-Out)                                                                Outbound  Allow                 Private
	mDNS (UDP-Out)                                                                Outbound  Allow                  Domain
	Microsoft Key Distribution Service (RPC EPMAP)                                 Inbound  Allow                     Any
	Microsoft Key Distribution Service (RPC)                                       Inbound  Allow                     Any
	OpenSSH SSH Server (sshd)                                                      Inbound  Allow                 Private
	Remote Desktop - Shadow (TCP-In)                                               Inbound  Allow                     Any
	Remote Desktop - User Mode (TCP-In)                                            Inbound  Allow                     Any
	Remote Desktop - User Mode (UDP-In)                                            Inbound  Allow                     Any
	RPC (TCP, Incoming)                                                            Inbound  Allow                     Any
	RPC Endpoint Mapper (TCP, Incoming)                                            Inbound  Allow                     Any
	Windows Device Management Certificate Installer (TCP out)                     Outbound  Allow                     Any
	Windows Device Management Device Enroller (TCP out)                           Outbound  Allow                     Any
	Windows Management Instrumentation (ASync-In)                                  Inbound  Allow Domain, Private, Public
	Windows Management Instrumentation (DCOM-In)                                   Inbound  Allow Domain, Private, Public
	Windows Management Instrumentation (WMI-In)                                    Inbound  Allow Domain, Private, Public
	Windows Management Instrumentation (WMI-Out)                                  Outbound  Allow Domain, Private, Public
	Windows Remote Management (HTTP-In)                                            Inbound  Allow                  Public
	Windows Remote Management (HTTP-In)                                            Inbound  Allow         Domain, Private
	
### We also need to see specific the specific ports that are open and their relevant info

	Get-NetFirewallRule | Where-Object {$_.Enabled -eq 'True'}  # Gets all enabled firewall rules
	| Get-NetFirewallPortFilter 								# Gets the actual port/protocol details for each rule 
	| Select-Object Protocol, LocalPort, RemotePort 			# Shows protocol and port numbers
	| Where-Object {$_.LocalPort} 								# Filters out rules with no local port
	| Sort-Object LocalPort -Unique								# Sorts by port numbers and removes duplicates
	
### This provides us with the protocols and port numbers


	Protocol LocalPort   RemotePort
	-------- ---------   ----------
	UDP      123         Any
	TCP      135         Any
	UDP      137         Any
	UDP      138         Any
	TCP      139         Any
	TCP      22          Any
	TCP      3268        Any
	TCP      3269        Any
	TCP      3389        Any
	TCP      389         Any
	TCP      445         Any
	TCP      464         Any
	TCP      49152-65535 {80, 443}
	TCP      53          Any
	UDP      5353        Any
	UDP      5355        Any
	UDP      546         547
	TCP      5985        Any
	TCP      636         Any
	UDP      68          67
	UDP      7680        Any
	TCP      88          Any
	TCP      9389        Any
	TCP      9955        Any
	UDP      Any         5353
	TCP      IPHTTPSIn   Any
	ICMPv6   RPC         Any
	TCP      RPCEPMap    Any
	UDP      Teredo      Any
	
### Ideally I want to see each port once, with all the rules that use it.

	Get-NetFirewallRule | Where-Object {$_.Enabled -eq 'True'} |
	ForEach-Object {												# Do this for every single firewalll 
		$Rule = $_													# Remember this current Rule
		$PortFilter = $Rule | Get-NetFirewallPortFilter				# look up the port numbers for this specific rule
		[PSCustomObject]@{											# This creates a custom object
			DisplayName = $Rule.DisplayName							# with 
			Protocol = $PortFilter.Protocol							# These
			LocalPort = $PortFilter.LocalPort						# four 
			RemotePort = $PortFilter.RemotePort						# objects
		}
	} | Where-Object {$_.LocalPort} 								# Removers rules that dont have specific port numbers
	| Group-Object LocalPort 										# groups objects together by 'LocalPort'
	| Sort-Object Name 												# sorts port groups in order
	| ForEach-Object {
		Write-Host "`nPort $($_.Name):" -ForegroundColor Yellow		# Prints a yellow header witn the port name and nuymber
		$_.Group | Format-Table -AutoSize
	}
	
### This gives us an extensive organised list of ports and all the relevant info associated.

	Port 123:

	DisplayName                                               Protocol LocalPort RemotePort
	-----------                                               -------- --------- ----------
	Active Directory Domain Controller - W32Time (NTP-UDP-In) UDP      123       Any



	Port 135:

	DisplayName                                  Protocol LocalPort RemotePort
	-----------                                  -------- --------- ----------
	Windows Management Instrumentation (DCOM-In) TCP      135       Any
	DFS Management (DCOM-In)                     TCP      135       Any
	File Server Remote Management (DCOM-In)      TCP      135       Any



	Port 137:

	DisplayName                           Protocol LocalPort RemotePort
	-----------                           -------- --------- ----------
	File and Printer Sharing (NB-Name-In) UDP      137       Any



	Port 138:

	DisplayName                                                           Protocol LocalPort RemotePort
	-----------                                                           -------- --------- ----------
	File and Printer Sharing (NB-Datagram-In)                             UDP      138       Any
	Active Directory Domain Controller - NetBIOS name resolution (UDP-In) UDP      138       Any



	Port 139:

	DisplayName                              Protocol LocalPort RemotePort
	-----------                              -------- --------- ----------
	File and Printer Sharing (NB-Session-In) TCP      139       Any



	Port 22:

	DisplayName               Protocol LocalPort RemotePort
	-----------               -------- --------- ----------
	OpenSSH SSH Server (sshd) TCP      22        Any



	Port 3268:

	DisplayName                                                           Protocol LocalPort RemotePort
	-----------                                                           -------- --------- ----------
	Active Directory Domain Controller - LDAP for Global Catalog (TCP-In) TCP      3268      Any



	Port 3269:

	DisplayName                                                                  Protocol LocalPort RemotePort
	-----------                                                                  -------- --------- ----------
	Active Directory Domain Controller - Secure LDAP for Global Catalog (TCP-In) TCP      3269      Any



	Port 3389:

	DisplayName                         Protocol LocalPort RemotePort
	-----------                         -------- --------- ----------
	Remote Desktop - User Mode (TCP-In) TCP      3389      Any
	Remote Desktop - User Mode (UDP-In) UDP      3389      Any



	Port 389:

	DisplayName                                        Protocol LocalPort RemotePort
	-----------                                        -------- --------- ----------
	Active Directory Domain Controller - LDAP (TCP-In) TCP      389       Any
	Active Directory Domain Controller - LDAP (UDP-In) UDP      389       Any



	Port 445:

	DisplayName                                              Protocol LocalPort RemotePort
	-----------                                              -------- --------- ----------
	File and Printer Sharing (SMB-In)                        TCP      445       Any
	DFS Management (SMB-In)                                  TCP      445       Any
	Active Directory Domain Controller - SAM/LSA (NP-TCP-In) TCP      445       Any
	Active Directory Domain Controller - SAM/LSA (NP-UDP-In) UDP      445       Any
	File and Printer Sharing (Restrictive) (SMB-In)          TCP      445       Any
	File Server Remote Management (SMB-In)                   TCP      445       Any



	Port 464:

	DisplayName                                     Protocol LocalPort RemotePort
	-----------                                     -------- --------- ----------
	Kerberos Key Distribution Center - PCR (TCP-In) TCP      464       Any
	Kerberos Key Distribution Center - PCR (UDP-In) UDP      464       Any



	Port 49152-65535:

	DisplayName                                               Protocol LocalPort   RemotePort
	-----------                                               -------- ---------   ----------
	Windows Device Management Certificate Installer (TCP out) TCP      49152-65535 Any
	Windows Device Management Device Enroller (TCP out)       TCP      49152-65535 {80, 443}



	Port 53:

	DisplayName         Protocol LocalPort RemotePort
	-----------         -------- --------- ----------
	DNS (UDP, Incoming) UDP      53        Any
	DNS (TCP, Incoming) TCP      53        Any



	Port 5353:

	DisplayName   Protocol LocalPort RemotePort
	-----------   -------- --------- ----------
	mDNS (UDP-In) UDP      5353      Any
	mDNS (UDP-In) UDP      5353      Any
	mDNS (UDP-In) UDP      5353      Any



	Port 5355:

	DisplayName                                           Protocol LocalPort RemotePort
	-----------                                           -------- --------- ----------
	File and Printer Sharing (LLMNR-UDP-In)               UDP      5355      Any
	File and Printer Sharing (Restrictive) (LLMNR-UDP-In) UDP      5355      Any



	Port 546:

	DisplayName                                                                Protocol LocalPort RemotePort
	-----------                                                                -------- --------- ----------
	Core Networking - Dynamic Host Configuration Protocol for IPv6(DHCPV6-In)  UDP      546       547
	Core Networking - Dynamic Host Configuration Protocol for IPv6(DHCPV6-Out) UDP      546       547



	Port 5985:

	DisplayName                         Protocol LocalPort RemotePort
	-----------                         -------- --------- ----------
	Windows Remote Management (HTTP-In) TCP      5985      Any
	Windows Remote Management (HTTP-In) TCP      5985      Any



	Port 636:

	DisplayName                                               Protocol LocalPort RemotePort
	-----------                                               -------- --------- ----------
	Active Directory Domain Controller - Secure LDAP (TCP-In) TCP      636       Any



	Port 68:

	DisplayName                                                      Protocol LocalPort RemotePort
	-----------                                                      -------- --------- ----------
	Core Networking - Dynamic Host Configuration Protocol (DHCP-In)  UDP      68        67
	Core Networking - Dynamic Host Configuration Protocol (DHCP-Out) UDP      68        67



	Port 7680:

	DisplayName                    Protocol LocalPort RemotePort
	-----------                    -------- --------- ----------
	Delivery Optimization (UDP-In) UDP      7680      Any
	Delivery Optimization (TCP-In) TCP      7680      Any



	Port 88:

	DisplayName                               Protocol LocalPort RemotePort
	-----------                               -------- --------- ----------
	Kerberos Key Distribution Center (TCP-In) TCP      88        Any
	Kerberos Key Distribution Center (UDP-In) UDP      88        Any



	Port 9389:

	DisplayName                            Protocol LocalPort RemotePort
	-----------                            -------- --------- ----------
	Active Directory Web Services (TCP-In) TCP      9389      Any



	Port 9955:

	DisplayName             Protocol LocalPort RemotePort
	-----------             -------- --------- ----------
	AllJoyn Router (TCP-In) TCP      9955      Any



	Port Any:

	DisplayName                                                     Protocol LocalPort RemotePort
	-----------                                                     -------- --------- ----------
	Core Networking - IPv6 (IPv6-In)                                41       Any       Any
	mDNS (UDP-Out)                                                  UDP      Any       5353
	mDNS (UDP-Out)                                                  UDP      Any       5353
	Windows Management Instrumentation (ASync-In)                   TCP      Any       Any
	Windows Management Instrumentation (WMI-In)                     TCP      Any       Any
	Core Networking - Teredo (UDP-Out)                              UDP      Any       Any
	Windows Management Instrumentation (WMI-Out)                    TCP      Any       Any
	Core Networking - Group Policy (LSASS-Out)                      TCP      Any       Any
	Core Networking - IPHTTPS (TCP-Out)                             TCP      Any       IPHTTPSOut
	Core Networking - Internet Group Management Protocol (IGMP-Out) 2        Any       Any
	Core Networking - DNS (UDP-Out)                                 UDP      Any       53
	Remote Desktop - Shadow (TCP-In)                                TCP      Any       Any
	AllJoyn Router (UDP-Out)                                        UDP      Any       Any
	AllJoyn Router (TCP-Out)                                        TCP      Any       Any
	AllJoyn Router (UDP-In)                                         UDP      Any       Any
	Connected User Experiences and Telemetry                        TCP      Any       443
	mDNS (UDP-Out)                                                  UDP      Any       5353
	Core Networking - IPv6 (IPv6-Out)                               41       Any       Any
	Core Networking - Internet Group Management Protocol (IGMP-In)  2        Any       Any
	Core Networking - Group Policy (NP-Out)                         TCP      Any       445
	Core Networking - Group Policy (TCP-Out)                        TCP      Any       Any
	File and Printer Sharing (NB-Session-Out)                       TCP      Any       139
	File and Printer Sharing (LLMNR-UDP-Out)                        UDP      Any       5355
	File and Printer Sharing (SMB-Out)                              TCP      Any       445
	File and Printer Sharing (NB-Datagram-Out)                      UDP      Any       138
	File and Printer Sharing (NB-Name-Out)                          UDP      Any       137
	Active Directory Domain Controller (UDP-Out)                    UDP      Any       Any
	Active Directory Domain Controller (TCP-Out)                    TCP      Any       Any
	Active Directory Web Services (TCP-Out)                         TCP      Any       Any
	All Outgoing (TCP)                                              TCP      Any       Any
	All Outgoing (UDP)                                              UDP      Any       Any
	File and Printer Sharing (Restrictive) (SMB-Out)                TCP      Any       445
	File and Printer Sharing (Restrictive) (LLMNR-UDP-Out)          UDP      Any       5355



	Port IPHTTPSIn:

	DisplayName                        Protocol LocalPort RemotePort
	-----------                        -------- --------- ----------
	Core Networking - IPHTTPS (TCP-In) TCP      IPHTTPSIn Any



	Port RPC:

	DisplayName                                                                Protocol LocalPort RemotePort
	-----------                                                                -------- --------- ----------
	Core Networking - Packet Too Big (ICMPv6-Out)                              ICMPv6   RPC       Any
	Core Networking - Parameter Problem (ICMPv6-Out)                           ICMPv6   RPC       Any
	Core Networking - Router Advertisement (ICMPv6-In)                         ICMPv6   RPC       Any
	Core Networking - Destination Unreachable Fragmentation Needed (ICMPv4-In) ICMPv4   RPC       Any
	Core Networking - Router Solicitation (ICMPv6-Out)                         ICMPv6   RPC       Any
	Core Networking - Neighbor Discovery Solicitation (ICMPv6-In)              ICMPv6   RPC       Any
	Core Networking - Multicast Listener Report v2 (ICMPv6-Out)                ICMPv6   RPC       Any
	Core Networking - Multicast Listener Done (ICMPv6-Out)                     ICMPv6   RPC       Any
	Core Networking - Router Solicitation (ICMPv6-In)                          ICMPv6   RPC       Any
	Core Networking - Router Advertisement (ICMPv6-Out)                        ICMPv6   RPC       Any
	Core Networking - Neighbor Discovery Advertisement (ICMPv6-In)             ICMPv6   RPC       Any
	Core Networking - Parameter Problem (ICMPv6-In)                            ICMPv6   RPC       Any
	Core Networking - Neighbor Discovery Advertisement (ICMPv6-Out)            ICMPv6   RPC       Any
	Core Networking - Multicast Listener Query (ICMPv6-Out)                    ICMPv6   RPC       Any
	Core Networking - Time Exceeded (ICMPv6-Out)                               ICMPv6   RPC       Any
	Core Networking - Multicast Listener Report (ICMPv6-Out)                   ICMPv6   RPC       Any
	Core Networking - Packet Too Big (ICMPv6-In)                               ICMPv6   RPC       Any
	Core Networking - Multicast Listener Done (ICMPv6-In)                      ICMPv6   RPC       Any
	Core Networking - Destination Unreachable (ICMPv6-In)                      ICMPv6   RPC       Any
	Core Networking - Neighbor Discovery Solicitation (ICMPv6-Out)             ICMPv6   RPC       Any
	Core Networking - Multicast Listener Report v2 (ICMPv6-In)                 ICMPv6   RPC       Any
	Core Networking - Multicast Listener Query (ICMPv6-In)                     ICMPv6   RPC       Any
	Core Networking - Multicast Listener Report (ICMPv6-In)                    ICMPv6   RPC       Any
	Core Networking - Time Exceeded (ICMPv6-In)                                ICMPv6   RPC       Any
	File and Printer Sharing (Echo Request - ICMPv4-Out)                       ICMPv4   RPC       Any
	File and Printer Sharing (Spooler Service Worker - RPC)                    TCP      RPC       Any
	File and Printer Sharing (Echo Request - ICMPv4-In)                        ICMPv4   RPC       Any
	File and Printer Sharing (Spooler Service - RPC)                           TCP      RPC       Any
	File and Printer Sharing (Echo Request - ICMPv6-Out)                       ICMPv6   RPC       Any
	File and Printer Sharing (Echo Request - ICMPv6-In)                        ICMPv6   RPC       Any
	Microsoft Key Distribution Service (RPC)                                   TCP      RPC       Any
	DFS Management (TCP-In)                                                    TCP      RPC       Any
	DFS Management (WMI-In)                                                    TCP      RPC       Any
	DFS Replication (RPC-In)                                                   TCP      RPC       Any
	Active Directory Domain Controller (RPC)                                   TCP      RPC       Any
	File Replication (RPC)                                                     TCP      RPC       Any
	Active Directory Domain Controller -  Echo Request (ICMPv4-Out)            ICMPv4   RPC       Any
	Active Directory Domain Controller -  Echo Request (ICMPv6-Out)            ICMPv6   RPC       Any
	Active Directory Domain Controller -  Echo Request (ICMPv4-In)             ICMPv4   RPC       Any
	Active Directory Domain Controller -  Echo Request (ICMPv6-In)             ICMPv6   RPC       Any
	RPC (TCP, Incoming)                                                        TCP      RPC       Any
	File and Printer Sharing (Restrictive) (Spooler Service - RPC)             TCP      RPC       Any
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv6-Out)         ICMPv6   RPC       Any
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv6-In)          ICMPv6   RPC       Any
	File and Printer Sharing (Restrictive) (Spooler Service Worker - RPC)      TCP      RPC       Any
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv4-Out)         ICMPv4   RPC       Any
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv4-In)          ICMPv4   RPC       Any
	File Server Remote Management (WMI-In)                                     TCP      RPC       Any



	Port RPCEPMap:

	DisplayName                                                          Protocol LocalPort RemotePort
	-----------                                                          -------- --------- ----------
	File and Printer Sharing (Spooler Service - RPC-EPMAP)               TCP      RPCEPMap  Any
	DFS Replication (RPC-EPMAP)                                          TCP      RPCEPMap  Any
	File Replication (RPC-EPMAP)                                         TCP      RPCEPMap  Any
	Active Directory Domain Controller (RPC-EPMAP)                       TCP      RPCEPMap  Any
	Microsoft Key Distribution Service (RPC EPMAP)                       TCP      RPCEPMap  Any
	RPC Endpoint Mapper (TCP, Incoming)                                  TCP      RPCEPMap  Any
	File and Printer Sharing (Restrictive) (Spooler Service - RPC-EPMAP) TCP      RPCEPMap  Any



	Port Teredo:

	DisplayName                       Protocol LocalPort RemotePort
	-----------                       -------- --------- ----------
	Core Networking - Teredo (UDP-In) UDP      Teredo    Any
	
### After a breif scan I have identified some ports that remain open that could be a possible threat to security:
	
	- Port 9955 - this is for smart devices and IoT. Definitley dont need for a domain controller!
	- Ports 5353 and 5355, these are mDNS and LLMNR, similar to APIPA but its when DNS resolution fails  it broadcasts name queries locally
	- I will be going on to configure SSH(22) and WinRM(5985) later however for the puporses of the excersize ill block them temporarily.

### Ill go on to disable them Now

	Disable-NetFirewallRule -DisplayName "AllJoyn Router*"
	Disable-NetFirewallRule -DisplayName "*LLMNR*"
	Disable-NetFirewallRule -DisplayName "*mDNS*"
	Disable-NetFirewallRule -DisplayName "*SSH*"
	Disable-NetFirewallRule -DisplayName "Windows Remote Management*"

### Veirification

	Get-NetFirewallRule | Where-Object {
	$_.DisplayName -like "*AllJoyn*" 
	-or $_.DisplayName -like "*LLMNR*" 
	-or $_.DisplayName -like "*mDNS*" 
	-or $_.DisplayName -like "*SSH*" 
	-or $_.DisplayName -like "*Remote Management*"} 
	| Select-Object DisplayName, Enabled
	
	DisplayName                                              Enabled
	-----------                                              -------
	Network Discovery (LLMNR-UDP-In)                           False
	mDNS (UDP-Out)                                             False
	mDNS (UDP-Out)                                             False
	Windows Remote Management (HTTP-In)                        False
	Windows Defender Firewall Remote Management (RPC-EPMAP)    False
	Network Discovery (LLMNR-UDP-Out)                          False
	Windows Remote Management - Compatibility Mode (HTTP-In)   False
	AllJoyn Router (UDP-Out)                                   False
	mDNS (UDP-In)                                              False
	AllJoyn Router (TCP-Out)                                   False
	Windows Defender Firewall Remote Management (RPC)          False
	AllJoyn Router (TCP-In)                                    False
	AllJoyn Router (UDP-In)                                    False
	Windows Remote Management (HTTP-In)                        False
	mDNS (UDP-Out)                                             False
	mDNS (UDP-In)                                              False
	mDNS (UDP-In)                                              False
	OpenSSH SSH Server (sshd)                                  False
	File and Printer Sharing (Restrictive) (LLMNR-UDP-In)      False
	File and Printer Sharing (LLMNR-UDP-Out)                   False
	File and Printer Sharing (LLMNR-UDP-In)                    False
	File and Printer Sharing (Restrictive) (LLMNR-UDP-Out)     False
	File and Printer Sharing (Restrictive) (LLMNR-UDP-Out)     False
	File and Printer Sharing (Restrictive) (LLMNR-UDP-In)      False
	File Server Remote Management (DCOM-In)                     True
	File Server Remote Management (WMI-In)                      True
	File Server Remote Management (SMB-In)                      True

### The firewall hardening was successfully implemented! 

### The veirification confirms that 24 unnecessary firewall rules have been disables, significantly reducting our network attack surface, while maintaining  essential domain controller functionality!

### Next ill move onto protocoll hardening, this is the process of disabling vulnerable protocol versions (same door, better locks) ill go over:

	- SMBv1	
	- LM/NTLMv1
	- SMB signing
	- LDAP signing
	- NetBIOS

### SMBv1 is a file sharing protocol from 1984 that contains critical vulnerabilities exploited by ransomware like WannaCry. Since we only need modern SMBv2/SMBv3 for domain operations, I'll disable this legacy protocol to eliminate a major attack vector.

	Disable-WindowsOptionalFeature 
	-Online 
	-FeatureName SMB1Protocol 
	-NoRestart
	
### Then verify 

	Get-WindowsOptionalFeature -Online -FeatureName SMB1Protocol | Select-Object FeatureName, State

	FeatureName     State
	-----------     -----
	SMB1Protocol Disabled
	
### Next ill check the LM/NTLmv1  current authenticaton level

	Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" 
	| Select-Object LmCompatibilityLevel

### The authentication level check returned a blank result, indicating the system is using default settings (likely weak Level 0 or 1). I'll enforce Level 3 security to eliminate decades of weak authentication vulnerabilities and force all domain communications to use modern, secure NTLMv2.
	
	Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" 
	-Name "LmCompatibilityLevel" -Value 3

### Then we verify
	
	Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" 
	| Select-Object LmCompatibilityLevel

	LmCompatibilityLevel
	--------------------
					   3
					   
### Next ill check SMB current status to see if we have signing enabled

	
	PS C:\Users\Administrator> Get-SmbServerConfiguration | Select-Object RequireSecuritySignature
	>>

	RequireSecuritySignature
	------------------------
						True


	PS C:\Users\Administrator> Get-SmbClientConfiguration | Select-Object RequireSecuritySignature
	>>

	RequireSecuritySignature
	------------------------
						True

### All good with SMB signing, now Ill look at NetBIOS configs. NetBIOS is like the old school DNS before the introduction of a central directory

	PS C:\Users\Administrator> Get-WmiObject -Class Win32_NetworkAdapterConfiguration | Where-Object {$_.TcpipNetbiosOptions -ne $null} | Select-Object Description, TcpipNetbiosOptions
	>>

	Description                       TcpipNetbiosOptions
	-----------                       -------------------
	Microsoft Hyper-V Network Adapter                   0
	
### Now for LDAP signing and channel binding - this ensures all directory communications are secure and tamper-proof

	PS C:\Users\Administrator> Write-Host "Current LDAP Security Settings:" -ForegroundColor Yellow
	>> Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" -ErrorAction SilentlyContinue | Select-Object LDAPServerIntegrity, LdapEnforceChannelBinding # ErrorAction SilentlyContinue is to just ignore red error messages
	Current LDAP Security Settings:

	ldapserverintegrity LdapEnforceChannelBinding
	------------------- -------------------------
					  1

### The pre configuration check shows that LDAPServerIntegrity = 1 (Weak - "Sign if client supports it") and LdapEnforceChannelBinding = (Blank - Disabled - INSECURE) we should set both of these values to 2 (Require signing)

	PS C:\Users\Administrator> Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" -Name "LDAPServerIntegrity" -Value 2
	>>
	PS C:\Users\Administrator> Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" -Name "LdapEnforceChannelBinding" -Value 2
	>>
	PS C:\Users\Administrator> Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" | Select-Object LDAPServerIntegrity, LdapEnforceChannelBinding

	ldapserverintegrity LdapEnforceChannelBinding
	------------------- -------------------------
					  2                         2

### The netbios setting is set to 0, this means that DHCP gets to decided wether to use it or not. Ill take control and make it quiet by setting it to 2 on all netowrk adaptors.

	Get-WmiObject -Class Win32_NetworkAdapterConfiguration   # Gets all network adapter configs
	| Where-Object 
	{$_.TcpipNetbiosOptions -ne $null} 						 # Filter this network adapters NetBIOS setting thats not emptyy
	| ForEach-Object {
    $_.SetTcpipNetbios(2)
    Write-Host "Disabled NetBIOS on: $($_.Description)"      # Print confirmation 
}



	PS C:\Users\Administrator> Get-WmiObject -Class Win32_NetworkAdapterConfiguration | Where-Object {$_.TcpipNetbiosOptions -ne $null} | ForEach-Object {
	>>     $_.SetTcpipNetbios(2)
	>>     Write-Host "Disabled NetBIOS on: $($_.Description)"
	>> }


	__GENUS          : 2
	__CLASS          : __PARAMETERS
	__SUPERCLASS     :
	__DYNASTY        : __PARAMETERS
	__RELPATH        :
	__PROPERTY_COUNT : 1
	__DERIVATION     : {}
	__SERVER         :
	__NAMESPACE      :
	__PATH           :
	ReturnValue      : 0
	PSComputerName   :

	Disabled NetBIOS on: Microsoft Hyper-V Network Adapter
	
### The NetBIOS setting was configured to use DHCP defaults (value 0), leaving the protocol potentially enabled. Since we have a fully functional DNS infrastructure, I disabled NetBIOS across all network adapters to reduce network broadcast noise and eliminate an outdated attack vector.

### With protocol security established, I'll now implement access controls to restrict who can interact with critical services. First I'll verify RDP permissions by checking the actual security policy:

	PS C:\Users\Administrator> Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" | Select-Object *
	>>


	AllowRemoteRPC                 : 1
	DelayConMgrTimeout             : 0
	DeleteTempDirsOnExit           : 1
	fDenyTSConnections             : 0									# 0 = RDP Enabled. normal for a managed DC
	fSingleSessionPerUser          : 1									# Users can only have one RDP session at a time
	NotificationTimeOut            : 0
	PerSessionTempDir              : 1
	ProductVersion                 : 5.1
	RCDependentServices            : {CertPropSvc, SessionEnv}
	SessionDirectoryActive         : 0
	SessionDirectoryCLSID          : {005a9c68-e216-4b27-8f59-b336829b3868}
	SessionDirectoryExCLSID        : {ec98d957-48ad-436d-90be-bc291f42709c}
	SessionDirectoryExposeServerIP : 1
	SnapshotMonitors               : 1
	StartRCM                       : 0
	TSUserEnabled                  : 0									# Remote Desktop Users group is disabled
	InstanceID                     : 519a36b5-4626-4785-9433-2883611
	GlassSessionId                 : 1
	LastRemoteLogonTime            : 2025-11-17T11:59:14.120Z
	PSPath                         : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server
	PSParentPath                   : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control
	PSChildName                    : Terminal Server
	PSDrive                        : HKLM
	PSProvider                     : Microsoft.PowerShell.Core\Registry

### RDP is enabled (fDenyTSConnections: 0) but the Remote Desktop Users group is disabled (TSUserEnabled: 0), confirming access is restricted to administrative accounts only. Single session enforcement (fSingleSessionPerUser: 1) provides additional security.

### Now ill disable and then check network discovery. (stop advertising on network)
	
	PS C:\Users\Administrator> Get-NetFirewallRule -DisplayName "*Network Discovery*" | Disable-NetFirewallRule
	>>
	PS C:\Users\Administrator> Get-NetFirewallRule -DisplayName "*Network Discovery*" | Select-Object DisplayName, Enabled, Direction, Action | Sort-Object DisplayName
	
	DisplayName                              Enabled Direction Action
	-----------                              ------- --------- ------
	Network Discovery (LLMNR-UDP-In)           False   Inbound  Allow
	Network Discovery (LLMNR-UDP-Out)          False  Outbound  Allow
	Network Discovery (NB-Datagram-In)         False   Inbound  Allow
	Network Discovery (NB-Datagram-Out)        False  Outbound  Allow
	Network Discovery (NB-Name-In)             False   Inbound  Allow
	Network Discovery (NB-Name-Out)            False  Outbound  Allow
	Network Discovery (Pub WSD-Out)            False  Outbound  Allow
	Network Discovery (Pub-WSD-In)             False   Inbound  Allow
	Network Discovery (SSDP-In)                False   Inbound  Allow
	Network Discovery (SSDP-Out)               False  Outbound  Allow
	Network Discovery (UPnPHost-Out)           False  Outbound  Allow
	Network Discovery (UPnP-In)                False   Inbound  Allow
	Network Discovery (UPnP-Out)               False  Outbound  Allow
	Network Discovery (WSD Events-In)          False   Inbound  Allow
	Network Discovery (WSD Events-Out)         False  Outbound  Allow
	Network Discovery (WSD EventsSecure-In)    False   Inbound  Allow
	Network Discovery (WSD EventsSecure-Out)   False  Outbound  Allow
	Network Discovery (WSD-In)                 False   Inbound  Allow
	Network Discovery (WSD-Out)                False  Outbound  Allow

### ICMP (ping) protocols can be used by attackers to discover active hosts on the network. I'll block inbound ICMP to prevent network scanning while maintaining essential outbound connectivity.

### First Ill assess the current situation

PS C:\Users\Administrator> Get-NetFirewallRule -DisplayName "*ICMP*" | Select-Object DisplayName, Enabled
>>

	DisplayName                                                                Enabled
	-----------                                                                -------
	Core Networking - Packet Too Big (ICMPv6-Out)                                 True
	Core Networking - Parameter Problem (ICMPv6-Out)                              True
	Core Networking - Router Advertisement (ICMPv6-In)                            True
	Core Networking - Destination Unreachable Fragmentation Needed (ICMPv4-In)    True
	Core Networking - Router Solicitation (ICMPv6-Out)                            True
	Core Networking - Neighbor Discovery Solicitation (ICMPv6-In)                 True
	Core Networking - Multicast Listener Report v2 (ICMPv6-Out)                   True
	Core Networking Diagnostics - ICMP Echo Request (ICMPv6-Out)                 False
	Core Networking Diagnostics - ICMP Echo Request (ICMPv4-Out)                 False
	Core Networking - Multicast Listener Done (ICMPv6-Out)                        True
	Core Networking - Router Solicitation (ICMPv6-In)                             True
	Core Networking - Router Advertisement (ICMPv6-Out)                           True
	Core Networking - Neighbor Discovery Advertisement (ICMPv6-In)                True
	Core Networking - Parameter Problem (ICMPv6-In)                               True
	Core Networking - Neighbor Discovery Advertisement (ICMPv6-Out)               True
	Core Networking - Multicast Listener Query (ICMPv6-Out)                       True
	Core Networking Diagnostics - ICMP Echo Request (ICMPv6-In)                  False
	Core Networking - Time Exceeded (ICMPv6-Out)                                  True
	Virtual Machine Monitoring (Echo Request - ICMPv4-In)                        False
	Virtual Machine Monitoring (Echo Request - ICMPv6-In)                        False
	Core Networking - Multicast Listener Report (ICMPv6-Out)                      True
	Core Networking - Packet Too Big (ICMPv6-In)                                  True
	Core Networking - Multicast Listener Done (ICMPv6-In)                         True
	Core Networking - Destination Unreachable (ICMPv6-In)                         True
	Core Networking Diagnostics - ICMP Echo Request (ICMPv4-Out)                 False
	Core Networking - Neighbor Discovery Solicitation (ICMPv6-Out)                True
	Core Networking Diagnostics - ICMP Echo Request (ICMPv6-Out)                 False
	Core Networking - Multicast Listener Report v2 (ICMPv6-In)                    True
	Core Networking - Multicast Listener Query (ICMPv6-In)                        True
	Core Networking - Multicast Listener Report (ICMPv6-In)                       True
	Core Networking Diagnostics - ICMP Echo Request (ICMPv4-In)                  False
	Core Networking - Time Exceeded (ICMPv6-In)                                   True
	Core Networking Diagnostics - ICMP Echo Request (ICMPv4-In)                  False
	Core Networking Diagnostics - ICMP Echo Request (ICMPv6-In)                  False
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv4-In)            False
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv4-Out)           False
	File and Printer Sharing (Echo Request - ICMPv4-Out)                          True
	File and Printer Sharing (Echo Request - ICMPv4-In)                           True
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv6-In)            False
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv6-Out)           False
	File and Printer Sharing (Echo Request - ICMPv6-Out)                          True
	File and Printer Sharing (Echo Request - ICMPv6-In)                           True
	Active Directory Domain Controller -  Echo Request (ICMPv4-Out)               True
	Active Directory Domain Controller -  Echo Request (ICMPv6-Out)               True
	Active Directory Domain Controller -  Echo Request (ICMPv4-In)                True
	Active Directory Domain Controller -  Echo Request (ICMPv6-In)                True
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv6-Out)            True
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv6-In)             True
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv4-Out)            True
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv4-In)             True

### Now Ill implement changes and check

	PS C:\Users\Administrator> Get-NetFirewallRule -DisplayName "*ICMP*" | Where-Object {$_.Direction -eq "Inbound"} | Disable-NetFirewallRule
	>>
	PS C:\Users\Administrator> Get-NetFirewallRule -DisplayName "*ICMP*" | Where-Object {$_.Direction -eq "Inbound"} | Select-Object DisplayName, Enabled

	DisplayName                                                                Enabled
	-----------                                                                -------
	Core Networking - Router Advertisement (ICMPv6-In)                           False
	Core Networking - Destination Unreachable Fragmentation Needed (ICMPv4-In)   False
	Core Networking - Neighbor Discovery Solicitation (ICMPv6-In)                False
	Core Networking - Router Solicitation (ICMPv6-In)                            False
	Core Networking - Neighbor Discovery Advertisement (ICMPv6-In)               False
	Core Networking - Parameter Problem (ICMPv6-In)                              False
	Core Networking Diagnostics - ICMP Echo Request (ICMPv6-In)                  False
	Virtual Machine Monitoring (Echo Request - ICMPv4-In)                        False
	Virtual Machine Monitoring (Echo Request - ICMPv6-In)                        False
	Core Networking - Packet Too Big (ICMPv6-In)                                 False
	Core Networking - Multicast Listener Done (ICMPv6-In)                        False
	Core Networking - Destination Unreachable (ICMPv6-In)                        False
	Core Networking - Multicast Listener Report v2 (ICMPv6-In)                   False
	Core Networking - Multicast Listener Query (ICMPv6-In)                       False
	Core Networking - Multicast Listener Report (ICMPv6-In)                      False
	Core Networking Diagnostics - ICMP Echo Request (ICMPv4-In)                  False
	Core Networking - Time Exceeded (ICMPv6-In)                                  False
	Core Networking Diagnostics - ICMP Echo Request (ICMPv4-In)                  False
	Core Networking Diagnostics - ICMP Echo Request (ICMPv6-In)                  False
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv4-In)            False
	File and Printer Sharing (Echo Request - ICMPv4-In)                          False
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv6-In)            False
	File and Printer Sharing (Echo Request - ICMPv6-In)                          False
	Active Directory Domain Controller -  Echo Request (ICMPv4-In)               False
	Active Directory Domain Controller -  Echo Request (ICMPv6-In)               False
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv6-In)            False
	File and Printer Sharing (Restrictive) (Echo Request - ICMPv4-In)            False



