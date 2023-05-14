- Kernel Exploits?

**GTFOBins**
- `sudo -l`
- setuid? `find / -type f -perm -04000 -ls 2>/dev/null`
- Capabilities? `getcap -r /`

- Crontab - Scripts that are ran by root and can be edited by user?
- Exploit PATH variable
- `no_root_squash` NFS Shares - mount and create suid bit executable