# AD Group policies

## Planning Group Policiy Implementation
	
	Now that ive got users and security groups configured, Ill move onto GPO's (group policy objects) to enforce settings and security across the domain.

### What group policies do:
	
	- Automatically apply settings to users and computers
	- Enforce security rules company-wide
	- Control user desktop envrironments
	- Set password account policies
	
### My GPO strategy:

	1. Check what already exists - )See default policies)
	2. Create basic security policies - Password rules, Lockout settings etc
	3. Apply GPO's to OU's 
	4. Test it!
	
## Policies I plan to create:
	
### For security
	
	- workstation security policy - Basic security for all computers
	- User restrictions policy - Control what users can change
	- Server hanrdening policy - Extra security for servers

### For management:
	
	- IT Admin policy - Special settings for IT staff computers
	- Department policies - Specific settings for Sales, HR, etc.
	
	
## Step 1: checking current configuaration
	
	Lets see what GPO's are already in place. First we need to import the group policy tools. We can do this with:
	
		Import-Module GroupPoliciy
	
	Then we can use 
		
		Get-GPO -All | Format-Table DisplayName, Id, GpoStatus
		
### The result of that command shows us that:

	DisplayName                                GpoStatus
	-----------                                ---------
	Default Domain Policy             AllSettingsEnabled
	Default Domain Controllers Policy AllSettingsEnabled
	
### Default domain policy is the main policy that applies to EVERY computer and user in the domain. Its currently active and enforcing settings.

### Default domain controllers policy is a policy that provides extra protection specifically for Domain Controllers, and is applied only to them.

### Now I need to see what these policies are actually enforcing. Ill do this for Default domain policty first for important password and security settings.

### There are two ways we can do this, one for legacy systems and it only shows basic info or we can use an AD module.

	net accounts  # Legacy command works on any windows machine. Results below
	
		Force user logoff how long after time expires?:       Never  # This means that users can stay logged in indefinitely, even if their account expires!
		Minimum password age (days):                          1		 # Users must wait one day before changing passwords again
		Maximum password age (days):                          42	 # Forces password change every 42 days
		Minimum password length:                              7		 # Passwords must be at least 7 characters
		Length of password history maintained:                24	 # Remembers the last 24 passwords (prevents reuse)
		Lockout threshold:                                    Never	 # To account lockout after failed attempts
		Lockout duration (minutes):                           10	 # Length of lockout  )if enabled 10 minutes)
		Lockout observation window (minutes):                 10	 # Failed attempts are counted within 10 minutes
		Computer role:                                        PRIMARY # This is the main domain controlelr
		The command completed successfully.
	
	Get_ADDefaultDomainPasswordPolicy # AD module command, shows a more detailed summary.
	
		ComplexityEnabled           : True									# Passwords to require 3 of 4 charactyer types (uppercase, lowercase,numbers and symbols)
		DistinguishedName           : DC=greg,DC=local						# Shows this policy aaplied the the whole greg.local  domain
		LockoutDuration             : 00:10:00
		LockoutObservationWindow    : 00:10:00
		LockoutThreshold            : 0
		MaxPasswordAge              : 42.00:00:00
		MinPasswordAge              : 1.00:00:00
		MinPasswordLength           : 7
		objectClass                 : {domainDNS}							# Technical AD object type
		objectGuid                  : 1999d1ee-8df3-4e7f-b756-8c1f02289254	# Unique identifier for the domain greg.local in AD
		PasswordHistoryCount        : 24
		ReversibleEncryptionEnabled : False									# Are passwords stored able to with reversible hashing? (No, only one way)

### Analysing the setting we currently have for Default Domain Policy there are some key changes to make to harden the policy.

### High risk security gaps that I have identified are Account lockout settings and weak password length.

### Account lockout settings (LockoutThreshold: 0) are not enabled at all! This makes it easier for brute force attacks. To fix this illenable the setting after 5 attempts.

### Ive also identified the password length (MinPasswordLength) is a major security issue. Industry standards now suggest a length of at least 12 characters.

### Medium risk settings I can see are no forced logoff for expired accounts, means thats expired accounts can remain active and too short of a lockout duration.

### Ill set a reasonable forced logoff time and increase the lockout duration to 60 mins.

### A low risk improvement I could make would be to reduce the password age limit from 42 to 30 maxium.

