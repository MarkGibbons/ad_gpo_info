## Retrieving Active Directory GPO Information from Linux

### Introduction

We needed to be able to read GPOs attached to the OUs holding our computer objects.  I couldn't find clear documentation about the process so compiled these notes.  My hope is that this doc will save someone a few of the weeks I spent getting this to work. We are using RHEL 7 and  AD 2008\.  These methods solved my problems, use at your own risk without any warranty or promise that these methods are at all safe to use.

### <span style="color: rgb(0,0,0);font-size: 16.0px;font-weight: bold;">GPO structure</span>

### <span style="color: rgb(0,0,0);font-size: 16.0px;font-weight: bold;"> </span> <span style="color: rgb(0,0,0);font-size: 16.0px;font-weight: bold;">OU structure</span>

<span style="color: rgb(0,0,0);font-size: 16.0px;">Our computer objects are kept in an OU tree with a structure similar to this:</span>

<span style="color: rgb(0,0,0);font-size: 16.0px;">   cn -> ou=type, ou=app, ou=computers, ou=unix</span>

<span style="color: rgb(0,0,0);font-size: 16.0px;"> </span> <span style="color: rgb(0,0,0);font-size: 16.0px;font-weight: bold;">Packages</span>

RHEL 7 packages to install.  This list is probably sufficient but may include extra packages.

*   adcli
*   authconfig
*   krb5-workstation
*   openldap-clients

##### Join active directory

###### Kerberos

Kerberos is a protocol for network authentication. Authentication tokens are strongly encrypted.  We end up using one clear text password read from a secured file.

###### /etc/krb5.keytab

The adcli command may be used to join a server to active directory.  As part of the join host level credentials are created and are stored in /etc/krb5.keytab.  The credentials in the keytab may be used to create a ticket granting ticket.

###### /etc/krb5.conf

````
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log
[libdefaults]
 default_realm = NORDSTROM.NET
 dns_lookup_realm = true
 dns_lookup_kdc = true
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
[domain_realm]
 nordstrom.net = NORDSTROM.NET
 .nordstrom.net = NORDSTROM.NET
````

###### Join command

This command reads a password from the password file.  You could also distribute a key tab file with established host credentials.  

````
# tr d "\012" <passwordfile | adcli join -U <user> --stdin-password -O <container' -D nordstrom.net
````

##### kinit

###### krbtgt

The ticket granting ticket is created using the credentials from the keytab.  TGT is usually only valid for a limited amount of time/

###### /tmp/krb5cc_0

The ticket granting ticket and service principle tickets are kept in this file.

###### Service principle

Using ldapsearch will cause a ticket for an ldap service principle to be created.

###### kinit to refresh the TGT.  

Use the credentials in the key tab to refresh the krbtgt ticket.

````
kinit -k -t /etc/krb5.keytab
````

###### klist

Can be used to list the currently active tickets.

````
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: host/y0319t11043.nordstrom.net@NORDSTROM.NET
Valid starting       Expires              Service principal
02/18/2017 11:23:27  02/18/2017 21:23:27  krbtgt/NORDSTROM.NET@NORDSTROM.NET
	renew until 02/25/2017 11:23:27
02/18/2017 11:24:21  02/18/2017 21:23:27  ldap/dc7.nordstrom.net@NORDSTROM.NET
	renew until 02/25/2017 11:23:27
````

##### Domain Controller

        Nslookup or equivalent may be used to find domain controllers.  In my case nordstrom.net is a round robin c-name.  It doesn't play well with ldapsearch.  I pull a specific server from the list and use that. 

````
nslookup nordstrom.net
````

##### Server OU

######         Find the server OU.  

From that name put together a table parent OUs.

````
ldapsearch -h 'dc7.nordstrom.net' -s sub -b 'ou=unix,dc=nordstrom,dc=net' -Y GSSAPI "(samaccountname=Y0319P01$)"
````

distinguishedName: CN=y0319p01,OU=Prod,OU=UNIXALL,OU=Computers,OU=UNIX,DC=nord  

##### GPO attached to OU

        Find the parent OUs, search each of them for attached GPOs.

````
ou=prod,ou=unixall,ou=computers,ou=unix
ou=unixall,ou=computers,ou=unix
ou=computers,ou=unix
ou=unix
````

Search each OU for gPlink, get cn in policies,system,nordstrom,net

````
# ldapsearch -h 'dc7.nordstrom.net' -s base -b 'ou=prod,ou=unixall,ou=computers,ou=unix,dc=nordstrom,dc=net' -Y GSSAPI "(objectclass=organizationalunit)"
````

gPLink: [LDAP://cn={84C43D09-B377-421E-82C2-FFFFFFFFF},cn=policies,cn=system,DC=nordstrom,DC=net;0]

##### smbclient 

We can use smbclient to list the file structure or retrieve files from the policy.  Here's an example of retrieving a file.  You can treat smbclient much like an ftp client.

````
smbclient '\\dc7\SYSVOL' --directory 'nordstrom.net\Policies\{84C43D09-B377-421E-82C2-FFFFFFFFF}' -c 'get file.xml' -k
````

### Acknowlegements

RHEL 7 documentation from Red Hat

<u>[http://unix.stackexchange.com/questions/134577/get-a-kerberos-service-ticket-from-the-command-line](http://unix.stackexchange.com/questions/134577/get-a-kerberos-service-ticket-from-the-command-line)</u>

<u>[http://www.roguelynn.com/words/explain-like-im-5-kerberos/](http://www.roguelynn.com/words/explain-like-im-5-kerberos/)</u>

Craig Barrett

<u><span style="color: rgb(0,51,102);">Kerberos: The Definitive Guide </span></u> <span style="color: rgb(0,51,102);">Jason Garman, O'Reilly Media, Inc. August 2003</span>

<span style="color: rgb(0,51,102);"><u>LDAP System Administration</u> Gerald Carter O'reilly Media, Inc March 2003</span>

<span style="color: rgb(0,51,102);">Countless web searches and weeks of confusion.</span>

