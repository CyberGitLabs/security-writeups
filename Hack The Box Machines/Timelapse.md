# Timelapse

Timelapse is an HTB Windows machine classified as Easy.

*******1 Service Enumeration*******

![diagram](../images/Timelapse/Timelapse_nmap1.png)

The result shows several TCP ports open. We can recognize this machine as a domain controller for the presence of the kerberos service on port 88.
Also there are interesting services which we can interact with like SMB (445), RPC (135), ldap (389, 636) and possibly winrm over https (5986).
I tried to get more information with the version scan + default scripts scan.

![diagram](../images/Timelapse/Timelapse_nmap2.png)

![diagram](../images/Timelapse/Timelapse_nmap3.png)

From this scan we have obtained the domain name *timelaps.htb* and the FQDN of the machine *dc01.timelaps.htb*. We can add them to /etc/hosts assigned to the machine IP.
Also we can confirm that the service running on port 5986 is winrm over HTTPS (Microsoft HTTPAPI).

*******2 Foothold*******

Since SMB is the easier service to interact with i started with that one.

![diagram](../images/Timelapse/Timelapse_shares.png)

The Guest user has read access over the *Shares* share. So i downloaded everything:

![diagram](../images/Timelapse/Timelapse_shares2.png)

In the Dev share there is a zip file called *winrm_backup.zip*. The name suggests that has something to do with winrm, so i tried to extract it:

![diagram](../images/Timelapse/Timelapse_unzip.png)

It's password protected, we have to extract the hash, crack it and then use the resulting password to extract the zip file:

![diagram](../images/Timelapse/Timelapse_zip2john.png)

![diagram](../images/Timelapse/Timelapse_john1.png)

![diagram](../images/Timelapse/Timelapse_unzip2.png)

The zip contained a .pfx file.

![diagram](../images/Timelapse/Timelapse_pxffile.png)

A .pfx file is a password-protected, binary file used in windows to securely store a SSL/TLS certificate, its corresponding private key, and optionally other intermediate certificates.
The winrm protocol on windows supports client authentication through SSL/TLS certificates, if the certificate is mapped to a valid user with Remote Management access and client authentication is enabled. This is another authentication mechanism valid for winrm other than Basic auth (username+password), NTLM and Kerberos. At this point we don't know yet that this is the case, first we need to inspect the content of the .pfx file, but it's a good guess (given also the name of the zip file). Trying to inspect the .pfx file we see that it's password protected, so we do the same procedure as above:

![diagram](../images/Timelapse/Timelapse_readpfx1.png)

![diagram](../images/Timelapse/Timelapse_pfx2john2.png)

![diagram](../images/Timelapse/Timelapse_john2.png)

![diagram](../images/Timelapse/Timelapse_readpfx2.png)

Inspecting the content of the file we see that it contains both the certificate and the private key.
To use them with evil-wirm on linux we need to have one (PEM) file for each:

![diagram](../images/Timelapse/Timelapse_gencert.png)

![diagram](../images/Timelapse/Timelapse_genkey.png)

Now let's inspect the content of the certificate:

![diagram](../images/Timelapse/Timelapse_certinfo1.png)

![diagram](../images/Timelapse/Timelapse_certinfo2.png)

From this we can see that has been issued by *`legacyy@timelapse.htb`* (Active Directory uses that UPN to map the certificate to a user account) and that is a *TLS Web Client Authentication* certificate.

Now we can use the new files to try to access with evil-winrm:

![diagram](../images/Timelapse/Timelapse_legacyyauth.png)


*******3 Post-Exploitation and Privilege Escalation*******

We are logged in with the *legacyy* user. The first thing i did was try to see which privileges he had:

![diagram](../images/Timelapse/Timelapse_legacyywhoami1.png)

![diagram](../images/Timelapse/Timelapse_legacyywhoami2.png)

Beside being part of the *Remote Management Users* group, the privileges and groups are the ones assigned usually to normal users.
Going ahead with internal enumeration we can see that this user doesn't have many permissions and have access to few resources.

However i remembered that we had the *Helpdesk* folder in the share which we didn't use yet (to access we used only the content of the *Dev* folder).
*Helpdesk* contains Microsoft documentation about LAPS (Local Administrator Password Solution) and the installer file. LAPS is a Microsoft solution that automatically manages and rotates the local Administrator password on domain-joined machines. The password of the local administrator is written as a plaintext attribute *ms-Mcs-AdmPwd* on the computer object. Inspecting the documentation we can read that a component called CSE is installed by default when running the installer and the main binary will be located at: *C:\Program Files\LAPS\CSE\AdmPwd.dll*. We can see that this binary is present on the machine so LAPS should be installed. We can try to read the *ms-Mcs-AdmPwd* property with this command:

![diagram](../images/Timelapse/Timelapse_getAdmPwd.png)

But we see that it's not returned. Likely our user doesn't have the permission.
We can use LAPSToolkit.ps1 to enumerate who has permission to manage LAPS (you first need to upload it on the machine). The command *Find-AdmPwdExtendedRights* retrieves who has permission to read the LAPS password attribute.

![diagram](../images/Timelapse/Timelapse_LAPSToolkit.png)

From this we can see that on *dc01.timelapse.htb* the group *LAPS_Readers* has extended rights which implies it can also read the *ms-Mcs-AdmPwd* property. Now we want to see who is contained in this group:

![diagram](../images/Timelapse/Timelapse_LAPSReaders.png)

We see that the *svc_deploy* user is member of the group, so if we can access with this user we should be able to read the admin password.
I used this command to search the filesystem for the *svc_deploy* string:

![diagram](../images/Timelapse/Timelapse_Search_svc_deploy.png)

We can see that the Powershell history file contains the credentials for *the svc_deploy* user:

![diagram](../images/Timelapse/Timelapse_PowershellHistory.png)

Now we can access with *svc_deploy* and read the *Administrator* password:

![diagram](../images/Timelapse/Timelapse_svc_login.png)

![diagram](../images/Timelapse/Timelapse_AdminPassword.png)

Then we can access as Administrator.



