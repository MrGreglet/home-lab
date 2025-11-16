# Active Directory Organisational Unit Structure

## Planning the OU design

### Before creating users and groups, I need to design a logical Organisational Unit Structure that follows best practices for security and management.

### OU Deisgn Principles:

	- **Security Boundaries**: Seperate administrative permissions
	- **Delegation**: Different IT reas to manage different OU'security
	- **Group Policy**: Apply settings to logical groups
	- **Scalability**: Easy to expand as the organisation grows
	
### Proposed OU Structure:	

	greg.local/
		
		- User Accounts/ # All user objects
		
			- Employees/ # Employee Accounts
			- IT/ # IT Staff Accounts
			- Service Accounts/ # Application/service Accounts
		
		- Computer Accounts/
		
			- Servers/   # Member Servers
			- Workstations/ # User computers 
			- Domain controllers/  # (default - already exists)
		
		- Security Groups/
			
			- Administrative/ # Admin role groups 
			- Department/	  # Department-based groups
			- Resource/ 	  # Permission groups for resources

## Creating the OU structure

### First I need to create top level OU's. I'll do this using commands. 

	New-ADOrganizationalUnit 						# Creates a new OU
	-Name "User Accounts" 							# The display name of the OU
	-Path "DC=greg,DC=local" 						# Location in AD (domain root)
	-Description "Container for all user Accounts"	# Human readable Description
	-ProtectedFromAccidentalDeletion $true			# Prevents accidental deletion

### Ill do this for all 3 top level OU's, changing the name and desription as necessary.

### Then we can check the OU's we created with the command:
	
	Get-ADOrganizationalUnit  # Retrieves infor about OUs from AD
	-Filter * 			      # Applies a filter to the search * means everything or all items.
	| Format-Table
	Name,					  # Simple OU name
	DistinguishedName 		  # Full AD path
	
 ### The command shows this is a great success!
		
		Name               DistinguishedName
	----               -----------------
	Domain Controllers OU=Domain Controllers,DC=greg,DC=local
	Computer Accounts  OU=Computer Accounts,DC=greg,DC=local
	User Accounts      OU=User Accounts,DC=greg,DC=local
	Security Groups    OU=Security Groups,DC=greg,DC=local
	
### Now Ill add sub OU's under user accounts for Employees, IT and Service Accounts to organise users by role and department.

### You can create all three subgroups with one command, here ill walk through it and explain it.

	"Employees", "IT", "Service Accounts"		# A list of what names to create
	| ForEach-Object {							# Loop through each name in the list
	New-ADOrganizationalUnit					# New OU
	-Name $_									# This is the current item in the loop, IT, Employees etc
	-Path "OU=User Accounts,DC=greg,DC=local"	# Location in AD, /User Accounts 
	-Description "User accounts sub-OU"			# Will be the description for all sub groups 
	-ProtectedFromAccidentalDeletion $true		# What it says on the tin
	}											# Close loop 

### And now we verify the sub groups have been added with:
	
	Get-ADOrganizationalUnit -Filter * | Format-Table Name, DistinguishedName -AutoSize
	
### Verification

		Name               DistinguishedName
	----               -----------------
	Domain Controllers OU=Domain Controllers,DC=greg,DC=local
	Computer Accounts  OU=Computer Accounts,DC=greg,DC=local
	User Accounts      OU=User Accounts,DC=greg,DC=local
	Security Groups    OU=Security Groups,DC=greg,DC=local
	Employees          OU=Employees,OU=User Accounts,DC=greg,DC=local			# We can see here that Employees, IT, and Service Accounts are now members of two OU's
	IT                 OU=IT,OU=User Accounts,DC=greg,DC=local
	Service Accounts   OU=Service Accounts,OU=User Accounts,DC=greg,DC=local		

### Ill also use the same command to create the sub groups in Computer Accounts and Security Groups: 
	
		Name               DistinguishedName
	----               -----------------
	Domain Controllers OU=Domain Controllers,DC=greg,DC=local
	Computer Accounts  OU=Computer Accounts,DC=greg,DC=local
	User Accounts      OU=User Accounts,DC=greg,DC=local
	Security Groups    OU=Security Groups,DC=greg,DC=local
	Employees          OU=Employees,OU=User Accounts,DC=greg,DC=local
	IT                 OU=IT,OU=User Accounts,DC=greg,DC=local
	Service Accounts   OU=Service Accounts,OU=User Accounts,DC=greg,DC=local
	Servers            OU=Servers,OU=Computer Accounts,DC=greg,DC=local
	Workstations       OU=Workstations,OU=Computer Accounts,DC=greg,DC=local
	Administrative     OU=Administrative,OU=Security Groups,DC=greg,DC=local
	Department         OU=Department,OU=Security Groups,DC=greg,DC=local
	Resource           OU=Resource,OU=Security Groups,DC=greg,DC=local


## This concludes me setting up the AD OU Structure!!