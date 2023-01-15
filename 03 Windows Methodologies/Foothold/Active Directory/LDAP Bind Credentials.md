Lightweight Directory Access Protocol (LDAP) is a method of AD authentication that applications and services can use. LDAP is very common in third-party/custom software that authenticates against AD (GitHub, Jenkins, Printers, VPNs, etc.). An application using LDAP has a pair of AD credentials, one of which can be used to query the LDAP server, and the other can be used to verify an AD user credentials.

## Authenticated Attack Methodology

Assuming you are authenticated (and have the correct privileges) on a server running the LDAP application, the attack may be as simple as reading the LDAP AD Credentials from the software configuration files.

## LDAP Pass-back Attack

LDAP Pass-back attacks are very common against network devices, and are useful when there is already a foothold into the network. This attack is usually possible when a network device or application exposes a settings page with LDAP configuration. For example, a printer may expose a settings page that allows the user to enter an IP address for the LDAP server. This IP address can be changed to a rouge LDAP server (the attacker's machine). This will then cause the printer to send the credential pair to the attacker.

This attack is easily performed with an application `slapd`. It is not included in Kali by default but can be installed with `apt`. It will likely respond with an error, say `No` when asked to retry.

The rouge LDAP server can then be set up with `sudo dpkg-reconfigure -p low slapd`. On the first screen `Omit OpenLDAP server configuration?`, answer `No`. For the DNS domain name and the Organisation Name, answer with the victim domain address. For the password, provide a memorable password. Answer `No` when asked about removing the database on purge. The server is now setup, but it is not configured to be 'Rouge'. For this to happen, a configuration file is required to lower the security settings:

`./olcSaslSecProps.ldif`
```
#olcSaslSecProps.ldif
dn: cn=config
replace: olcSaslSecProps
olcSaslSecProps: noanonymous,minssf=0,passcred
```

Apply the `ldif` file using the following command (Start the `slapd` service first):

```
sudo ldapmodify -Y EXTERNAL -H ldapi:// -f ./olcSaslSecProps.ldif && sudo service slapd restart
```

Now the LDAP server is configured to listen for incoming requests and downgrade them, the following command can now be used to sniff the credentials:

`sudo tcpdump -SX -i [interface] tcp port 389`

With `tcpdump` running, trigger the settings change. The username and password should appear somewhere in the dump.