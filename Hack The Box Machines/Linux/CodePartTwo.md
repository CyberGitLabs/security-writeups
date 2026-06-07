# CodePartTwo

## Information

CodePartTwo is an HTB Linux machine classified as Easy.

Notable Topics:
  - Exposed Vulnerable application
  - Sudo permission on a GTFOBin

## 1 Service Enumeration

I always feel safer if I can do an overall port scan. So if the process is fast enough I try it:

![diagram](../../images/CodePartTwo/CodePartTwo_nmapscan1.png)

The result shows that for tcp, only ports 22 and 8000 are open.

I tried to fingerprint the services.

![diagram](../../images/CodePartTwo/CodePartTwo_nmapscan2.png)

This didn't give too much information, so at this point I visited the web service with the browser.

## 2 Foothold

On the web page we have the option to register, login or download the app source code.

I downloaded the code, then I proceeded with registration and login.

Once signed in we are given the possibility to run code. At this point to have a better grasp of the attack angle I went to look at the app source code.

These are the principal components we can find:

![diagram](../../images/CodePartTwo/CodePartTwo_appcontent.png)

Looking at app.py, this is the flask route which lets us run the code:

![diagram](../../images/CodePartTwo/CodePartTwo_appcode.png)

We can see that the code we submit in the browser is posted on this route on the backend, then is run with the js2py.eval_js function.
This function is used to execute Javascript code in a python context (such as app.py) and return the last expression.
Is used, among other things, in web scraping.
Since this library can only execute JS code, I didn't see an attack angle unless it were vulnerable versions.
So I searched for a vulnerability:

![diagram](../../images/CodePartTwo/CodePartTwo_vuln.png)

CVE-2024-28397 allow an attacker to break the JS sandbox and run commands in the python environment. So we may be able to execute code if we are in the right circumnstances.
The vulnerability requirements are:

![diagram](../../images/CodePartTwo/CodePartTwo_vulnreq.png)

and looking at the application requirements.txt:

![diagram](../../images/CodePartTwo/CodePartTwo_reqtxt.png)

The library version is vulnerable, also the app is running flask 3.0.3 which supports python3, so with the right payload we should have RCE.

Using the specific PoC for this vulnerability, adjusted with some trial and error you should be able to get RCE as the "app" user.


## 3 Lateral Movement

The only two users having a home directory are "app" and "marco".
Having the shell as "app", the first thing I did was "sudo -l", but the user wasn't able to run sudo.
The second thing I did was to look at the app database.
We know that we have a db file at app/instance/users.db.
So connecting to the database with sqlite3 and enumerating the tables we find:

![diagram](../../images/CodePartTwo/CodePartTwo_hashes.png)

These are md5 hashes (which can be confirmed also looking at the app source code). Trying to crack with John, we retrieve the marco credentials:

![diagram](../../images/CodePartTwo/CodePartTwo_john.png)


## 4 Privilege Escalation

With the marco user and its cute password we can access through ssh, and we can run "sudo -l":

![diagram](../../images/CodePartTwo/CodePartTwo_sudo.png)

This time marco can run the command **/usr/local/bin/npbackup-cli** with sudo. I didn't know this executable so I searched some info. It is an executable for managing backups. With **npbackup-cli --help**, we can see that the **raw** option allows us to run commands "against the backend".

![diagram](../../images/CodePartTwo/CodePartTwo_npbackuphelp.png)

So I tried:

![diagram](../../images/CodePartTwo/CodePartTwo_restic.png)

The error suggests that this binary is using the **restic** command.
So I looked it up on GTFObins:

![diagram](../../images/CodePartTwo/CodePartTwo_gtfobins.png)

I tried the suggested payload:

![diagram](../../images/CodePartTwo/CodePartTwo_escpayload1.png)

The error suggests that this process cannot spawn a terminal process, indeed we can see that the payload is trying to open an sh shell.
If we replace the payload with something which runs on the background it could work. At first I tried with netcat, but the nc on the target doesn't provide the "-e" option to spawn /bin/bash, so you would have to transfer nc from your attack machine (or use another payload).
With this final payload I got the root shell on my listener:

![diagram](../../images/CodePartTwo/CodePartTwo_escpayload2.png)

## 5 Remediation
- Do not expose vulnerable services on external interfaces (the features of the service running on port 8000 suggest it is a development platform). Use internal networks only.
- Sandbox running services in containers.
- Do not globally expose source code running in reachable environment.
- Keep Libraries up to date.
- Set up a good password policy, strong encryption algorithms and avoid password reuse.
- Specify the exact command the users can run with sudo in the */etc/sudoers* file. 


I hope you enjoyed, Bye.
