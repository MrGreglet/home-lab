# Active Directory User Security Group Assignments

## Understanding the Structure

## Before I begin assigning groups, let's clarify how everything fits together:

### Organizational Units (OUs) - Where users LIVE

	- Each user is placed in ONE OU based on their role
	- This is like their "home address" in Active Directory
	- Used for organization and Group Policy application

### Security Groups - What users can ACCESS  

	- Users can be members of MULTIPLE security groups
	- This determines their permissions and access rights
	- Like having multiple "keys" to different resources

### Visual Example:

	User: Alice Smith
		
		- OU Location: Employees OU (her home)
		- Security Groups:
			- Sales-Team (department access)
			- File-Server-Access (file permissions)
			- VPN-Users (remote access)

### First lets check a few key groups to ensure there are no members, as expected.

	Get-ADGroupMember -Identity "IT-Admins" | Select-Object Name, SamAccountName
	
	Get-ADGroupMember -Identity "Sales-Team" | Select-Object Name, SamAccountName
	
	Get-ADGroupMember -Identity "File-Server-Access" | Select-Object Name, SamAccountName

### They all came back with nothing which is what we expected.

### The next step is to plan some logical group assingments.
	
	Now I'll define why we assign users to specific groups. I'll start with the user who 
	needs the most permissions and work my way down to regular employees and service accounts. This approach follows the Principle of Least Privilege. 
	This means giving each user only the access they absolutely need for their job function. I'll document the reasoning behind each group assignment 
	to create a clear security model that's both functional and secure.

## Carol Davis - Systems Administrator (Highest Privileges)

### Why She Needs Maximum Access:

	- Manages entire domain - servers, AD, security policies
	- Responsible for everything - needs ability to fix any issue  
	- Top-level support - ultimate escalation point

## Carols complete group membership:

### Administrative Groups:

	Add-ADGroupMember -Identity "IT-Admins" -Members "cdavis"              # Full domain control

### Department Groups:

	Add-ADGroupMember -Identity "IT-Staff" -Members "cdavis"               # IT department resources

### Resource Groups:

	Add-ADGroupMember -Identity "File-Server-Access" -Members "cdavis"     # Access all files
	Add-ADGroupMember -Identity "Printer-Users" -Members "cdavis"          # Can print anywhere  
	Add-ADGroupMember -Identity "VPN-Users" -Members "cdavis"              # Remote access always

### What she DOESN'T get:
	
	- NOT in "Help-Desk" - too senior for frontline support
	- NOT in "Network-Admins" - Frank handles network gear
	- NOT in department groups like "Sales-Team" or "HR-Team"
	
### Lets check the groups carol is in with 

	Get-ADPrincipalGroupMembership -Identity "cdavis" | Select-Object Name # Get-ADPrincipalGroupMembership gets all groups a user belongs to
	
### Verification

	Name                GroupScope GroupCategory
	----                ---------- -------------
	Domain Users            Global      Security
	IT-Admins               Global      Security
	IT-Staff                Global      Security
	File-Server-Access DomainLocal      Security
	Printer-Users      DomainLocal      Security
	VPN-Users          DomainLocal      Security

### Confirms groups Carol is in
	
### Carol gets high privileges but only in her specific domain (systems administration), not blanket access to everything. This is much more secure!

## Frank Thompson - Network Administrator (Specialized Privileges)

### Frank's Role vs Carol's Role:

	- Frank: Manages network infrastructure (routers, switches, firewalls, VPN)
	- Carol: Manages systems infrastructure (servers, AD, applications)
	- Different tools, different permissions

### Frank's Targeted Group Membership:

### Administrative Groups:

	Add-ADGroupMember -Identity "Network-Admins" -Members "fthompson"      # Network devices
	Add-ADGroupMember -Identity "Help-Desk" -Members "fthompson"           # User support

### Department Groups:

	Add-ADGroupMember -Identity "IT-Staff" -Members "fthompson"            # IT department

### Resource Groups:

	Add-ADGroupMember -Identity "VPN-Users" -Members "fthompson"           # Manages VPN access
	Add-ADGroupMember -Identity "File-Server-Access" -Members "fthompson"  # Network shares
	Add-ADGroupMember -Identity "Printer-Users" -Members "fthompson"       # Network printers

