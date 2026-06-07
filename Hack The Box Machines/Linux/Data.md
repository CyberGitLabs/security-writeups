# Data

Data is an HTB Linux machine classified as Easy.

*******1 Service Enumeration*******

Doing a full tcp port scan we can see that the machine has only two open ports 22 and 3000:

![diagram](../../images/Data/Data_nmap.png)

While 22 is hosting the ssh service, we can see that on port 3000 is running Grafana.

![diagram](../../images/Data/Data_Grafana.png)


*******2 Foothold*******

This version of Grafana has a path traversal vulnerability which results in Local File Inclusion, allowing us to access files in the target operating system which can be accessed by the user running the Grafana Service.

I have used an exploit to download the database file:

![diagram](../../images/Data/Data_exp.png)

But the same result can be obtained making the following request with curl:

**curl `http://10.129.234.47:3000/public/plugins/stat/../../../../../../../../../../../../../../../../var/lib/grafana/grafana.db` --path-as-is -o file_db**

Checking the type of the .db file we can see that is a Sqlite3 file:

![diagram](../../images/Data/Data_dbfile.png)

We can interact with the database and query the users:

![diagram](../../images/Data/Data_users.png)

We can save the hashes and try to crack them to get the passwords. Examining them with hashid we find that the format cannot be recognized, searching online I found a tool that can transform the grafana hashes in a format recognized by hashcat.

![diagram](../../images/Data/Data_hashes.png)

Now we can use hashcat to try to retrieve a password:

![diagram](../../images/Data/Data_hashcat1.png)

![diagram](../../images/Data/Data_hashcat2.png)

We retrieved the password "beutiful1". Using it with the user boris we can access with ssh:

![diagram](../../images/Data/Data_bob.png)

So now we got a shell. One thing we can notice is that if we try to read /etc/passwd wit the local file inclusion vulnerability we don't see the boris user:

![diagram](../../images/Data/Data_passwd.png)

and the content differs from the /etc/passwd file we can read in the boris shell. This means that the Grafana application is likely running in a container.

*******3 Privilege Escalation*******

The user can run the following command as sudo:

![diagram](../../images/Data/Data_sudo.png)

This command allow a user to "Run a command in a running container":

![diagram](../../images/Data/Data_docker_exec.png)

However it requires the container id which we want to run a command in. Our user doesn't have access to the locations where the containers usually live, but watching for running processes we can retrieve the container id:

![diagram](../../images/Data/Data_containerid.png)

Now we can run the command. Because of the wildcard we can add whatever argument we want to the command, so we try to spawn a shell inside the container as the root user:

![diagram](../images/Data/Data_docker_root.png)

We have root privileges inside the container, so we only have to escape the container environment to acquire root privileges on the host. If we enumerate the capabilities we find:

![diagram](../../images/Data/Data_Cap.png)

We are interested in the field CapEff, Effective Capabilities, the set that determines whether a privileged operation succeeds.
This field is computed based on other Cap fields and the kernel checks it (at syscall time) to know if a privileged process can be executed. Let's decode them:

![diagram](../../images/Data/Data_Cap2.png)

We can see that the set of effective capabilities comprises CAP_SYS_ADMIN a capability that allow us to mount the file system and manipulate namespaces. 
So we see that we can mount /dev/sda1, the host root partition, to a new directory inside the container and move there.

![diagram](../../images/Data/Data_esc.png)

The mount operation is succesfull and now we can access the host filesystem from inside the container where we have full root privileges. From here we can manipulate every file of the host and gain real root privileges.

![diagram](../../images/Data/Data_esc2.png)

*******4 Remediation*******
- Keep Grafana up to date.
- Define a good password policy.
- Whitelist specific commands in the */etc/sudoers* file to be run in the container (instead of exec *)
- Start the container more securely:
  - drop capabilities and add back only the needed ones
  - run as a non root user
  - consider read-only mode, or give write access only to specific mounted volumes
  - use the *no-new-privileges* flag.

