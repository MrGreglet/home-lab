# AD user management

## Planning user creation

	Now that the OU sturcture and security groups are complete, I need to populate the domain with user accounts organised by department and role.
	Ill make 7 user accounts individually then explore bulk creation methods for scalability.

### User Design principles:

	- Organizational Placement: Users in correct OUs (Employees, IT, Service Accounts)
	- Attribute Completeness: Fill in department, manager, profile details
	- Security Basics: Secure passwords, account policies
	- Group Membership: Add users to appropriate security groups

### Planned user accounts:

	- Employees OU/
		
		- Alice Smith (Sales)
		- Bob Johnson (Marketing)
		- David Lee (Operations)
		
	- IT OU/ 
	
		- Carol Davies (Sytems Admin)
		- Frank Thompson (Network Admin)
		
	- Service Accounts OU/ 				
	
		- svc-backup (Backup service)	# "svc-" prefix identifies service accounts vs human users for security and management.
		- svc-monitoring (Monitoring Service)
		
## Employee OU Users

### User 1: Alice Smith (Sales)

	New-ADUser `
		-Name "Alice Smith" # Display name in AD
		-GivenName "Alice" 
		-Surname "Smith" 
		-SamAccountName "asmith" # Legacy authentication system, Security Account Manager
		-UserPrincipalName "asmith@greg.local" # Modern login format
		-Path "OU=Employees,OU=User Accounts,DC=greg,DC=local" 
		-AccountPassword (ConvertTo-SecureString "xxxxxxxx" -AsPlainText -Force) 
		-Enabled $true # By default when users are created they are disabled, so we ensure it is enabed.
		-Department "Sales" 
		-Title "Sales Representative" 
		-EmailAddress "asmith@company.com"


## IPORTANT NOTE:

### (ConvertTo-SecureString "xxxxxxxx" -AsPlainText -Force)

	ConvertTo-SecureString converts plain text to an encrypted SecureString object that Active Directory can use
	
	-AsPlainText Explanation:
	
	PowerShell assumes passwords are already encrypted for security. `-AsPlainText` explicitly tells PowerShell:
	"I know this is plain text and potentially insecure, but I'm intentionally using it for this lab environment."

### Production Alternative:
	
	In real environments, you'd never use plain text passwords, you would use:
	
	- Secure password vaults
	- Encrypted credential files  
	- Interactive password prompts
	- Automated password generators
	 
	 NEVER STORE PASSWORDS IN SCRIPTS OR SOURCE CONTROL!!!

### So now when we execute the command we should have added the first user to to one of our OUs... Time to test!
	
	Get-ADUser # Gets user accounts from AD
	-Filter 
	* -Properties DistinguishedName # Include the distinguished name  property (full AD path)
	| Where-Object {$_.DistinguishedName -like "*Employees*"} # filters results using the DistinguishedName propety that contain Employees in the path
	| Format-Table 
	Name, 
	DistinguishedName
	
### and the results are in!

	Name        DistinguishedName
	----        -----------------
	Alice Smith CN=Alice Smith,OU=Employees,OU=User Accounts,DC=greg,DC=local
	
### Now ill add Bob, from marketing, using the same method as we did with Alice changing parameters where necessary.

### User 2: Bob Johnson (Marketing)

	New-ADUser 
		-Name "Bob Johnson" 
		-GivenName "Bob" 
		-Surname "Johnson" 
		-SamAccountName "bjohnson" 
		-UserPrincipalName "bjohnson@greg.local" 
		-Path "OU=Employees,OU=User Accounts,DC=greg,DC=local" 
		-AccountPassword (ConvertTo-SecureString "xxxxxxxx" -AsPlainText -Force) 
		-Enabled $true 
		-Department "Marketing" 
		-Title "Marketing Coordinator" 
		-EmailAddress "bjohnson@company.com"
		
### And the verification!

		Name        DistinguishedName
	----        -----------------
	Alice Smith CN=Alice Smith,OU=Employees,OU=User Accounts,DC=greg,DC=local
	Bob Johnson CN=Bob Johnson,OU=Employees,OU=User Accounts,DC=greg,DC=local
	
### User 3: David Lee (Operations)

	New-ADUser 
		-Name "David Lee" 
		-GivenName "David" 
		-Surname "Lee" 
		-SamAccountName "dlee" 
		-UserPrincipalName "dlee@greg.local" 
		-Path "OU=Employees,OU=User Accounts,DC=greg,DC=local" 
		-AccountPassword (ConvertTo-SecureString "xxxxxxxx" -AsPlainText -Force) 
		-Enabled $true 
		-Department "Operations" 
		-Title "Operations Analyst" 
		-EmailAddress "dlee@company.com"
		
### Final check

	Name        DistinguishedName
	----        -----------------
	Alice Smith CN=Alice Smith,OU=Employees,OU=User Accounts,DC=greg,DC=local
	Bob Johnson CN=Bob Johnson,OU=Employees,OU=User Accounts,DC=greg,DC=local
	David Lee   CN=David Lee,OU=Employees,OU=User Accounts,DC=greg,DC=local

### And thats the Employees OU fleshed out. Now time to move to my IT OU.

## IT OU users

### User 4: Carol Davis (Systems Administrator)


	New-ADUser 
		-Name "Carol Davis" 
		-GivenName "Carol" 
		-Surname "Davis" 
		-SamAccountName "cdavis" 
		-UserPrincipalName "cdavis@greg.local" 
		-Path "OU=IT,OU=User Accounts,DC=greg,DC=local" 
		-AccountPassword (ConvertTo-SecureString "xxxxxxxx" -AsPlainText -Force) 
		-Enabled $true 
		-Department "IT" 
		-Title "Systems Administrator" 
		-EmailAddress "cdavis@company.com"

### Ill use basicall the same command but changing the filter:

	Get-ADUser -Filter * -Properties DistinguishedName | Where-Object {$_.DistinguishedName -like "*OU=IT*"} | Format-Table Name, DistinguishedName

### verification

	Name        DistinguishedName
	----        -----------------
	Carol Davis CN=Carol Davis,OU=IT,OU=User Accounts,DC=greg,DC=local
	
	
### User 5: Frank Thompson (Network Administrator)

	New-ADUser 
		-Name "Frank Thompson" 
		-GivenName "Frank" 
		-Surname "Thompson" 
		-SamAccountName "fthompson" 
		-UserPrincipalName "fthompson@greg.local" 
		-Path "OU=IT,OU=User Accounts,DC=greg,DC=local" 
		-AccountPassword (ConvertTo-SecureString "xxxxxxxx" -AsPlainText -Force) 
		-Enabled $true 
		-Department "IT" 
		-Title "Network Administrator" 
		-EmailAddress "fthompson@company.com"
		
### After adding Frank we verify

	Name           DistinguishedName
	----           -----------------
	Carol Davis    CN=Carol Davis,OU=IT,OU=User Accounts,DC=greg,DC=local
	Frank Thompson CN=Frank Thompson,OU=IT,OU=User Accounts,DC=greg,DC=local

### Confirmed! Ill next move on to the service accounts OU for the non human accounts!

## Service Accounts OU

### What are Service Accounts?

	Service accounts are non-human accounts used by applications and services to perform automated tasks. They:
	
		- Run applications/services (backups, monitoring, databases)
		- Have no interactive login (typically disabled for GUI login)
		- Have specific permissions (only what the service needs)
		- Have the "svc-" prefix, which identifies them as service accounts

### Service Account Security Principles:

	- Least Privilege: Only grant permissions the service absolutely needs
	- No Interactive Login: Prevent direct human access
	- Regular Auditing: Monitor service account activity
	- Secure Passwords: Long, complex passwords that never expire

### Current Service Accounts:

	svc-backup - For backup software
	
		- Job: Lets backup programs access and read all files
		- Access: Can read files across the entire domain
		- Security: Cannot log in, only does backup tasks

	svc-monitoring - For monitoring tools

		- Job: Lets monitoring software check server health
		- Access: Can read system performance and status
		- Security: Read-only access, cannot make changes

### Why Keep Them Separate?

	- Security: Keep them away from human user accounts
	- Different Rules: They need different password settings
	- Easy Management: All service accounts in one place
	- Clear Tracking: Easy to spot in logs and reports
	
### Now that is understood, illl move on to creating the first service user:

### User 6: svc-backup (Backup Service Account)

	New-ADUser
		-Name "svc-backup" 
		-DisplayName "Backup Service Account" 
		-SamAccountName "svc-backup" 
		-UserPrincipalName "svc-backup@greg.local" 
		-Path "OU=Service Accounts,OU=User Accounts,DC=greg,DC=local" 
		-AccountPassword (ConvertTo-SecureString "xxxxxxxx" -AsPlainText -Force) 
		-Enabled $true
		-Description "Backup software service account - read access for backups" 
		-PasswordNeverExpires $true 
		-CannotChangePassword $true 

### Notice the differences compared to the previous user creations for human users:
		
	- -PasswordNeverExpires $true = Services can't handle password changes automatically
	- -CannotChangePassword $true = Prevents accidental changes, increases security
	- No personal attributes = No -GivenName, -Surname, -Department, -Title
	- Descriptive -DisplayName and -Description = Clear purpose documentation
	- No email address = Services don't need email communication
	
### I also had no errors come up from this so ill continue to making svc-monitoring


### Ill use the same command but changing name displayname, description and password. 

### Then i verify both of these have worked

		Name           DistinguishedName
	----           -----------------
	svc-backup     CN=svc-backup,OU=Service Accounts,OU=User Accounts,DC=greg,DC=local
	svc-monitoring CN=svc-monitoring,OU=Service Accounts,OU=User Accounts,DC=greg,DC=loca


# This concludes my work creating different types of users with AD!


