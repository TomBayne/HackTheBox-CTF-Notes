`ldapsearch` can be used to check variables on users (Account Lockout Policy, for example).

```
ldapsearch -D 'domainName\user' -w 'password' -H "ldap://10.10.10.192" -b "dc=domainname,dc=local" -s sub "*" | grep lockoutThreshold
```

This command will show all LDAP variables for the user, and then use `grep` to filter for the `lockoutThreshold` variable.

If `lockoutThreshold: 0` then there is no account lockout policy. This means that a password spray attack is possible.