### The command to do this is
	
	Set-ADDefaultDomainPasswordPolicy -Identity greg.local 
    -LockoutThreshold 5 									# Lock after 5 failed attempts
    -MinPasswordLength 12									# 12 character minimum
    -LockoutDuration "00:30:00" 							# 30 minute lockout
    -LockoutObservationWindow "00:30:00" 					# 30 minute observation window
    -MaxPasswordAge "30.00:00:00" 							# 30 day password expiry
    -ComplexityEnabled $true 								# Keep complexity enabled
    -ReversibleEncryptionEnabled $false						# Keep secure hashing
	
### Confirming the results with Get_ADDefaultDomainPasswordPolicy

	
	ComplexityEnabled           : True
	DistinguishedName           : DC=greg,DC=local
	LockoutDuration             : 00:30:00
	LockoutObservationWindow    : 00:30:00
	LockoutThreshold            : 5
	MaxPasswordAge              : 30.00:00:00
	MinPasswordAge              : 1.00:00:00
	MinPasswordLength           : 8
	objectClass                 : {domainDNS}
	objectGuid                  : 1999d1ee-8df3-4e7f-b756-8c1f02289254
	PasswordHistoryCount        : 24
	ReversibleEncryptionEnabled : False

### This confirms the settings change took place! Nowe to check the domain controllers group policiy, I already knwo from earlier it exists and is enabled

### but now I need to know where its applied and what settings it controls.

	Get-GPO -Name "Default Domain Controllers Policy" # Grabs the Default Domain controlelrs policy
	| Get-GPOReport -ReportType Xml 				  # What it does: Takes that GPO object and generates a detailed report in XML format
	| Select-String "SOMName"						  # The file is messy and contains a lot of info so were searching the a string with SOMname (Scope of management name) which tells us where the policy aaplies

### As a result I get the whole file but can see at the bottom where the domain controllet gpo is linked
	
		<SOMName>Domain Controllers</SOMName>				# Applied to - Domain Controllers
		<SOMPath>greg.local/Domain Controllers</SOMPath>	# Full path to the OU in AD
		<Enabled>true</Enabled>								# Policy enabled and enforcing
		<NoOverride>false</NoOverride>						# Other policies can ovveride this? no
	  </LinksTo>
	</GPO>
	
### Domain Controllers Policy - Conclusion

	- The policy is correctly linked to the Domain Controllers OU
	- Default settings are secure enough for our homelab
	- No immediate changes needed

## Now ill move on to creating custom GPO's! Ill continue with security as this is a priortity. Lets start with a Workstation Security Policy:

	New-GPO -Name "Workstation Security Policy"

### And then we will link it to the worksations OU:
	
	New-GPLink -Name "Workstation Security Policy" -Target "OU=Workstations,OU=Computer Accounts,DC=greg,DC=local"
	

	GpoId       : 23272593-de31-4d70-be6c-6bbb29e3354d
	DisplayName : Workstation Security Policy
	Enabled     : True
	Enforced    : False # This means it can be overridden
	Target      : OU=Workstations,OU=Computer Accounts,DC=greg,DC=local # This shows we have successfully linked it the the correct OU!
	Order       : 1 # This is the priority order, 1 means first in line, this policy applies before others. 1 First priority then 2 etc

### Ill now configure some settings and explain why I chose them

### 1. Set 15-minute screen saver timeout (forces auto-lock on idle timeout)

	Set-GPRegistryValue 					
	-Name "Workstation Security Policy"  	# Which GPO to modify
	-Key "HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Control Panel\Desktop" # Path
	-ValueName "ScreenSaveTimeOut"  # What setting to change
	-Type String # What type of data (text/number)
	-Value "900" # 900 seconds = 15 minutes

### 2. Require CTRL+ALT+DEL before login (prevents password grabbers, ie mimicking login screen)
 
	Set-GPRegistryValue 
	-Name "Workstation Security Policy" 
	-Key "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" # Win security settings location
	-ValueName "DisableCAD"  # Setting name: "Disable CTRL+ALT+DEL"
	-Type DWord        # Data type: Number (0 or 1)
	-Value 0 # 0 = FALSE (don't disable CTRL+ALT+DEL)

### 3. Disable guest account access (blocks anonymous network enumeration and reduces attack surface)

	Set-GPRegistryValue 
	-Name "Workstation Security Policy" 
	-Key "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\LanManServer\Parameters" 
	-ValueName "restrictnullsessaccess" # Restrict all null session (anonymous) access
	-Type DWord 
	-Value 1  # 1 = TRUE (restrict access)

