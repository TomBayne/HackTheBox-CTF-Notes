Authentication Relay attacks focus on exploiting the Server Message Block (SMB) protocol. This protocol is commonly found throughout Active Directory domains, in things such as file shares, network printing, and remote administration. Several vulnerabilities are present in older versions of SMB due to the weak NetNTLM challenge-response system.

- Since NetNTLM challenges can be intercepted over the network, they can be sniffed by an attacker who can then attempt to bruteforce the related password. Whilst this is possible, it is significantly slower than attempting to crack the NTLM hash itself.

- An attacker can stage a man-in-the-middle attack, relaying the challenge-response between the client and server. This will provide the attacker access to an authenticated session with the target server.

## Capture-only Attack

In order to carry out this attack, [Responder](https://github.com/lgandx/Responder) can be used. This tool essentially attempts to 'poison' some types of NetNTLM challenges by responding to LLMNR requests before the actual target system can, making the originating system assume the rouge machine is the intended target. On a real network, only the intended target machine would respond to the LLMNR request. By responding to this request, it forces the target to connect to the rouge server, rather than the server it was looking for. From there, the originating system will attempt to authenticate with the rouge system.

Responder can be ran using `sudo responder -I [interface name]`.

On a real LAN, it is recommended to leave Responder running for as long as possible, as sometimes it can take a while to get a hit. On a CTF/simulation network, this is usually setup to take less time.

Once Responder intercepts an event, it output information about the authentication request. This information should include a NTLM hash which can be cracked later with Hashcat (mode 5600) or John

## Relay Attack

The above attack can be taken further by relaying the authentication attempt rather than just capturing it, this will provide access to whatever resource the victim machine was attempting to access (and anything further the account may have access to.). The issue with this type of attack is that it requires quite a few variables to be in the attacker's favour:

- SMB signing must be disabled or not enforced. If it is enabled, the attacker will not be able to forge the digital signature on the request, and therefore will fail to relay.
- The relayed account must have the permissions that are required by the attacker or it is not useful.

**!! TO DO: ADD STEPS FOR RELAY ATTACK HERE**