# Initial VM set up and Windows server configurations

	First lets download windows server .iso

	In hyperV I have created a new virtual machine, giving it 4gb RAM and 40GB drive storage. Upon turning it on a and connecting, im 
	told the boot loader has failed, no boot image was found on the network adaper and the boot loader didnt install an operating system.
	I ejected the Virtual CD with the ISO and re mounted it. Its also a common issue when secure boot is activated when trying to install
	an operating system. Toggled off and the installation works. 

### After changing the Administrator password ill set a static IP address. First Ill check the current IP with
	
	Get-NetIPAddress 

### Im given a lot of information however scrolling through I find the relevant info. 

	IPAddress         : 192.168.1.xxx      #Current IP 
	InterfaceAlias    : Ethernet		   #Netowrk adaptor name
	PrefixLength      : 24				   #24 bit subnet mask (255.255.255.0)
	PrefixOrigin      : Dhcp			   #Confirms dynamicallty assigned 

### Ill now begin to configure this using the info gathered using: 

	New-NetIPAddress 
	
### This creates a new IP address config. you need at least 3 paramenters, being: 

	-InterfaceAlias #Specifies which entwork adaptor																				
	-IPAddress 		#Defines the static IP 
	-PrefixLength 	#Defines the local network size

### Ill also define the DefaultGateway to access the internet

	-DefaultGateway #Router IP for internet
	
###(Ill need to check my home network settings to find the DefaultGateway.)

### So my whole command is 
	
	New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "192.168.1.xxx" -PrefixLength 24 -DefaultGateway "192.168.1.xxx"

### Im now shown this information

	IPAddress         : 192.168.1.xxx
	InterfaceIndex    : 6 
	InterfaceAlias    : Ethernet 
	AddressFamily     : IPv4
	Type              : Unicast 
	PrefixLength      : 24
	PrefixOrigin      : Manual
	SuffixOrigin      : Manual 
	AddressState      : Tentative
	ValidLifetime     :
	PreferredLifetime :
	SkipAsSource      : False
	PolicyStore       : ActiveStore

	IPAddress         : 192.168.1.xxx
	InterfaceIndex    : 6
	InterfaceAlias    : Ethernet
	AddressFamily     : IPv4
	Type              : Unicast
	PrefixLength      : 24
	PrefixOrigin      : Manual
	SuffixOrigin      : Manual
	AddressState      : Invalid
	ValidLifetime     :
	PreferredLifetime :
	SkipAsSource      : False
	PolicyStore       : PersistentStore
	
### This shows windows in a state of transferring my IP settings from the previous to the new. To check this has all worked ill wait and then check the configuraiton again. 

### I'll use some targetting and formatting commands to show the information I need cleanly, by piping it into Format-Table 

### My input will be
	
	Get-NetIPAddress -InterfaceAlias "Ethernet" | Select-Object IPAddress, PrefixOrigin, SuffixOrigin, AddressState, PolicyStore

### Verifcation
	
	IPAddress    : 192.168.1.xxx
	PrefixOrigin : Manual  			# Shows IP is statically asssigned
	SuffixOrigin : Manual  
	AddressState : Preferred		# and activley in use
	PolicyStore  : ActiveStore

### After confirming the static IP has been set its now time to check the default gateway. Ill use these commands.

	Get-NetRoute #Gets all routing table entries
	Where-Object 							# Search/Filter tool
	{$_.DestinationPrefix -eq "0.0.0.0/0"}  # $_.DestinationPrefix (this route's destination), -eq (=), 0.0.0.0/0 (everywhere else)
	Format-Table 
	DestinationPrefix, NextHop, RouteMetric # Parameters to filter into table

### Whole line

	Get-NetRoute | Where-Object {$_.DestinationPrefix -eq "0.0.0.0/0"} | Format-Table DestinationPrefix, NextHop, RouteMetric
	
### Output

	DestinationPrefix NextHop       RouteMetric
	----------------- -------       -----------
	0.0.0.0/0         192.168.1.xxx         256

### This confirms the default gateway has been set.

### Now I think its time we rename that computer.

	Rename-Computer -NewName "DC01" -Restart