### I forgot to add forced logoff settings, will force logoff when logon hours expire (prevents indefinitye access with expired accounts) then verification

	Set-GPRegistryValue "Workstation Security Policy" 
	-Key "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters" 
	-ValueName "ForceLogoffWhenHourExpire" # Forces logoff when account expires or outside permitted hours
	-Type DWord -Value 1 
	
	KeyPath     : SYSTEM\CurrentControlSet\Services\Netlogon\Parameters
	FullKeyPath : HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters
	Hive        : LocalMachine
	PolicyState : Set
	Value       : 1
	Type        : DWord
	ValueName   : ForceLogoffWhenHourExpire
	HasValue    : True

### Now my windows server will automaticall logoff when user accounts expire, prevent indefinite access with expired credentials and enforces account lifecycle management. I could also later configure permitted hours to automatically log users off but ill do this later.

## Now lets check these settings have been implemented;

### First lets confirm what screensaver/ timeout setting have been configured:

	Get-GPRegistryValue 
	-Name "Workstation Security Policy"
	-Key "HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Control Panel\Desktop" # Location for desktop/screensaver policies
	
	
	KeyPath     : Software\Policies\Microsoft\Windows\Control Panel\Desktop
	FullKeyPath : HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Control Panel\Desktop
	Hive        : CurrentUser
	PolicyState : Set
	Value       : 900
	Type        : String
	ValueName   : ScreenSaveTimeOut
	HasValue    : True

### Now lets check the CTRL-ALT-DEL setting I added:

	Get-GPRegistryValue 
	-Name "Workstation Security Policy" 
	-Key "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System"  # This is where CAD reqs, login restricitons, and other system security settings live
	
	KeyPath     : SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System
	FullKeyPath : HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System
	Hive        : LocalMachine
	PolicyState : Set
	Value       : 0
	Type        : DWord
	ValueName   : DisableCAD
	HasValue    : True
	
### And finally lets check that the anonymous access has been revooked:

	Get-GPRegistryValue
	-Name "Workstation Security Policy" 
	-Key "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\LanManServer\Parameters" # This is where guest access, anonymous connections and share permissions are controlled
	
	KeyPath     : SYSTEM\CurrentControlSet\Services\LanManServer\Parameters
	FullKeyPath : HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\LanManServer\Parameters
	Hive        : LocalMachine
	PolicyState : Set
	Value       : 1
	Type        : DWord
	ValueName   : restrictnullsessaccess
	HasValue    : True

### Now I have confirmed this custom GPO has been configured to my standards, ill move on to configuring a few more GPO's 

### They will be as follows; User Restriction, Server Hardening, and IT Admin policies.

## User restrictions Policy. This will control what regular users can and cannot do on their own workstations to prevent accidental system changes and improve security.

	New-GPO -Name "User Restrictions Policy" # Create a new GPO
	
	New-GPLink -Name "User Restrictions Policy" -Target "OU=User Accounts,DC=greg,DC=local" # Then link it to the OU User Accounts

### Now lets add a setting to prevent users fom accessing the contol Panel

	Set-GPRegistryValue
	-Name "User Restrictions Policy" 
	-Key "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" 
	-ValueName "NoControlPanel" # Setting we are configuring
	-Type DWord 
	-Value 1	

### I think another good measure would be to disable the command prompt and batch files (scripts that could be used to compromise security)

	
	Set-GPRegistryValue 
	-Name "User Restrictions Policy" 
	-Key "HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\System" 
	-ValueName "DisableCMD" 
	-Type DWord 
	-Value 2 # Command prompt AND batch files disabled
	
	

### Now its time to check that the commands have actually worked: 

	Get-GPRegistryValue 
	-Name "User Restrictions Policy" 
	-Key "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer"
	
	KeyPath     : Software\Microsoft\Windows\CurrentVersion\Policies\Explorer
	FullKeyPath : HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer
	Hive        : CurrentUser
	PolicyState : Set
	Value       : 1
	Type        : DWord
	ValueName   : NoControlPanel # No control panel setting
	HasValue    : True			 # Is on and active

	Get-GPRegistryValue 
	-Name "User Restrictions Policy" 
	-Key "HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\System"
	
	KeyPath     : Software\Policies\Microsoft\Windows\System
	FullKeyPath : HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\System
	Hive        : CurrentUser
	PolicyState : Set
	Value       : 2			 # CMD AND batch files disabled
	Type        : DWord
	ValueName   : DisableCMD # Disable command prompt
	HasValue    : True 		
	
