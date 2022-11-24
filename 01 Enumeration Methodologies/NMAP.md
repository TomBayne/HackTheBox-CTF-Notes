Run nmap with service detection in order to discover what ports are open and what services are running on them. The command looks like this:

`sudo nmap [server address] -p- -sV -A`

`-p-` selects all ports (1-65535)
`-sV` runs service detection
`-A` detects OS and service
Running as `sudo` allows for SYN Stealth scanning, which is more reliable.

Other useful options:

`--script=ssl-heartbleed` runs Heartbleed detection, or another [script](https://nmap.org/nsedoc/scripts/) if specified.

`-Pn`: doesn't send ping probes. Allows port scanning even if ICMP is blocked.

**Interpreting Output:**
The resulting output from nmap should contain various information about the OS, and services running on it. Particularly useful information that should be noted is:

- Open [[01 Common Ports|ports]]
- Services running on those ports and their versions
- OS Version information
- Any other information that stands out as possibly useful.

For a more in-depth nmap scan, including UDP. Use the following command:
`sudo nmap 10.129.161.240 -p- -sV -A -Pn -T4 --min-rate 10000 -sU -sS`