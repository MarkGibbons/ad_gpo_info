## Retrieving Active Directory GPO Information from Linux

### Introduction

We needed to be able to read GPOs attached to the OUs holding our computer objects.  I couldn't find clear documentation about the process so compiled these notes.  We are using RHEL 7 and  AD 2008\.  These methods solved my problems, use at your own risk without any warranty or promise that these methods are at all safe to use.

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

<ac:structured-macro ac:macro-id="c2fa551b-9b99-4882-9e85-2db7fcd9d08e" ac:name="code" ac:schema-version="1"><ac:plain-text-body></ac:plain-text-body></ac:structured-macro>

###### Join command

This command reads a password from the password file.  You could also distribute a key tab file with established host credentials.  

<ac:structured-macro ac:macro-id="b9cc0b25-a3e6-4dad-9e8e-4d5a346e9b46" ac:name="code" ac:schema-version="1"><ac:plain-text-body>--stdin-password -O<container' -d="" nordstrom.net="" ]]=""></container'></ac:plain-text-body></ac:structured-macro>

##### kinit

###### krbtgt

The ticket granting ticket is created using the credentials from the keytab.  TGT is usually only valid for a limited amount of time/

###### /tmp/krb5cc_0

The ticket granting ticket and service principle tickets are kept in this file.

###### Service principle

Using ldapsearch will cause a ticket for an ldap service principle to be created.

###### kinit to refresh the TGT.  

Use the credentials in the key tab to refresh the krbtgt ticket.

<ac:structured-macro ac:macro-id="e276f54b-02d5-4d69-b52c-582479e3a57b" ac:name="code" ac:schema-version="1"><ac:plain-text-body></ac:plain-text-body></ac:structured-macro>

###### klist

Can be used to list the currently active tickets.

<ac:structured-macro ac:macro-id="27494780-5836-4619-a076-9c1c5ade0396" ac:name="code" ac:schema-version="1"><ac:plain-text-body></ac:plain-text-body></ac:structured-macro>

##### Domain Controller

        Nslookup or equivalent may be used to find domain controllers.  In my case nordstrom.net is a round robin c-name.  It doesn't play well with ldapsearch.  I pull a specific server from the list and use that. 

<ac:structured-macro ac:macro-id="a59ebdae-87b6-4de1-a4e5-8af645f7bbf7" ac:name="code" ac:schema-version="1"><ac:plain-text-body></ac:plain-text-body></ac:structured-macro>

##### Server OU

######         Find the server OU.  

From that name put together a table parent OUs.

<ac:structured-macro ac:macro-id="49ce5ae6-d42a-483f-b207-4dfc40fc181c" ac:name="code" ac:schema-version="1"><ac:plain-text-body></ac:plain-text-body></ac:structured-macro>

distinguishedName: CN=y0319p01,OU=Prod,OU=UNIXALL,OU=Computers,OU=UNIX,DC=nord  

##### GPO attached to OU

        Find the parent OUs, search each of them for attached GPOs.

ou=prod,ou=unixall,ou=computers,ou=unix

ou=unixall,ou=computers,ou=unix

ou=computers,ou=unix

ou=unix

Search each OU for gPlink, get cn in policies,system,nordstrom,net

<ac:structured-macro ac:macro-id="10c7c73b-a8f9-43e2-b268-bbdad742786f" ac:name="code" ac:schema-version="1"><ac:plain-text-body></ac:plain-text-body></ac:structured-macro>

gPLink: [LDAP://cn={84C43D09-B377-421E-82C2-FFFFFFFFF},cn=policies,cn=system,DC=nordstrom,DC=net;0]

##### smbclient 

We can use smbclient to list the file structure or retrieve files from the policy.  Here's an example of retrieving a file.  You can treat smbclient much like an ftp client.

<ac:structured-macro ac:macro-id="d0c57d8a-b6e9-4938-975b-e3951ec390e8" ac:name="code" ac:schema-version="1"><ac:plain-text-body></ac:plain-text-body></ac:structured-macro>

### Acknowlegements

RHEL 7 documentation from Red Hat

<u>[http://unix.stackexchange.com/questions/134577/get-a-kerberos-service-ticket-from-the-command-line](http://unix.stackexchange.com/questions/134577/get-a-kerberos-service-ticket-from-the-command-line)</u>

<u>[http://www.roguelynn.com/words/explain-like-im-5-kerberos/](http://www.roguelynn.com/words/explain-like-im-5-kerberos/)</u>

Craig Barrett

<u><span style="color: rgb(0,51,102);">Kerberos: The Definitive Guide </span></u> <span style="color: rgb(0,51,102);">Jason Garman, O'Reilly Media, Inc. August 2003</span>

<span style="color: rgb(0,51,102);"><u>LDAP System Administration</u> Gerald Carter O'reilly Media, Inc March 2003</span>

<span style="color: rgb(0,51,102);">Countless web searches and weeks of confusion.</span>