# User Restrictions Policy - COMPLETE. With user workstation controls in place, I'll now implement server-specific security through a Server Hardening Policy.
	
# This will add extra security measures specifically to servers. Since they host critical servers and data, they need stronger protection than regular workstations.

### Ill implement:

	 - Larger event logs - servers need more logging capacity for security monitoring
	 - Enhanced auditing - Better tracking of who does what on servers
	 - Service restricitons - Disable unnecessary services to reduce attack surface

### Why This Matters:

	- Servers have different security requirements than workstations
	- Critical infrastructure needs extra protection layers
	- Enterprise environments always have separate server vs workstation policies

### The implementation: First create the GPO and then link is to the relevant OU's 

	New-GPO -Name "Server Hardening Policy"

	New-GPLink -Name "Server Hardening Policy" -Target "OU=Servers,OU=Computer Accounts,DC=greg,DC=local"

### Then Ill set maximum log size for applications:

	Set-GPRegistryValue 
	-Name "Server Hardening Policy" 
	-Key "HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\EventLog\Application" 
	-ValueName "MaxSize"  # Setting for maxium log file size
	-Type DWord 
	-Value 67108864		  # Log max file size to be set in bytes
	
### Then the same for the system event log:

	Set-GPRegistryValue 
	-Name "Server Hardening Policy" 
	-Key "HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\EventLog\System" 
	-ValueName "MaxSize" 
	-Type DWord 
	-Value 67108864
		
### Then also for security event log: 

	Set-GPRegistryValue 
	-Name "Server Hardening Policy" 
	-Key "HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\EventLog\Security" 
	-ValueName "MaxSize" 
	-Type DWord 
	-Value 67108864
	
### Then check for each:
	
	Get-GPRegistryValue 
	-Name "Server Hardening Policy" 
	-Key "HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\EventLog\Application"
	
	KeyPath     : SOFTWARE\Policies\Microsoft\Windows\EventLog\Application
	FullKeyPath : HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\EventLog\Application
	Hive        : LocalMachine
	PolicyState : Set
	Value       : 67108864
	Type        : DWord
	ValueName   : MaxSize
	HasValue    : True
	
	
	
	Get-GPRegistryValue 
	-Name "Server Hardening Policy" 
	-Key "HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\EventLog\System"
	
	KeyPath     : SOFTWARE\Policies\Microsoft\Windows\EventLog\System
	FullKeyPath : HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\EventLog\System
	Hive        : LocalMachine
	PolicyState : Set
	Value       : 67108864
	Type        : DWord
	ValueName   : MaxSize
	HasValue    : True
	
	
	
	Get-GPRegistryValue 
	-Name "Server Hardening Policy" 
	-Key "HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\EventLog\Security"
	
	KeyPath     : SOFTWARE\Policies\Microsoft\Windows\EventLog\Security
	FullKeyPath : HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\EventLog\Security
	Hive        : LocalMachine
	PolicyState : Set
	Value       : 67108864
	Type        : DWord
	ValueName   : MaxSize
	HasValue    : True

### That confirms the max file size change for the logs!

