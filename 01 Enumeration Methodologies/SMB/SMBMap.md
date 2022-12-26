The `smbmap` command can be used to enumerate available shares as any user (or without credentials). For example, to enumerate the available shares as the 'guest' user (unauthenticated user), use the following command:

```
smbmap -H 10.10.10.192 -u guest
```

The `smbclient` command can then be used to work with any available share. This command will run `ls` on the `profiles$` share:

```
smbclient -N \\\\10.10.10.192\\profiles$ -c ls
```

___

To enumerate using known credentials, use the following command, adjusting username, password, and IP address as required:

```
smbmap -u victim -p s3cr3t -H 192.168.86.61
```

`smbclient` can then be used to work with any available SMB share:

```
smbclient -U "user" \\\\10.10.10.192\\profiles$ -c "ls" password
```