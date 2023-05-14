If a user has sudo permissions to run `pip download`, you can use this exploit to escalate privileges with it:
https://embracethered.com/blog/posts/2022/python-package-manager-install-and-download-vulnerability/

In the RunCommand() function you can set whatever command you want to be ran here.

`os.system("chmod u+s /bin/bash")` will set SUID bit on /bin/bash, which can then be used to root
