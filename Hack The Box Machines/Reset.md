# Reset

Reset is an HTB Linux machine classified as Easy.

*******1 Service Enumeration*******

As always, approaching this machine I did a full TCP scan.

![diagram](../images/Reset/Reset_nmap.png)

The scan result shows, beside ssh and http, three interesting services: rsh, rlogin and rexec. I tried for a while to check misconfigurations on those, but I couldn't find anything. 
Then I moved to the http web server.

*******2 Foothold*******

Navigating to the target we can see an admin login portal:

![diagram](../images/Reset/Reset_admin_portal.png)

Trying the some users with the change password functionality, we can find the user *admin*.

![diagram](../images/Reset/Reset_rest_password.png)

While when trying a not registered user, we get an error message:

![diagram](../images/Reset/Reset_wrong_user.png)

Examining the response we can see that the new password value is sent in the response body:

![diagram](../images/Reset/Reset_rest_response.png)

So now we have the credentials to access the web appplication. From within we can see that we are provided with the functionality to view logs:

![diagram](../images/Reset/Reset_logs2.png)

Even though only two options are given from the frontend we can always try to include different files, crafting our requests. 
I tried to include /etc/passwd but it doesn't work.
So noticing that the webserver is running apache we can try to include apache logs:

![diagram](../images/Reset/Reset_apche_logs.png)

![diagram](../images/Reset/Reset_apche_logs2.png)

We see that we could successfully include apache logs. So now we should try *Log Poisoning*.
First we send a request with the payload in the user agent:

![diagram](../images/Reset/Reset_logs_poisoning.png)

Then we make a request to the access.log again:

![diagram](../images/Reset/Reset_logs_poisoning2.png)

From the response we see that the *id* command is correctly executed.
So we can use the following request to get a reverse shell:

![diagram](../images/Reset/Reset_logs_poisoning3.png)

![diagram](../images/Reset/Reset_shell.png)

*******3 Privilege Escalation*******

The *www-data* user is also a member of the *adm* group, this group can read logs (as we could infer from what we have done so far).
In the system are present 2 users: *local* and *sadm*.
Since from my internal enumeration I found that *sadm* was allowed to run rsh commands (from the content of /etc/hosts.equiv), I went to the */var/log* folder and searched for the string *sadm* in all the files:
![diagram](../images/Reset/Reset_grep_logs.png)

This will output all the content which includes *sadm*, recursively from all the files included in this folder and its subfolders.
The file audit.log, depending on the configuration can contain different events, in this case recorded system calls with also commands. Some of the commands where encoded in hex format.
Decoding one of these commands I could retrieve the password of the *sadm* user:

![diagram](../images/Reset/Reset_sadm_pass.png)

and I could get a shell as this user:

![diagram](../images/Reset/Reset_sadm_shell.png)

From the output above we can see that the *sadm* user can use *sudo* with the command *nano /etc/firewall.sh*.
This offers a privilege escalation path since we can spawn a shell through *nano*, as explained here:

![diagram](../images/Reset/Reset_GTFOBins.png)

Following the procedure we get a shell as root:

![diagram](../images/Reset/Reset_root.png)

*******4 Remediation*******

- Secure the Web Application:
  - The feeback message from the *Reset Password* functionality shouldn't distinguish between existing and not existing users.
  - The password shouldn't be included in the response body.
  - Whitelist log files that can be accessed by the web application allowing only *syslog* and *auth.log*.
  - Do not add the web server user to the adm group. If the application needs to read specific log files (*syslog* and *auth.log*), create a dedicated group with access only to those files instead of granting broad log-reading permissions.
  - Avoid using include() to render file content controlled by user input. Use file_get_contents() instead.
  - Set open_basedir in php.ini to restrict PHP file access to the webroot directory only.
  - Allowlist GTFOBins like *nano* to be run with sudo.
