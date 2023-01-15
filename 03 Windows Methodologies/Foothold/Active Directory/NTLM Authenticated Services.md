NTLM Authenticated Services are (as the name suggests) services that authenticate using the NTLM protocol. This is a simple protocol which uses 'Challenge-Response' to verify credentials without sending the password or the password hash across the network. This is considered a legacy authentication protocol and Kerberos is most commonly used on systems newer than XP instead. Some services that use NTLM may be exposed on the network and can be exploited in many ways.

## Password Spraying

Password Spraying is the act of using a single password against a list of usernames (like the inverse of a bruteforce against one account). The idea is that a known password (i.e. a default 'Change Me' password) is used against a list of accounts (known or bruteforced). This method works particularly well on domains that do not enforce an account lockout policy. Tools like [[Hydra]] can be used to carry out these attacks, however they are also easy to manually script in Python if customisation of the attack is required.

