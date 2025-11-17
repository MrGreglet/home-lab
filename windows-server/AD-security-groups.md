# Active Directory Security Groups

## Planning Security group structure

## Now that the OU structure is complete, I need to create security groups that will control permissions and access throughout the domain.

### Security group design principles:
	
	- Role Based Access Control (RBAC): Groups based on job functions
	- Least privilege: Only necessary permissions
	- Department Alignment: Match Organisational structure
	- Resource-Based: Groups for specific resource access

### Planned Security groups:

	- Administrative OU/
		
		- IT Admins
		- Help-Desk
		- Network Admins
	
	- Department OU/
		
		- IT Staff
		- Sales Team
		- HR Team
		
	- Resource OU/
		
		- Filer server access
		- Printer servers
		- VPN users
		
### Ill do this with very similar commands to how I did the basic OUs, this time adding extrra parameters for security based roles and permissions

### Ive just learnt about hastable arrays. Ill implement that here.

		@(
		#This is the open part of the command to create an array of hashtables, each group with a name AND Description
		@{Name="IT-Admins"; Description="IT administrators with full domain access"},
		@{Name="Help-Desk"; Description="Help desk staff with limited administrative access"},
		@{Name="Network-Admins"; Description="Network infrastructure administrators"}
	) | ForEach-Object {
		New-ADGroup 
			-Name $_.Name 
			-Path "OU=Administrative,OU=Security Groups,DC=greg,DC=local" 
			-GroupScope Global 
			-GroupCategory Security 
			-Description $_.Description
	}
		
		
### Understanding GroupScope and GroupCategory

	**GroupScope = Permission Reach**
	- `DomainLocal`: Can only assign permissions within this domain
	- `Global`: Can be used across the entire Active Directory forest (our choice)
	- `Universal`: Can be used across multiple forests (advanced)

	**GroupCategory = Group Purpose**  
	- `Security`: Used for assigning permissions and access rights (our choice)
	- `Distribution`: Used only for email distribution lists

	We use `Global` scope so admin groups can manage multiple domains, and `Security` category to actually grant permissions.

### Now time to verify this works, ill use a command that filters hyphons as the groups ive jsut made contain them.
	
	Get-ADGroup -Filter * | Where-Object {$_.Name -like "*-*"} | Sort-Object Name | Format-Table Name, GroupScope, GroupCategory -AutoSize

### Verification

		Name                                     GroupScope GroupCategory
	----                                     ---------- -------------
	Enterprise Read-only Domain Controllers   Universal      Security
	Help-Desk                                    Global      Security
	Hyper-V Administrators                  DomainLocal      Security
	IT-Admins                                    Global      Security
	Network-Admins                               Global      Security
	Pre-Windows 2000 Compatible Access      DomainLocal      Security
	Read-only Domain Controllers                 Global      Security
	
### Now ill use the same command for Department groups, however the group category and scope will be adjuusted so that IT staff have security category


	@(
		@{Name="IT-Staff"; Description="IT department staff"; Scope="Global"; Category="Security"},
		@{Name="Sales-Team"; Description="Sales department"; Scope="Global"; Category="Distribution"}, 
		@{Name="HR-Team"; Description="Human Resources"; Scope="Global"; Category="Distribution"}
	) | ForEach-Object {
		New-ADGroup 
			-Name $_.Name 
			-Path "OU=Department,OU=Security Groups,DC=greg,DC=local" 
			-GroupScope $_.Scope 
			-GroupCategory $_.Category 
			-Description $_.Description
	}
	
### Verify again with the command from earlier

	Name                                     GroupScope GroupCategory
	----                                     ---------- -------------
	Enterprise Read-only Domain Controllers   Universal      Security
	Help-Desk                                    Global      Security
	HR-Team                                      Global  Distribution
	Hyper-V Administrators                  DomainLocal      Security
	IT-Admins                                    Global      Security
	IT-Staff                                     Global      Security
	Network-Admins                               Global      Security
	Pre-Windows 2000 Compatible Access      DomainLocal      Security
	Read-only Domain Controllers                 Global      Security
	Sales-Team                                   Global  Distribution

### Now its time to create the Resource security groups, with an edited version of the command ive been using

	@(
		@{Name="File-Server-Access"; Description="Users with access to file servers"},
		@{Name="Printer-Users"; Description="Users with network printing access"},
		@{Name="VPN-Users"; Description="Users with remote VPN access"}
	) | ForEach-Object {
		New-ADGroup 
			-Name $_.Name 
			-Path "OU=Resource,OU=Security Groups,DC=greg,DC=local" 
			-GroupScope DomainLocal   # ← Resource permissions only!
			-GroupCategory Security 
			-Description $_.Description
	}

### Now again verify witb the Where_Object 

	Name                                     GroupScope GroupCategory
	----                                     ---------- -------------
	Enterprise Read-only Domain Controllers   Universal      Security
	File-Server-Access                      DomainLocal      Security
	Help-Desk                                    Global      Security
	HR-Team                                      Global  Distribution
	Hyper-V Administrators                  DomainLocal      Security
	IT-Admins                                    Global      Security
	IT-Staff                                     Global      Security
	Network-Admins                               Global      Security
	Pre-Windows 2000 Compatible Access      DomainLocal      Security
	Printer-Users                           DomainLocal      Security
	Read-only Domain Controllers                 Global      Security
	Sales-Team                                   Global  Distribution
	VPN-Users                               DomainLocal      Security

### This marks the completion of setting up all 9 security groups in my OU's structrue!	

### Key Learning:
	- Applied Role-Based Access Control principles
	- Used mixed GroupScope/GroupCategory based on security needs
	- Implemented bulk creation with hashtable arrays
	- Verified all groups with filtered PowerShell commands