You can bruteforce subdomains using a wordlist and `gobuster`. This will connect to the IP by bruteforcing the subdomain in the `Host:` header of the HTTP request. This **will not work** if the subdomain is on a different IP address.
`gobuster vhost -u example.com -w ./all.txt -t 50`

- **-u**: specifies the base URL to bruteforce.
- **-w**: specifies the wordlist to use for the attack
- **-t**: specifies the number of threads (simultanous attacks)

This requires a worldlist. On some poorly formatted words from the world list, you may find that you get `Status: 400`. These can be ignored, they are not real results.

You can use a very similar command to bruteforce VHosts. This is when the same IP hosts multiple domains (i.e. `example.com` and `website.com` are both on the IP address `10.10.10.1`)

`gobuster vhost -u 10.10.10.1 -w ./all.txt -t 50`

If there is a DNS server available (i.e. you are not attacking a server on a private network), you can use the `dns` option in gobuster. This will send the requests to the DNS server rather than the web server.

`gobuster dns -u example.com -w ./all.txt -t 50`