# Next up is enhanced auditing. 

	This is so I can track who does what on my servers. Default auditing is basic, we need detailed tracking of:
		
		- User logons/logoffs
		- File access attempts
		- Privilage use 
		- Policy changes
		- Account management
	
	Windows has these main audit categories:
		
		- Account Logon (When users authenticate to this server)
		- Logon/Logoff (When users access resources on this server)
		- Object Access (File/Folder Access)
		- Privilage use (When users use special rights)
		- Policy Change (When security policiesare modified
		- Account management (When user accounts are created/changed)

	Planning the audit settings	
		
		- Success and failure for critical events
		- More detailed tracking than workstations 
		- Focus on security related	activities

# User logons/offs
		
### Enabling account logon auditing. This tracks authentication attempts to the server.

	auditpol         					 # Windows audit policiy tool 
	/set 								 # Changing a setting 
	/subcategory:"Credential Validation" # Tracks when users prove their identity 
	/success:enable 					 # Log successful attempts 
	/failure:enable 					 # Log failed attempts

### Im then told by windows the command was successfully executed, but lets check with the command:
	
	auditpol 
	/get								 # Getting a setting
	/subcategory:"Credential Validation"

	System audit policy
	Category/Subcategory                      Setting
	Account Logon
		Credential Validation                   Success and Failure
			
### Now we have confirmed this, ill add the next layer - tracking user sessions on the Server (log ons). Then the test result to confirm.
	
	auditpol 
	/set 
	/subcategory:"Logon"  # Tracks when user sessions start on this server
	/success:enable 
	/failure:enable

	System audit policy
	Category/Subcategory                      Setting
	Logon/Logoff
		Logon                                   Success and Failure
		
### Successfully comfirmed, now ill start tracking when user sessions end on the server (log offs). Then ill test to confirm

	auditpol 
	/set 
	/subcategory:"Logoff" 		# Tracks when users logoff 
	/success:enable   			# Note that there is in failure:enable, becuase logoff cant fail

	System audit policy
	Category/Subcategory                      Setting
	Logon/Logoff
	  Logoff                                  Success

### Now ill begin tracking when accounts get locked out due to failed login attempts

	auditpol 
	/set 
	/subcategory:"Account Lockout" # Tracks when accounts are automatically locked
	/success:enable
	
	System audit policy 
	Category/Subcategory                      Setting
	Logon/Logoff
	  Account Lockout                         Success

## This is confirmed as configured and is important in security as it will alert to potential brute force attacks, shows which accounts are being targeted and is essential for security incedent response

# Now we will work on policy change auditing. This tracks when security policies are modified.

	auditpol 
	/set 
	/subcategory:"Policy Change" # Tracks modifications to security policies 
	/success:enable 
	/failure:enable

### Okay my guess for policy change was wrong, and after checking the for the correct subcategory name I discovered it is "Audit Policy Change" not simply "Policy Change"

	auditpol
	/set 
	/subcategory:"Audit Policy Change"  # Correct subcategory adjustment
	/success:enable 
	/failure:enable

### Then check:
	
	System audit policy
	Category/Subcategory                      Setting
	Policy Change
	  Audit Policy Change                     Success and Failure
	  
### So now we have completed logon/off and policy change auditing, time to move onto account management auditing. 

# Lets enable account management auditing, This tracks when user accounts are created, changed, or deleted.
	
	auditpol 
	/set 
	/subcategory:"User Account Management"  # Subcategory for user acct management
	/success:enable  # Who managed what user when
	/failure:enable	 # Who failed to manage what user when

### Then test

	System audit policy
	Category/Subcategory                      Setting
	Account Management
	  User Account Management                 Success and Failure
	  
### User account managment auditing successfully set up.

# Now for File System auditing, which tracks every access attempt to files and folders wether successful or not. There are two steps to this, To enable the audit policy and then to configure specific files/folders we want to target.

## Enable File System auditing and verify:

	auditpol 
	/set 
	/subcategory:"File System" # subcategory that controls file/folder access tracking
	/success:enable 
	/failure:enable
	
	System audit policy
	Category/Subcategory                      Setting
	Object Access
	  File System                             Success and Failure
	  
### Now we have file system auditing set up thats great, but its not going to do anything until I point it at the files/folders I want to audit. I am doing this in CLI only and from what im reading and hearing from people is that this is usually very complex in CLI and fer my level would be better to use GUI. Ill configure this later.

###  The last and final audit configuration I will do will be Privilage Use auditing. This tracks when users excersise special administrative rights such as changing the sytem time, backing up/restoring files, shutting down/ rebooting the system, loading device drivers etc

### First lets enable then check 
	
	auditpol 
	/set 
	/subcategory:"Sensitive Privilege Use"  # Tracks powerful admin rights usage
	/success:enable 
	/failure:enable

	System audit policy
	Category/Subcategory                      Setting
	Privilege Use
	  Sensitive Privilege Use                 Success and Failure
	  



### Now workstation and server security are established,  Ill create the final policy for IT administrative access:

	New-GPO -Name "IT Admin Policy"

	New-GPLink -Name "IT Admin Policy" -Target "OU=User Accounts,DC=greg,DC=local"
	
### Then enter the command to enable RDP access.
	
	Set-GPRegistryValue 
	-Name "IT Admin Policy" 
	-Key "HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" 
	-ValueName "fDenyTSConnections" 
	-Type DWord -Value 0

### Now that the policy has been set up and linked to the OU user accounts.

# I have just realsied my mistake. 

	I have given all User Accounts RDP this is a security risk as it gives all users the ability to enable RDP and potentially bypass security controls,
	and create unauthorised remote access points. Ill change the setting to deny OU User Accounts by default, this hardens the system by auto denying any RDP attempt. 
	I will then create a new GPO for Workstations and Computer OU's then ill use sucurity filtering to only apply it to IT-Admins group. This will implement a layered approach
	rather than trying to create complex solutiions.
	
### Fix current IT Admin policy

	Set-GPRegistryValue 
	-Name "IT Admin Policy"
	-Key "HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" 
	-ValueName "fDenyTSConnections" 
	-Type DWord 
	-Value 1

### Then ill create and link the new Secure RDP policy

	New-GPO -Name "IT-RDP-Enabled"
	
	New-GPLink -Name "IT-RDP-Enabled" 
	-Target "OU=Workstations,OU=Computer Accounts,DC=greg,DC=local"

### Then enable RDP in this controlled GPO:
	
	Set-GPRegistryValue 
	-Name "IT-RDP-Enabled" 
	-Key "HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" 
	-ValueName "fDenyTSConnections" 
	-Type DWord 
	-Value 0
	

## lets break down the next few steps for better understanding: 

###  First we will install the module for editing group policies:

	Get-Command -Module GroupPolicy

### Now we will allow computers to read the GPO 

	Set-GPPermission 
	-Name "IT-RDP-Enabled" 
	-PermissionLevel GpoRead # Read only perms
	-TargetName "Authenticated Users"
	-TargetType Group
	
### Now only allow IT-Admins to apply the GPO settings!

	Set-GPPermission 
	-Name "IT-RDP-Enabled" 
	-PermissionLevel GpoApply # Apply perms
	-TargetName "IT-Admins" 
	-TargetType Group
	
### Then of course its time to check to see if it has been configured properley

	Get-GPPermission -Name "IT-RDP-Enabled" -All
	
	Trustee     : IT-Admins
	TrusteeType : Group
	Permission  : GpoApply
	Inherited   : False

	Trustee     : Authenticated Users
	TrusteeType : WellKnownGroup
	Permission  : GpoApply               # For some reason or another the command to change to read only permissions hasnt worked! Were looking for GpoRead here.
	Inherited   : False

	Trustee     : Domain Admins
	TrusteeType : Group
	Permission  : GpoEditDeleteModifySecurity
	Inherited   : False

	Trustee     : Enterprise Admins
	TrusteeType : Group
	Permission  : GpoEditDeleteModifySecurity
	Inherited   : False

	Trustee     : ENTERPRISE DOMAIN CONTROLLERS
	TrusteeType : WellKnownGroup
	Permission  : GpoRead
	Inherited   : False

	Trustee     : SYSTEM
	TrusteeType : WellKnownGroup
	Permission  : GpoEditDeleteModifySecurity
	Inherited   : False

### Maybe if I try removing authenticated users group completely, then add them back with GpoRead...

	Set-GPPermission 
	-Name "IT-RDP-Enabled" 
	-PermissionLevel None 
	-TargetName "Authenticated Users" 
	-TargetType Group
	
	Set-GPPermission 
	-Name "IT-RDP-Enabled" 
	-PermissionLevel GpoRead                 # GRANT read-only access
	-TargetName "Authenticated Users" 
	-TargetType Group
	
### When trying to set the permission, powershell is asking for GUID parameter instead of using the -Name parameter.

### Ill set it in a variable so powershell can get it easily

	$GPO = Get-GPO -Name "IT-RDP-Enabled"
	# then
	$GUID = $GPO.Id
### Then ill use the GUID to setr permissions

		Set-GPPermission 
		-Guid $GUID 
		-PermissionLevel GpoRead 
		-TargetName "Authenticated Users" 
		-TargetType Group
	
### Lets test again

	GGet-GPPermission 
	-Name "IT-RDP-Enabled" 
	-All
	
	Trustee     : IT-Admins


	Trustee     : Authenticated Users
	TrusteeType : WellKnownGroup
	Permission  : GpoApply
	Inherited   : False

### How unfortunate. It still hasnt worked. Let's troubleshoot this until its fixed. ill get the full command syntax:

	Get-Command Set-GPPermission -Syntax

### So i can see we hve the parameter -Replace. Maybe this is the missing piece! Ill try a new command using -Replace

	Set-GPPermission 
	-Guid $GUID 
	-PermissionLevel "GpoRead" 
	-TargetName "Authenticated Users" 
	-TargetType Group 
	-Replace  # New parameter added

### Then we test... 

	Get-GPPermission -Name "IT-RDP-Enabled" -TargetName "Authenticated Users" -TargetType Group

	Trustee     : Authenticated Users
	TrusteeType : WellKnownGroup
	Permission  : GpoRead
	Inherited   : False
	
### Great success! We were just missing one parameter.  The architecture for secure RDP is now complete!


### I forgot to add specific department policies. Ill configure one now to demonstrate, and possibly come back to this later to expand.

### First ill need to make the OU's, Sales, HR, and Finance. These will go under the employees OU. Ill add the updated structre to AD-OU-Structure at the bottom.

	New-ADOrganizationalUnit -Name "Sales" -Path "OU=Employees,OU=User Accounts,DC=greg,DC=local"
	New-ADOrganizationalUnit -Name "HR" -Path "OU=Employees,OU=User Accounts,DC=greg,DC=local"  
	New-ADOrganizationalUnit -Name "Finance" -Path "OU=Employees,OU=User Accounts,DC=greg,DC=local"

### Then ill create the GPO and link it

	New-GPO -Name "Sales Department Policy"
	
	New-GPLink -Name "Sales Department Policy" 
	-Target "OU=Sales,OU=Employees,OU=User Accounts,DC=greg,DC=local"

### Now I can add the department specific policy, which will set the screensaver timeout to 20 minutes. 

	Set-GPRegistryValue "Sales Department Policy" 
	-Key "HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Control Panel\Desktop" 
	-ValueName "ScreenSaveTimeOut" 
	-Type String 
	-Value "1200"
	
### So now we have a Workstation Security Policy to set the screensave timeout for 900 seconds that applies to all computers, and then we have the Sales Department policiy screensaver timeout set to 1200 seconds. These are conflicting GPO's. By default, user policies override computer policies for user specific settings like screensaver timeouts. 

### Now sales should get 20 mins, while everyone else gets 15 mins screensaver timeout. Lets check that with this inheritence command

	Get-GPInheritance # Shows how group policies flow down through the OU structure.
	-Target "OU=Sales,OU=Employees,OU=User Accounts,DC=greg,DC=local" # Targets the Sales OU and see what policies apply there
	
	Name                  : sales
	ContainerType         : OU
	Path                  : ou=sales,ou=employees,ou=user accounts,dc=greg,dc=local
	GpoInheritanceBlocked : No # Means policies can flow down from parent OU's, if this said yes, Sales OU would ony get its own policies.
	GpoLinks              : {Sales Department Policy} # What policies are attatched to this OU
	InheritedGpoLinks     : {Sales Department Policy, User Restrictions Policy, IT Admin Policy, Default Domain Policy} # All the policies that apply here

### Conclusion

	Security Challenge Successfully Resolved:
	Through systematic troubleshooting, I identified and fixed the security filtering issue. The missing `-Replace` parameter was critical for forcing the permission change from GpoApply to GpoRead.

	Final Security Model Achieved:
	
	- IT-RDP-Enabled GPO: RDP enabled but restricted via security filtering
	- IT-Admins: GpoApply - IT staff computers CAN apply RDP settings
	- Authenticated Users: GpoRead - All other computers can read but NOT apply RDP
	- IT Admin Policy: RDP disabled by default (secure baseline)

	Enterprise Security Principles Demonstrated:
	
	1.	Default Deny: RDP disabled everywhere by default
	2. Controlled Exceptions: RDP enabled only for authorized IT staff
	3. Layered Security: Multiple GPOs with specific purposes
	4. Least Privilege: Granular permissions using security filtering
	5. Defense in Depth: Complementary policies working together

	Technical Breakthrough:
	
	The discovery that `-Replace` parameter was required highlights the importance of:
	
	- Understanding exact command syntax
	- Systematic troubleshooting methodology
	- Documenting technical challenges and solutions

	Phase 1 GPO Framework Status: COMPLETED
	The enterprise-grade Group Policy infrastructure is now fully implemented and secured!