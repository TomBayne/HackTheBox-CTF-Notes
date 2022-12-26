
After extracting a list of usernames from a server, it is a good idea to check for accounts with Kerberos Pre-Authentication disabled. If there are any found, it is possible to extract the password hash, which can then be bruteforced to uncover the user's password.

`GetNPUsers.py` from [Impacket](https://github.com/SecureAuthCorp/impacket) can be used to run this attack. The following command will iterate across the `users.txt` username list and attempt ASREPRoasting automatically.

`python3 GetNPUsers.py blackfield.local/ -no-pass -usersfile /home/kali/users.txt -dc-ip 10.10.10.192`

This will output mostly errors due to accounts not being found, or having pre-authentication enabled. However, one or more accounts may output a hash if they have pre-authentication disabled. This hash can then be bruteforced using John or Hashcat.