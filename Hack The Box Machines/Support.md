******Support******

Support is an HTB Windows machine classified as Easy.

*******1 Service Enumeration*******

![diagram](../images/Support/Support_nmap1.png)

![diagram](../images/Support/Support_nmap2.png)

From the running services we can say that this is a domain controller.

*******2 Foothold*******

Enumerating the SMB shares i found a interesting zip file *UserInfo.exe.zip* in the *support-tools* share:

![diagram](../images/Support/Support_smbclient.png)

So i downloaded it, extracted the content and started enumerating the resulting files with *strings*.
Examining *UserInfo.exe* with *strings* we can see that is a .NET file and it contains strings like 'enc_password'.
.NET executables compile to Intermediate Language (IL) / CIL bytecode, not native machine code.
This make possible to decompile and reverse engineer them with specialized tools. I used *ilspycmd* to do that:

![diagram](../images/Support/Support_ilspycmd1.png)

![diagram](../images/Support/Support_ilspycmd2.png)

As we can see in the class is contained the logic to decode an encrypted password.
So i made the AI generate a script to copy this logic:

![diagram](../images/Support/Support_script1.png)

![diagram](../images/Support/Support_script2.png)

Once we found a password i retrieved the list of users with rid-brute and then sprayed to retrieve the pair:

![diagram](../images/Support/Support_users.png)
