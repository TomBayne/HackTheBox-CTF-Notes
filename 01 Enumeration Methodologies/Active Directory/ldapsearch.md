`ldapsearch` can be used to check variables on users (Account Lockout Policy, for example).

```
ldapsearch -D 'domainName\user' -w 'password' -H "ldap://10.10.10.192" -b "dc=domainname,dc=local" -s sub "*" | grep lockoutThreshold
```

This command will show all LDAP variables for the user.

An example use case can be found in the [[Blackfield]] write up, which uses this command alongside `grep` to filter for the `lockoutThreshold` variable.

As is the case in the above example - when `lockoutThreshold: 0` then there is no account lockout policy. This means that a password spray attack or dictionary attack is possible.