### What Frank DOESN'T get:

	- NOT in "IT-Admins" - doesn't need domain-wide control
	- Not in other irrelevant departments
	
### Security Benefit:
	
	-If Frank's account is compromised, the attacker gets network access but not domain control. This creates a clear separation of duties.
	Frank handles the network, Carol handles the systems.
	
### Verifying the groups Frank is in
	
		Name                GroupScope GroupCategory
	----                ---------- -------------
	Domain Users            Global      Security
	Help-Desk               Global      Security
	Network-Admins          Global      Security
	IT-Staff                Global      Security
	File-Server-Access DomainLocal      Security
	Printer-Users      DomainLocal      Security
	VPN-Users          DomainLocal      Security
	
### Next ill add regular Employees - Bulk Assignment

### Added all three regular employees to basic resource groups at once:
	
### All employees get file and printer access

	$RegularEmployees = "asmith", "bjohnson", "dlee" # Creating a variable
	
	Add-ADGroupMember -Identity "File-Server-Access" -Members $RegularEmployees # Add to server file access 
	Add-ADGroupMember -Identity "Printer-Users" -Members $RegularEmployees	# Add to Printer-Users

### Department-specific assignments

	Add-ADGroupMember -Identity "Sales-Team" -Members "asmith"
	Add-ADGroupMember -Identity "HR-Team" -Members "bjohnson"

### Sales gets VPN for travel

	Add-ADGroupMember -Identity "VPN-Users" -Members "asmith"
	
### Now ill create a command to check all three users group memberships as I have defined them in  a variable.

	$RegularEmployees | ForEach-Object { # Takes $RegularEmployees and sends the list to ForEach-Object which loops through each user
 		$groups = Get-ADPrincipalGroupMembership -Identity $_ | Select-Object -ExpandProperty Name # Gets all groups from the current user, extracts group name as simple strings and stores the result in a variable.
		[PSCustomObject]@{
			User = $_
			Groups = ($groups -join ', ') #  Creates a custom object with two properties: "User" stores the current username, and "Groups" takes all the group names and joins them into a comma-separated string for clean display.
		}
	} | Format-Table -AutoSize # We know what this does :)

### The output show this:

	User     Groups
	----     ------
	asmith   Domain Users, Sales-Team, File-Server-Access, Printer-Users, VPN-Users
	bjohnson Domain Users, HR-Team, File-Server-Access, Printer-Users
	dlee     Domain Users, File-Server-Access, Printer-Users

### So the command was successfull! 

### Now to finish this phase, we just need to assign svc-backup to file server access group.
### Will possibly come to adding another group "Monitoring -Read only" or somthing later.
 
	Add-ADGroupMember -Identity "File-Server-Access" -Members "svc-backup"

### Now one last final verification to check all users in one go!

	$AllUsers = "cdavis", "fthompson", "asmith", "bjohnson", "dlee", "svc-backup", "svc-monitoring"
	$AllUsers | ForEach-Object {
		$groups = Get-ADPrincipalGroupMembership -Identity $_ | Select-Object -ExpandProperty Name
		[PSCustomObject]@{
			User = $_
			Groups = ($groups -join ', ')
		}
	} | Format-Table -AutoSize
	
### Output 

	User           Groups
	----           ------
	cdavis         Domain Users, IT-Admins, IT-Staff, File-Server-Access, Printer-Users, VPN-Users
	fthompson      Domain Users, Help-Desk, Network-Admins, IT-Staff, File-Server-Access, Printer-Users, VPN-Users
	asmith         Domain Users, Sales-Team, File-Server-Access, Printer-Users, VPN-Users
	bjohnson       Domain Users, HR-Team, File-Server-Access, Printer-Users
	dlee           Domain Users, File-Server-Access, Printer-Users
	svc-backup     Domain Users
	svc-monitoring Domain Users
	
## Verification Results ✅

	All regular employees now have correct group assignments:

	- Alice Smith: Sales-Team, File-Server-Access, Printer-Users, VPN-Users
	- Bob Johnson: HR-Team, File-Server-Access, Printer-Users  
	- David Lee: File-Server-Access, Printer-Users (basic access only)

	The security model is working as designed with proper role-based access control!
