# Cicada

## Information

Cicada is an HTB Windows machine classified as Easy.

Notable Topics:
  - SMB Shares
  - Ldap Queries
  - SeBackupPrivilege

## 1 Service Enumeration

![diagram](../../images/Cicada/Cicada_nmap1.png)

![diagram](../../images/Cicada/Cicada_nmap2.png)

From the running services we can say that this is very likely a domain controller. The info can be confirmed with a more comprehensive scan.

## 2 Foothold

Since SMB is running the first thing I did was enumerate the shares:

![diagram](../../images/Cicada/Cicada_shares.png)

We see that there are 2 custom shares *HR* and *DEV*, and that we have read access to *HR*. Let's try to access and retrieve the files:

![diagram](../../images/Cicada/Cicada_file.png)

![diagram](../../images/Cicada/Cicada_password1.png)

Nice, we already obtained a clear text password. Now let's try to enumerate the domain users:

![diagram](../../images/Cicada/Cicada_brte_users1.png)

![diagram](../../images/Cicada/Cicada_brte_users2.png)

This type of user enumeration can be done also leveraging RPC or LDAP protocol, if null sessions are enabled or if you already have credentials. Let's see how to do it with rpcclient. First we retrieve the domain SID:

![diagram](../../images/Cicada/Cicada_Dom_sid.png)

Then we loop over the user RID, here I took a range from 500, which is the Administrator Account, to 2000 but it can be adjusted:

![diagram](../../images/Cicada/Cicada_brte_users3.png)

![diagram](../../images/Cicada/Cicada_brte_users4.png)

Then let's spray the password over the found users:

![diagram](../../images/Cicada/Cicada_spray.png)

Good, we have found working domain user's credentials. At this point I tried to access with winrm but I couldn't. 
To confirm I enumerated the members of the *Remote Management Users* group with rpcclient. First we have to find the group RID querying the builtin groups:

![diagram](../../images/Cicada/Cicada_groups1.png)

![diagram](../../images/Cicada/Cicada_groups2.png)

![diagram](../../images/Cicada/Cicada_remusers.png)

Then we query for the members using the group RID:

![diagram](../../images/Cicada/Cicada_remusers2.png)

Comparing with the results above we can see that the only user member of this group is *emily.oscars*.

At this point I tried to follow the file's indication and change the user's password. 
I thought that maybe after the password was changed the user could have been assigned to the groups it was supposed to belong like the *Remote Management Users*.
But I couldn't change it, so I started to enumerate users with an ldap query, using the credentials we have:

![diagram](../../images/Cicada/Cicada_ldap_query.png)

![diagram](../../images/Cicada/Cicada_ldap_query2.png)

The password for *david.orelious* was saved in its description. With this user we can access the *DEV* share:

![diagram](../../images/Cicada/Cicada_shares2.png)

And downloading and reading the powershell script we retrieve the *emily.oscars* password:

![diagram](../../images/Cicada/Cicada_script1.png)

![diagram](../../images/Cicada/Cicada_script2.png)

We know already that this user is a member of *Remote Management Users* so we can access via evil-winrm.

## 3 Privilege Escalation

We can see that this user is very powerfull, since is member of the *Backup Operators* too and has *SeBackupPrivilege* and *SeRestorePrivilege*:

![diagram](../../images/Cicada/Cicada_whoami1.png)

![diagram](../../images/Cicada/Cicada_whoami2.png)

However when I tried to create shadow copies to access the NTDS.dit file I couldn't, as it requires admin privileges.
So I dumped the SAM and SYSTEM hives and ran *secretsdump* against them to extract the Administrator hash.:

![diagram](../../images/Cicada/Cicada_sam.png)

After that we can pass the hash and access as Administrator:

![diagram](../../images/Cicada/Cicada_admin.png)


## 4 Remediation
- Disable the SMB and RPC Null and Guest session.
- Do not store plaintext passwords in user object descriptions or scripts.
- Monitor activity for high privileged account (members of *Backup Operators*).
- Whitelist source IP addresses for remote connections by members of *Remote Management Users*