### This renames the computer to a more professional looking DC01 (Domain Controller) and restarts to apply changes.

### Now ill enable RDP service

	Set-ItemPropery 												 # Changes a setting/value in the windows registry
	-Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" 	 # Location where the RDP setting lives
	-Name "fDenyTSConnections" 										 # The specific setting we need to change to allow
	-Value 0														 # This is a 1(true) or 0 (false) 
	
### Then enable the RDP firewall rule

	Enable-NetFirwallRule 			# Turns on a firewall rule 
	-DisplayGroup "Remote Desktop"  # Applies to all rules in the "Remote Desktop" group (TCP, UDP etc)
	
### Now ill verify if the remote desktop service is runnning
	
	Get-service 			# Checks the status of windows services
	-Name "TermService" 	# Looks specifically for the "Terminal Services" (TermService = actual serice name for remote desktop)

### Confrim	

	Status   Name               DisplayName
	------   ----               -----------
	Running  TermService        Remote Desktop Services

### Then ill Check which RDP firewall rules are currently active

	Get-NetFirewallRule							# Gets all firewall rules
	-DisplayGroup "Remote Desktop" 				# Filters to RDP rules
	| Where-Object {$_.Enabled -eq "True"}		# Filters again to show only enabled ($_.Enabled) rules (-eq "True")
	| Format-Table									
	DisplayName									# Parameter for human readable firewall rule
	Enabled										# Shows if the rule is on or off
	Direction									# Show if rule applies to incoming or outgoing traffic
	Action										# Shows what the rule does with the traffic
	-Autosize									# Automatically sizes the table columns to fit the content
	
### Output 

	DisplayName                         Enabled Direction Action
	-----------                         ------- --------- ------
	Remote Desktop - Shadow (TCP-In)       True   Inbound  Allow
	Remote Desktop - User Mode (TCP-In)    True   Inbound  Allow
	Remote Desktop - User Mode (UDP-In)    True   Inbound  Allow

### Now would be a good time to check for any updates before going on to install AD

	Get-WindowsUpdate

### It seems like we need an update module to be able to be able to use that command. Ill use sconfig to get updates instead.

### After waiting a painstakingly long time ill check hyperV to see CPU usage. 

### 0% CPU usage for a good 5 mins, safe to say we have crashed. Shut down the VM and rebooted. Applied update procedure through sconfig, restarted VM after prompt.

### Now its time to install ADDS (Active Directory Domain Services)
	
	Install-WindowsFeature 	# Installs a windows feature or server role
	AD-Domain-Services 		# This is the ADDS role
	-IncludeManagementTools	# Actual management tools for ADDS, PowerShell modules for AD commands, Remote server admin tools, And console tools for GUI (Maybe ill look into using a GUI at a later stage)
	
### Verifcation

	Success Restart Needed Exit Code      Feature Result
	------- -------------- ---------      --------------
	True    No             Success        {Active Directory Domain Services, Group P...
	
### Now ADDS is installed we need to make the server a Domain Controller, which will automatically create the domain we choose.

	Install-ADDSForest     			# Forest is the top level container for everything in AD
	-DomainName "greg.local"		# Defining the domain name
	-InstallDNS -Force				# Suppresses confirmation prompts when intalling DNS server role.
	
### Windows then goes on to ask and confirm my password before it begins the promotion process and reboots.

### Now it has rebooted it asks for username and password. The windows server  has been promoted to a fully functional Domain Controller for the greg.local domain!

	================================================================================
					Welcome to Windows Server 2025 Standard Evaluation
	================================================================================

		1)  Domain/workgroup:                   Domain: greg.local
		2)  Computer name:                      DC01
		3)  Add local administrator
		4)  Remote management:                  Enabled

		5)  Update setting:                     Download only
		6)  Install updates
		7)  Remote desktop:                     Enabled (more secure clients)

		8)  Network settings
		9)  Date and time
		10) Diagnostic data setting:            Required
		
	
### We can also check this with the commands

	Get-ADDomain 			# Shows domain common details	
	Get-Service DNS			# Check if DNS is Running
	syteminfo				# Shows Domain status
	

## Windows server configuration COMPLETED!

11/16/2025 @ 10.30 p.m.
	
	



