# Harmfull BadUSB Scripts:
These scripts are capable of harm or illegal actions.
Please make sure only to run these scripts in a lab environment.

## Saved Browser Passwords:
PowerShell scripts to copy the local password db to your Flipper (using Mass Storage emulation) or a different USB storage drive.
Currently works on Google Chrome and Chrome based browsers.
Works in theory on Mozilla Firefox and Firefox based browsers, but should be tested in lab. (Note Firefox uses two password files; logins.json & key4.db)

Both Google Chrome and Mozilla Firefox based browsers (and possibly many others) use SQLite encoding, I find [SQLite DB](https://www.sqlite.org/index.html) to work wel in viewing these files. Just go to the Browse Data tab and scroll through the tables until you find your passwords in plain text (assuming decrypted).

Upcomming scripts might include; CMD alternatives, scripts for other browsers, and more.

May this serve as a reminder to **_NEVER_** store passwords **locally, in a predefined and static folder, without any access limitations.**
Even if the file is encrypted (using the local user's password).
